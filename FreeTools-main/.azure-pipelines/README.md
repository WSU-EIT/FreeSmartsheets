# FreeTools Azure DevOps Pipelines

Reusable pipeline templates for analyzing Blazor projects with FreeTools.

## Templates

### `freetools-test-template.yml` - FreeCICD Integration ⭐

**Designed for FreeCICD pipelines.** Runs after deployment using the source snapshot artifact.

- Downloads source snapshot from build stage (no re-clone needed!)
- Extracts and analyzes the exact code that was deployed
- Captures screenshots from the live deployed URL
- Generates comprehensive report
- Publishes as pipeline artifact

**Use when:** Integrating with existing FreeCICD build-and-deploy pipelines.

### `freetools-analyze.yml` - Standalone Analysis

Full analysis pipeline for standalone use (clones repo fresh).

### `freetools-code-analysis.yml` - Code Only

Lightweight code metrics without screenshots.

---

## Quick Start: FreeCICD Integration

Replace `playwright-screenshot-template.yml` with `freetools-test-template.yml`:

```yaml
# BEFORE (in your build-and-deploy.yml):
- template: Templates/playwright-screenshot-template.yml@TemplateRepo
  parameters:
    testUrl: "https://azuredev.em.wsu.edu/FreeCICD"
    baseImageName: "playwright-dotnet-pwsh"
    projectImageName: "playwright-test-project"
    crawlerImageName: "playwright-link-crawler"
    poolName: "BuildVM"

# AFTER:
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://azuredev.em.wsu.edu/FreeCICD"
    sourceArtifactName: "$(CI_ProjectName).SourceSnapshot"
    dependsOn:
      - DeployDEVStage
```

That's it! The template will:
1. ✅ Download the source snapshot (created by `build-template.yml`)
2. ✅ Extract and analyze the code
3. ✅ Test endpoints on the live deployed site
4. ✅ Capture screenshots with smart SPA timing
5. ✅ Generate a markdown report with route maps, file metrics, screenshot gallery
6. ✅ Publish everything as a pipeline artifact

---

## Template Parameters

### freetools-test-template.yml (FreeCICD)

| Parameter | Required | Default | Description |
|-----------|:--------:|---------|-------------|
| `projectName` | ✅ | - | Project name (e.g., `$(CI_ProjectName)`) |
| `testUrl` | ✅ | - | Deployed URL to test |
| `sourceArtifactName` | | `{projectName}.SourceSnapshot` | Source snapshot artifact name |
| `poolName` | | `BuildVM` | Agent pool |
| `stageName` | | `FreeToolsTestStage` | Stage identifier |
| `stageDisplayName` | | `FreeTools Analysis` | Display name in pipeline |
| `dependsOn` | | `[]` | Stages to wait for |
| `maxThreads` | | `5` | Parallel browser instances |
| `pageSettleDelayMs` | | `3000` | SPA render wait time |
| `skipScreenshots` | | `false` | Code analysis only |
| `testCredentials` | | `''` | JSON auth credentials |
| `freetoolsRepo` | | `https://github.com/WSU-EIT/FreeTools.git` | FreeTools repo |
| `freetoolsBranch` | | `main` | FreeTools branch |

---

## Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FreeCICD + FreeTools Integration                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BuildStage (build-template.yml)                                            │
│  ├─► Checkout code                                                          │
│  ├─► Create source snapshot (.zip) ◄─── This is used by FreeTools!         │
│  ├─► Build & publish                                                        │
│  └─► Publish artifacts                                                      │
│           │                                                                 │
│           ▼                                                                 │
│  DeployDEVStage (deploy-template.yml)                                       │
│  └─► Deploy to https://azuredev.em.wsu.edu/MyApp                           │
│           │                                                                 │
│           ▼                                                                 │
│  FreeToolsTestStage (freetools-test-template.yml)                          │
│  ├─► Download source snapshot artifact                                      │
│  ├─► Extract code                                                           │
│  ├─► Clone FreeTools & build                                                │
│  ├─► WorkspaceInventory (analyze code)                                      │
│  ├─► EndpointMapper (discover routes)                                       │
│  ├─► EndpointPoker (test live endpoints)                                    │
│  ├─► BrowserSnapshot (capture screenshots from deployed site)               │
│  ├─► WorkspaceReporter (generate report)                                    │
│  └─► Publish FreeTools-{Project}-{BuildNumber} artifact                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Output Artifacts

