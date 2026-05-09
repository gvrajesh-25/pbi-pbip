# Power BI Version Control with PBIP & Azure Repos

A guide for implementing professional version control for Power BI projects using the **Power BI Project (PBIP)** format and **Azure Repos** (Git).

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [What is PBIP?](#what-is-pbip)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Branching Strategy](#branching-strategy)
- [Workflow](#workflow)
- [CI/CD with Azure Pipelines](#cicd-with-azure-pipelines)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

---

## Overview

This project demonstrates how to manage Power BI reports and semantic models using source control. By saving reports in the **PBIP format**, all report artifacts are stored as human-readable files (JSON, TMDL), making it possible to track changes, review diffs, and collaborate using standard Git workflows in **Azure DevOps**.

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Power BI Desktop | June 2023+ | Required for PBIP format support |
| Azure DevOps | Any | Azure Repos (Git) enabled |
| Git | 2.x+ | Installed locally |
| Power BI Service | Any | Workspace with XMLA endpoint (Premium / PPU) |

> **Note:** PBIP format must be enabled in Power BI Desktop under **File → Options → Preview features → Store reports using enhanced metadata format (PBIP)**.

---

## What is PBIP?

The **Power BI Project (PBIP)** format saves Power BI files as a folder of plain-text source files instead of a single binary `.pbix` file.

### Key Components

```
MyReport.Report/
├── definition/
│   ├── report.json          # Report layout and configuration
│   ├── pages/               # Individual report pages
│   └── bookmarks/           # Bookmarks (if any)
├── .pbi/
│   └── localSettings.json   # Local-only settings (gitignored)
└── StaticResources/         # Custom themes and images

MyModel.SemanticModel/
├── definition/
│   ├── model.bim            # Or TMDL folder (Tabular Model Definition Language)
│   ├── tables/              # One file per table (TMDL format)
│   ├── relationships.tmdl
│   └── cultures/
└── .pbi/
    └── localSettings.json
```

> **Advantage over `.pbix`:** Diffs are meaningful. You can see exactly which measure, visual, or relationship changed between commits.

---

## Repository Structure

```
📁 powerbi-project/
├── 📁 reports/
│   ├── 📁 SalesAnalysis.Report/
│   └── 📁 FinanceDashboard.Report/
├── 📁 models/
│   └── 📁 CompanyDataModel.SemanticModel/
├── 📁 pipelines/
│   ├── deploy-dev.yml
│   ├── deploy-test.yml
│   └── deploy-prod.yml
├── 📁 docs/
│   └── architecture.md
├── .gitignore
└── README.md
```

---

## Getting Started

### 1. Clone the Repository

```bash
git clone https://dev.azure.com/<your-org>/<your-project>/_git/<repo-name>
cd <repo-name>
```

### 2. Open a Report in Power BI Desktop

- Open Power BI Desktop.
- Go to **File → Open** and navigate to a `.pbip` file in the cloned repo.
- Make your changes to the report or model.

### 3. Save Changes

- **File → Save** (Ctrl+S) — saves changes back to the PBIP folder structure.
- Do **not** use "Save As .pbix" as this converts it to a binary format.

### 4. Commit and Push

```bash
git status
git add reports/SalesAnalysis.Report/
git commit -m "feat: add YTD sales measure to SalesAnalysis"
git push origin feature/ytd-sales-measure
```

### 5. Create a Pull Request

Open a Pull Request in Azure DevOps for review before merging to `main`.

---

## Branching Strategy

This project follows a **GitFlow-inspired** branching model:

```
main                   → Production-ready reports
  └── test             → QA / UAT environment
        └── dev        → Development environment
              └── feature/<name>   → Individual feature branches
              └── fix/<name>       → Bug fix branches
```

| Branch | Purpose | Deploys To |
|--------|---------|------------|
| `main` | Stable, approved reports | Production Workspace |
| `test` | QA and user testing | Test Workspace |
| `dev` | Active development | Dev Workspace |
| `feature/*` | New features or reports | Local / Dev |
| `fix/*` | Bug fixes | Local / Dev |

---

## Workflow

```
1. Branch off dev
       ↓
2. Open PBIP in Power BI Desktop
       ↓
3. Make changes (visuals, measures, etc.)
       ↓
4. Save → Git Add → Commit → Push
       ↓
5. Open Pull Request to dev
       ↓
6. Code Review (diff check in Azure Repos)
       ↓
7. Merge to dev → auto-deploy to Dev Workspace
       ↓
8. Promote to test → auto-deploy to Test Workspace
       ↓
9. Promote to main → auto-deploy to Production Workspace
```

---

## CI/CD with Azure Pipelines

Automated deployments use the [Power BI REST API](https://learn.microsoft.com/en-us/rest/api/power-bi/) or the [Fabric REST API](https://learn.microsoft.com/en-us/rest/api/fabric/) to publish PBIP files to Power BI workspaces.

### Example Pipeline (`deploy-dev.yml`)

```yaml
trigger:
  branches:
    include:
      - dev

pool:
  vmImage: 'windows-latest'

variables:
  - group: PowerBI-ServicePrincipal

steps:
  - checkout: self

  - task: PowerShell@2
    displayName: 'Install Power BI Cmdlets'
    inputs:
      targetType: 'inline'
      script: |
        Install-Module -Name MicrosoftPowerBIMgmt -Force -Scope CurrentUser

  - task: PowerShell@2
    displayName: 'Deploy Reports to Dev Workspace'
    inputs:
      targetType: 'inline'
      script: |
        $credential = New-Object System.Management.Automation.PSCredential(
          "$(CLIENT_ID)",
          (ConvertTo-SecureString "$(CLIENT_SECRET)" -AsPlainText -Force)
        )
        Connect-PowerBIServiceAccount -ServicePrincipal `
          -Credential $credential -TenantId "$(TENANT_ID)"

        # Publish each report
        Get-ChildItem -Path "reports" -Filter "*.pbip" | ForEach-Object {
          New-PowerBIReport -Path $_.FullName `
            -WorkspaceId "$(DEV_WORKSPACE_ID)" -ConflictAction CreateOrOverwrite
        }
```

### Azure DevOps Variable Groups

Store secrets securely in **Azure DevOps Library → Variable Groups**:

| Variable | Description |
|----------|-------------|
| `TENANT_ID` | Azure AD Tenant ID |
| `CLIENT_ID` | Service Principal App ID |
| `CLIENT_SECRET` | Service Principal Secret |
| `DEV_WORKSPACE_ID` | Power BI Dev Workspace GUID |
| `TEST_WORKSPACE_ID` | Power BI Test Workspace GUID |
| `PROD_WORKSPACE_ID` | Power BI Prod Workspace GUID |

---

## Best Practices

### Git

- ✅ Commit frequently with descriptive messages (use [Conventional Commits](https://www.conventionalcommits.org/))
- ✅ Keep Pull Requests small and focused on one change
- ✅ Always review diffs in Azure Repos before merging
- ✅ Require at least one reviewer for PRs into `dev`, two for `main`
- ❌ Never commit `.pbix` files — they are binary and not diffable
- ❌ Never commit secrets or connection strings

### Power BI

- ✅ Use **parameters** for data source paths/URLs to support multiple environments
- ✅ Use **TMDL format** for semantic models (better diffs than `.bim`)
- ✅ Keep reports and semantic models in **separate PBIP folders** when models are shared
- ✅ Disable auto-date table to reduce noise in diffs
- ❌ Avoid embedding credentials in reports

### `.gitignore` Recommendations

```gitignore
# Power BI local settings (machine-specific)
**/.pbi/localSettings.json

# Legacy binary format — do not version control
*.pbix

# OS files
.DS_Store
Thumbs.db
```

---

## Troubleshooting

**PBIP option not visible in Power BI Desktop**
Enable it under *File → Options and settings → Options → Preview features → Store reports using enhanced metadata format*.

**Git shows large diffs on every save**
Check if Power BI is regenerating GUIDs on each save. Enable the *"Preserve layout GUID"* option if available, or normalize JSON formatting using a pre-commit hook.

**Pipeline fails to authenticate**
Ensure the Service Principal has the **Power BI Admin** or appropriate workspace role, and that the `CLIENT_SECRET` has not expired in Azure AD.

**Report opens blank after clone**
Data source credentials are not stored in Git. Reconnect data sources in Power BI Desktop via *Transform Data → Data source settings*.

---

## Contributing

1. Fork or branch from `dev`
2. Follow the [branching strategy](#branching-strategy)
3. Submit a Pull Request with a clear description of changes
4. Link any related Azure Boards work items in the PR description

---

## License

This project is licensed under the [MIT License](LICENSE).

---

*Maintained by the BI Engineering Team · Questions? Open an issue or contact the team via Microsoft Teams.*
