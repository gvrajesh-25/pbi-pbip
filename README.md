# Power BI Version Control with PBIP & Azure Repos

A guide for implementing professional version control for Power BI projects using the **Power BI Project (PBIP)** format, **Azure Repos** (Git), and **Microsoft Fabric Deployment Pipelines**.

> ⚠️ **Important:** PBIP files **cannot be published directly** via the Power BI REST API. This guide uses the correct approach: **Fabric Git Integration** syncs PBIP content into workspaces, and **Fabric Deployment Pipelines** promote changes across environments.

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
| Microsoft Fabric | F SKU | Fabric capacity workspace required for Git Integration & Deployment Pipelines |

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

## How It Works

This solution has two distinct layers:

```
Azure Repos (Git)                   Microsoft Fabric
─────────────────                   ────────────────────────────────────────
                                    
  feature/* branch                  
       ↓ PR merge                   
    dev branch      ──Git Sync──→   Dev Workspace  ─┐
                                                     │  Fabric Deployment
                                                     │  Pipeline (GUI)
                                    Test Workspace ◄─┤
                                                     │
                                    Prod Workspace ◄─┘
```

- **Azure Repos** is the source of truth for all PBIP source files.
- **Fabric Git Integration** keeps the Dev workspace in sync with the `dev` branch automatically.
- **Fabric Deployment Pipelines** promote validated content from Dev → Test → Prod — no re-publishing needed.
- **Azure Pipelines** (optional) can trigger a Git sync via the Fabric REST API after a merge, ensuring the Dev workspace is always up to date.

---

## Setting Up Fabric Git Integration

This is a **one-time setup** per workspace. Once connected, the workspace stays in sync with the Azure Repo branch automatically.

### Step 1 — Connect Dev Workspace to Azure Repos

1. Open your **Dev Fabric workspace** in the browser.
2. Go to **Workspace settings → Git integration**.
3. Select **Azure DevOps** and authenticate.
4. Choose your **Organisation**, **Project**, **Repository**, and set the **Branch** to `dev`.
5. Set the **Git folder** to `/` (root) or a subfolder if your PBIP files are nested.
6. Click **Connect and sync**.

The workspace will import all items from the repo. From this point, every commit to `dev` can be synced into the workspace.

### Step 2 — Set Up Fabric Deployment Pipeline

1. In the Fabric portal, go to **Deployment pipelines → New pipeline**.
2. Name it (e.g., `PowerBI-CICD`) and create **3 stages**: Dev, Test, Prod.
3. Assign your workspaces to each stage:
   - **Dev stage** → Dev Workspace (the one connected to Git)
   - **Test stage** → Test Workspace
   - **Prod stage** → Prod Workspace
4. Click **Deploy** to promote from Dev → Test, then Test → Prod via the GUI.

> ✅ Deployment Pipelines handle content promotion — no script-based publishing of PBIP files required.

---

## CI/CD with Azure Pipelines (Git Sync Trigger)

Azure Pipelines cannot publish PBIP files directly. Instead, the pipeline calls the **Fabric REST API** to trigger a Git sync on the Dev workspace after a merge to `dev`. This ensures the workspace reflects the latest commit without manual intervention.

### Example Pipeline (`sync-dev.yml`)

```yaml
trigger:
  branches:
    include:
      - dev

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: Fabric-ServicePrincipal

steps:
  - checkout: self

  - task: PowerShell@2
    displayName: 'Get Entra ID Access Token'
    inputs:
      targetType: 'inline'
      script: |
        $body = @{
          grant_type    = "client_credentials"
          client_id     = "$(CLIENT_ID)"
          client_secret = "$(CLIENT_SECRET)"
          scope         = "https://api.fabric.microsoft.com/.default"
        }
        $response = Invoke-RestMethod `
          -Uri "https://login.microsoftonline.com/$(TENANT_ID)/oauth2/v2.0/token" `
          -Method POST -Body $body
        Write-Host "##vso[task.setvariable variable=ACCESS_TOKEN;issecret=true]$($response.access_token)"

  - task: PowerShell@2
    displayName: 'Trigger Fabric Git Sync on Dev Workspace'
    inputs:
      targetType: 'inline'
      script: |
        $headers = @{
          Authorization  = "Bearer $(ACCESS_TOKEN)"
          "Content-Type" = "application/json"
        }

        # Get latest commit SHA from the current build
        $commitSha = "$(Build.SourceVersion)"

        $body = @{
          workspaceHead = @{
            itemsToInclude = "All"
          }
          remoteCommitHash = $commitSha
          conflictResolution = @{
            conflictResolutionType  = "PreferRemote"
            conflictResolutionPolicy = "PreferRemote"
          }
        } | ConvertTo-Json -Depth 5

        # Trigger update from Git to workspace
        $uri = "https://api.fabric.microsoft.com/v1/workspaces/$(DEV_WORKSPACE_ID)/git/updateFromGit"
        Invoke-RestMethod -Uri $uri -Method POST -Headers $headers -Body $body
        Write-Host "Git sync triggered successfully for Dev workspace."
```

### Azure DevOps Variable Groups

Store secrets in **Azure DevOps Library → Variable Groups** (name: `Fabric-ServicePrincipal`):

| Variable | Description |
|----------|-------------|
| `TENANT_ID` | Azure AD / Entra ID Tenant ID |
| `CLIENT_ID` | Service Principal App (Client) ID |
| `CLIENT_SECRET` | Service Principal Secret |
| `DEV_WORKSPACE_ID` | Fabric Dev Workspace GUID |

> The Service Principal must be added as a **workspace Member or Admin** in the Fabric Dev workspace, and must have **Contributor** access to the Azure Repo.

### Promotion: Dev → Test → Prod

Promotion is handled manually (or via approval gate) using the **Fabric Deployment Pipeline GUI** or the Fabric REST API:

```powershell
# Optional: trigger deployment pipeline stage via REST API
$uri = "https://api.fabric.microsoft.com/v1/pipelines/$(PIPELINE_ID)/deployAll"
Invoke-RestMethod -Uri $uri -Method POST -Headers $headers
```


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

**Fabric Git sync fails with 403 / Unauthorized**
Ensure the Service Principal is added as a **Member or Admin** on the Fabric Dev workspace, and has **Contributor** access to the Azure Repo in DevOps.

**Workspace shows "Uncommitted changes" after sync**
This usually means a local workspace change (e.g., a refresh or manual edit) conflicts with the repo. Use **Update all** in the workspace Git integration panel to pull from the repo and discard workspace changes.

**Deployment Pipeline deploy button is greyed out**
The source stage must have no uncommitted Git changes. Sync the Dev workspace from Git first, then redeploy.

**Report opens blank after clone**
Data source credentials are not stored in Git. Reconnect data sources in Power BI Desktop via *Transform Data → Data source settings*, or set credentials in the Fabric workspace via **Settings → Data source credentials**.

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