Each run produces `FreeTools-{ProjectName}-{BuildNumber}`:

```
FreeTools-FreeCICD-20250101.1/
├── FreeCICD-Report.md           # Comprehensive markdown report
├── pages.csv                    # All discovered routes
├── workspace-inventory.csv      # File metrics
└── snapshots/
    ├── Account/
    │   └── Login/
    │       ├── default.png      # Screenshot
    │       ├── default.html     # HTTP response
    │       └── metadata.json    # Capture metadata
    └── ...
```

---

## Examples

### Basic DEV Testing

```yaml
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://azuredev.em.wsu.edu/$(CI_DEV_VirtualPath)"
    dependsOn:
      - DeployDEVStage
```

### With Authentication

```yaml
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://azuredev.em.wsu.edu/$(CI_DEV_VirtualPath)"
    testCredentials: '{"email": "$(TestEmail)", "password": "$(TestPassword)"}'
    dependsOn:
      - DeployDEVStage
```

### Test Both DEV and PROD

```yaml
# After DEV deploy
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://azuredev.em.wsu.edu/$(CI_DEV_VirtualPath)"
    stageName: "FreeToolsDevTest"
    stageDisplayName: "FreeTools Analysis (DEV)"
    dependsOn:
      - DeployDEVStage

# After PROD deploy
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://prod.em.wsu.edu/$(CI_PROD_VirtualPath)"
    stageName: "FreeToolsProdTest"
    stageDisplayName: "FreeTools Analysis (PROD)"
    dependsOn:
      - DeployPRODStage
```

### Code Analysis Only (Fast)

```yaml
- template: Templates/freetools-test-template.yml@TemplateRepo
  parameters:
    projectName: "$(CI_ProjectName)"
    testUrl: "https://azuredev.em.wsu.edu/$(CI_DEV_VirtualPath)"
    skipScreenshots: true
    dependsOn:
      - DeployDEVStage
```

---

## Migrating from playwright-screenshot-template

| playwright-screenshot-template | freetools-test-template |
|-------------------------------|-------------------------|
| Docker-based | Native .NET tools |
| Requires Docker daemon | Just .NET SDK |
| Custom crawler logic | Discovers routes from code |
| Screenshots only | Code metrics + routes + screenshots + report |
| Complex multi-image build | Single clone & build |
| Manual URL crawling | Reads `@page` directives |

---

## Requirements

- **Windows agent** (recommended for Playwright)
- **.NET 10 SDK** on the agent
- **Network access** to the deployed URL
- **Source snapshot artifact** from `build-template.yml`

---

## Troubleshooting

### "Source snapshot zip not found"
- Ensure `build-template.yml` ran successfully
- Check artifact name matches: `{ProjectName}.SourceSnapshot`

### Screenshots are blank
- Increase `pageSettleDelayMs` (try 5000-10000)
- Check if site requires VPN access

### Auth pages show login
- Provide `testCredentials` parameter
- Ensure test account exists

### FreeTools clone fails
- Check network access to GitHub
- Try using an internal mirror with `freetoolsRepo` parameter

---

## 📬 About

**FreeTools** is developed and maintained by **[Enrollment Information Technology (EIT)](https://em.wsu.edu/eit/meet-our-staff/)** at **Washington State University**.

📧 Questions or feedback? Visit our [team page](https://em.wsu.edu/eit/meet-our-staff/) or open an issue on [GitHub](https://github.com/WSU-EIT/FreeTools/issues).
