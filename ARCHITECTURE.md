# Oatlatte Azure Reference Architecture

**Status:** In Progress
**Author:** Oatlatte Engineering
**Date:** 2026-03-11
**Region:** Australia East (`australiaeast`)

---

## Overview

This is a reference architecture for Oatlatte Azure applications. It establishes a fully deployable baseline that new projects can be cloned from. The architecture covers a React frontend hosted on Azure Static Web Apps, an Azure Functions backend, temporary image storage with automatic cleanup, Azure AI Foundry LLM integration, and Azure SQL Server — all deployed across two isolated environments (test and production) via GitHub Actions.

**Key success measure:** Push a code change from an iOS device to a GitHub branch → GitHub Actions deploys all Azure resources automatically → change is live.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                            │
│                                                                     │
│   main branch ──────────────────────────────── (no auto-deploy)    │
│   test branch ──────────────────────────────── deploy-test.yml ──► │──┐
│   production branch ────────────────────────── deploy-prod.yml ──► │  │
└─────────────────────────────────────────────────────────────────────┘  │
                                                                         │
         ┌───────────────────────────────────┐  ┌─────────────────────────────────────┐
         │     oat-baseapp-rg-test            │  │     oat-baseapp-rg-prod             │
         │                                   │  │                                     │
         │  ┌─────────────────────────────┐  │  │  ┌─────────────────────────────┐   │
         │  │  oat-baseapp-swa-test        │  │  │  │  oat-baseapp-swa-prod        │  │
         │  │  Azure Static Web App        │  │  │  │  Azure Static Web App        │  │
         │  │  React + TypeScript + Vite   │  │  │  │  React + TypeScript + Vite   │  │
         │  └──────────────┬──────────────┘  │  │  └──────────────┬──────────────┘  │
         │                 │ /api/*           │  │                 │ /api/*           │
         │  ┌──────────────▼──────────────┐  │  │  ┌──────────────▼──────────────┐  │
         │  │  oat-baseapp-func-test        │  │  │  │  oat-baseapp-func-prod       │  │
         │  │  Azure Functions (Node 22)   │  │  │  │  Azure Functions (Node 22)   │  │
         │  │  - POST /api/upload-image    │  │  │  │  - POST /api/upload-image    │  │
         │  │  - POST /api/chat            │  │  │  │  - POST /api/chat            │  │
         │  │  - GET  /api/items           │  │  │  │  - GET  /api/items           │  │
         │  │  - POST /api/items           │  │  │  │  - POST /api/items           │  │
         │  │  - Timer: cleanupBlobs       │  │  │  │  - Timer: cleanupBlobs       │  │
         │  └───┬───────────┬─────────┬───┘  │  │  └───┬───────────┬─────────┬───┘  │
         │      │           │         │       │  │      │           │         │       │
         │   Blob        SQL DB     AI Foundry│  │   Blob        SQL DB     AI Foundry│
         │   Storage                          │  │   Storage                          │
         └───────────────────────────────────┘  └─────────────────────────────────────┘
```

---

## Naming Convention

**Formula:**
- With hyphens: `{org}-{project}-{component}-{env}`
- Without hyphens (storage/KV): `{org}{project}{component}{env}`

| Variable | Value |
|---|---|
| org | `oat` |
| project | `baseapp` |
| environments | `test`, `prod` |

### Resource Name Reference

| Resource Type | Abbreviation | Test Name | Production Name | Azure Limit |
|---|---|---|---|---|
| Resource Group | `rg` | `oat-baseapp-rg-test` | `oat-baseapp-rg-prod` | 90 chars |
| Static Web App | `swa` | `oat-baseapp-swa-test` | `oat-baseapp-swa-prod` | 40 chars |
| Function App | `func` | `oat-baseapp-func-test` | `oat-baseapp-func-prod` | 60 chars |
| App Service Plan | `asp` | `oat-baseapp-asp-test` | `oat-baseapp-asp-prod` | 40 chars |
| Storage Account (Functions) | `st` | `oatbaseappsttest` | `oatbaseappstprod` | 24 chars, no hyphens |
| Storage Account (Blobs) | `stblob` | `oatbaseappstblobtest` | `oatbaseappstblobprod` | 24 chars, no hyphens |
| SQL Server | `sql` | `oat-baseapp-sql-test` | `oat-baseapp-sql-prod` | 63 chars |
| SQL Database | `sqldb` | `oat-baseapp-sqldb-test` | `oat-baseapp-sqldb-prod` | 128 chars |
| AI Services | `aisvc` | `oatbaseappaisvctest` | `oatbaseappaisvcprod` | 64 chars, no hyphens |
| AI Foundry Hub | `aihub` | `oat-baseapp-aihub-test` | `oat-baseapp-aihub-prod` | 32 chars |
| AI Foundry Project | `aiproj` | `oat-baseapp-aiproj-test` | `oat-baseapp-aiproj-prod` | 32 chars |
| Key Vault | `kv` | `oatbaseappkvtest` | `oatbaseappkvprod` | 24 chars, no hyphens |
| Application Insights | `appi` | `oat-baseapp-appi-test` | `oat-baseapp-appi-prod` | 260 chars |
| Log Analytics Workspace | `log` | `oat-baseapp-log-test` | `oat-baseapp-log-prod` | 63 chars |

---

## Technology Stack

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Frontend framework | React | 19.x | Latest stable |
| Frontend language | TypeScript | 5.x | Strict mode |
| Frontend build tool | Vite | 6.x | SWA native support |
| Hosting | Azure Static Web Apps | Standard tier | Linked to Functions |
| Backend runtime | Azure Functions | v4 | Node.js model |
| Backend language | Node.js | 22 LTS | LTS for production |
| Backend language | TypeScript | 5.x | Compiled to JS |
| SQL client | mssql | 11.x | Connection pooling |
| Azure AI SDK | @azure-rest/ai-inference | Latest | AI Foundry integration |
| Azure Storage SDK | @azure/storage-blob | Latest | User delegation SAS |
| Azure Auth SDK | @azure/identity | Latest | DefaultAzureCredential |
| Infrastructure as Code | Bicep | Latest | Subscription-scoped |
| CI/CD | GitHub Actions | Latest | OIDC auth to Azure |
| LLM | gpt-4o-mini | 2024-07-18 | Via Azure AI Foundry |
| Database | Azure SQL | Latest | Standard S0 tier |
| Secret management | Azure Key Vault | Latest | Key Vault references |
| Observability | Application Insights | Latest | Workspace-based |

---

## Repository Structure

```
oatlatte-baseapp/
├── .github/
│   └── workflows/
│       ├── deploy-test.yml           # Triggers on push to 'test' branch
│       └── deploy-prod.yml           # Triggers on push to 'production' branch
│
├── infra/                            # All Bicep infrastructure (IaC)
│   ├── main.bicep                    # Root orchestrator, subscription-scoped
│   ├── modules/
│   │   ├── resourceGroup.bicep
│   │   ├── monitoring.bicep          # App Insights + Log Analytics
│   │   ├── keyVault.bicep
│   │   ├── storage.bicep             # Two storage accounts + blob container
│   │   ├── sqlServer.bicep           # SQL Server + database + firewall
│   │   ├── aiFoundry.bicep           # AI Services + Hub + Project + model deployment
│   │   ├── functionApp.bicep         # Function App + App Service Plan + app settings
│   │   ├── staticWebApp.bicep        # SWA resource
│   │   └── roleAssignments.bicep     # RBAC: Managed Identity → services
│   └── environments/
│       ├── test.bicepparam           # Non-secret params for test
│       └── prod.bicepparam           # Non-secret params for prod
│
├── frontend/                         # Azure Static Web App source
│   ├── src/
│   │   ├── components/
│   │   │   ├── ImageUploader.tsx     # Upload UI + calls /api/upload-image
│   │   │   └── ChatInterface.tsx     # Chat UI + calls /api/chat
│   │   ├── hooks/
│   │   │   ├── useImageUpload.ts
│   │   │   └── useChat.ts
│   │   ├── services/
│   │   │   └── api.ts                # Typed fetch wrapper for /api/*
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── index.html
│   ├── vite.config.ts                # /api proxy → localhost:7071 in dev
│   ├── staticwebapp.config.json      # SWA routing + security headers
│   ├── tsconfig.json
│   └── package.json
│
├── api/                              # Azure Functions backend
│   ├── src/
│   │   ├── functions/
│   │   │   ├── httpTrigger/
│   │   │   │   ├── uploadImage.ts    # POST /api/upload-image → SAS URL (15 min)
│   │   │   │   ├── chat.ts           # POST /api/chat → AI Foundry completion
│   │   │   │   ├── getItems.ts       # GET /api/items → SQL query
│   │   │   │   └── createItem.ts     # POST /api/items → SQL insert
│   │   │   └── timerTrigger/
│   │   │       └── cleanupBlobs.ts   # Every 5 min: delete expired blobs
│   │   └── lib/
│   │       ├── config.ts             # Validates all env vars at startup
│   │       ├── blobStorage.ts        # BlobServiceClient + SAS generation
│   │       ├── aiClient.ts           # AI Inference SDK singleton
│   │       └── database.ts           # mssql connection pool + query helper
│   ├── host.json
│   ├── local.settings.json           # NOT committed (.gitignore)
│   ├── local.settings.json.example   # Committed template
│   ├── tsconfig.json
│   └── package.json
│
├── infra/sql/
│   └── migrations/
│       └── 001_create_items_table.sql
│
├── .gitignore
├── ARCHITECTURE.md                   # This file
└── README.md
```

---

## Branch Strategy

| Branch | Purpose | Auto-Deploy | Azure Resources |
|---|---|---|---|
| `main` | Local development | None | Connects to test resources via `local.settings.json` |
| `test` | Test environment | On push → `deploy-test.yml` | `oat-baseapp-rg-test` (fully isolated) |
| `production` | Production environment | On push → `deploy-prod.yml` | `oat-baseapp-rg-prod` (fully isolated) |

---

## CI/CD Pipeline (GitHub Actions)

### Workflow Jobs (per environment)

```
Push to branch
  └── infra-deploy
        └── az deployment sub create (Bicep)
              → outputs: swaDeploymentToken, funcAppName, rgName
        ┌─────────────────────────┐
        ▼                         ▼
  api-deploy                frontend-deploy
  (depends on infra)         (depends on infra, parallel with api)
  npm ci && npm run build    npm ci && npm run build
  azure/functions-action     Azure/static-web-apps-deploy
```

### Authentication to Azure
- **OIDC federated credentials** — no client secrets to rotate
- Two separate service principals: one for test, one for prod (blast radius isolation)
- Service principals scoped to their respective resource group only

### GitHub Secrets Required

```
AZURE_TENANT_ID                    # Shared across environments
AZURE_SUBSCRIPTION_ID              # Shared across environments
AZURE_CLIENT_ID_TEST               # OIDC service principal for test deployments
AZURE_CLIENT_ID_PROD               # OIDC service principal for prod deployments
SQL_ADMIN_PASSWORD_TEST            # Injected to Bicep at deploy time only
SQL_ADMIN_PASSWORD_PROD            # Injected to Bicep at deploy time only
```

---

## Image Upload & Temporary URL Flow

**Design principle:** The browser never receives Azure credentials. It only receives a time-limited SAS URL generated server-side using the Function App's Managed Identity.

```
1. Browser sends file → POST /api/upload-image

2. Function uploads blob to 'temp-uploads' container
   Sets blob metadata:
     expiresAt = now + 15 minutes
     uploadedAt = now

3. Function generates User Delegation SAS URL
   Permission: read-only
   Expiry: 15 minutes from now
   (Uses Managed Identity — no storage account key needed)

4. Function returns { sasUrl, blobName, expiresAt }

5. Browser uses sasUrl for AI vision request or display

────────────────────────────────────────────────────
Timer Function: runs every 5 minutes

6. Lists all blobs in 'temp-uploads' with metadata
7. Deletes any blob where metadata.expiresAt < now
8. Logs deletion count to Application Insights
```

---

## Azure AI Foundry Integration

**Design principle:** Authentication via Managed Identity — no API keys stored anywhere.

- **SDK:** `@azure-rest/ai-inference` (Azure AI Inference REST client)
- **Auth:** `DefaultAzureCredential` — uses Managed Identity in Azure, `az login` token locally
- **Model:** `gpt-4o-mini` (deployment name: `gpt-4o-mini`, version: `2024-07-18`)
- **Client:** Module-level singleton, not recreated per request
- **Chat:** Client sends full conversation array each request (stateless Functions)
- **Vision:** If `imageUrl` (SAS URL) is provided with a chat message, it's injected as multimodal content

---

## Azure SQL Integration

**Design principle:** Connection string in Key Vault, read via Key Vault reference in App Settings.

- **Auth flow:** Bicep stores SQL connection string in Key Vault → Function App Settings reference it as `@Microsoft.KeyVault(SecretUri=...)` → Azure resolves at runtime
- **Client:** `mssql` npm package with a module-level connection pool (initialized once)
- **Migrations:** Versioned `.sql` files in `infra/sql/migrations/` — run as a separate GitHub Actions job using `sqlcmd` after infra deployment

---

## Secret & Config Management

| Config Item | Local Dev | Test / Prod |
|---|---|---|
| `BLOB_STORAGE_ACCOUNT_NAME` | `local.settings.json` | Bicep → App Settings (plaintext) |
| `SQL_CONNECTION_STRING` | `local.settings.json` | Key Vault secret → App Settings reference |
| `AI_PROJECT_ENDPOINT` | `local.settings.json` | Bicep → App Settings (plaintext) |
| SQL admin password | Not needed | GitHub Secret → Bicep parameter only |
| Azure credentials | `az login` | GitHub Secret (OIDC) → not stored in Azure |

**Layers:**
1. **Azure Key Vault** — sensitive secrets (SQL connection string)
2. **Azure App Settings** — non-sensitive config + Key Vault references
3. **GitHub Secrets** — deploy-time only (OIDC client IDs, SQL admin password)
4. **`local.settings.json`** — local dev only, never committed

---

## Local Development Setup

Developers on `main` branch run Functions and Vite locally but connect to **test environment resources** in Azure. This avoids the need for local SQL and AI emulators.

```
vite dev (port 5173)
  └── proxies /api/* → localhost:7071

func start (port 7071)
  └── reads local.settings.json
        ├── BLOB_STORAGE_ACCOUNT_NAME → oatbaseappstblobtest (Azure)
        ├── SQL_CONNECTION_STRING     → oat-baseapp-sql-test.database.windows.net (Azure)
        └── AI_PROJECT_ENDPOINT       → AI Foundry test endpoint (Azure)
```

**Prerequisites for developers:**
1. Node.js 22 LTS
2. Azure Functions Core Tools v4 (`npm install -g azure-functions-core-tools@4`)
3. Azure CLI (`brew install azure-cli`)
4. `az login` — enables `DefaultAzureCredential` for blob and AI access
5. RBAC grants on test resources (assigned by admin):
   - `Storage Blob Data Contributor` on `oatbaseappstblobtest`
   - SQL login on `oat-baseapp-sqldb-test`
   - `Cognitive Services OpenAI User` on `oatbaseappaisvctest`

**Quick start:**
```bash
git clone <repo-url>
cd oatlatte-baseapp

cp api/local.settings.json.example api/local.settings.json
# Edit SQL_CONNECTION_STRING with password from team vault

az login

# Terminal 1
cd api && npm install && npm run start

# Terminal 2
cd frontend && npm install && npm run dev
# → http://localhost:5173
```

---

## Infrastructure Deployment Order (Bicep)

Bicep modules are deployed in this dependency order:

```
1. Resource Group (subscription scope)
       ↓
2. Monitoring (App Insights + Log Analytics)    2. Key Vault    2. Storage (both accounts)
       ↓                                               ↓               ↓
3. SQL Server                                   3. AI Foundry (uses KV + Storage)
       ↓                                               ↓
4. Function App (uses all above — KV refs, storage names, AI endpoint, App Insights)
       ↓
5. Static Web App (independent, but needs RG)
       ↓
6. Role Assignments (Function App Managed Identity → Storage, KV, AI Services)
```

**RBAC granted to Function App Managed Identity:**
| Role | Resource | Purpose |
|---|---|---|
| Storage Blob Data Contributor | Blob storage account | Upload, read, delete blobs + generate user delegation SAS |
| Key Vault Secrets User | Key Vault | Read SQL connection string secret |
| Cognitive Services OpenAI User | AI Services account | Call gpt-4o-mini endpoint |

---

## Verification Checklist

After deploying to test environment:

- [ ] `az resource list --resource-group oat-baseapp-rg-test` shows all 12+ resources
- [ ] SWA URL loads React app (no blank page, no 404)
- [ ] `POST /api/upload-image` returns a valid SAS URL
- [ ] SAS URL is publicly accessible (HTTP 200) immediately after upload
- [ ] SAS URL returns HTTP 403/404 after 15+ minutes (or after timer cleanup)
- [ ] `POST /api/chat` with a text message returns an AI completion
- [ ] `POST /api/items` creates a record; `GET /api/items` returns it
- [ ] App Insights logs show timer function ran and reports deletion count
- [ ] Push from iOS → GitHub Actions green → change visible at SWA URL

---

## Guiding Principles

1. **Simple git deployment is the top priority.** A push to a branch must automatically deploy all required resources — no manual steps post-push.
2. **Minimise cost.** Use free tiers first, then consumption-based, then fixed-cost tiers. Cost tier changes can be made manually per project.
3. **Single Azure subscription** for both environments (isolated by resource group).

## Architectural Decisions (Peer Review Resolved)

| Decision | Resolution | Rationale |
|---|---|---|
| Region | `australiaeast` default; change per project | Supported for all services including SWA |
| SWA + Functions | **SWA managed functions** (`api/` folder) | Single deploy action covers frontend + API — simplest git-push deployment |
| Functions plan | **SWA Free tier** (managed functions included) | Zero additional cost |
| SQL tier | **General Purpose Serverless** (`GP_S_Gen5_1`) | Auto-pauses when idle, cheapest SQL option for dev/test |
| SQL auth | **Password in SWA app settings** (set via Bicep) | Simple; Key Vault optional upgrade for future projects |
| AI Foundry Hub | **No Hub** — use AI Services + deployments directly | Hub adds cost and complexity with no benefit for simple reference |
| Cold start | **Acceptable** | Consumption/Serverless cold starts are fine; upgrade to Premium per project if needed |
| Subscription isolation | **Single subscription**, isolated by resource group | Simpler billing and management for reference architecture |
| Storage SAS auth | **User delegation SAS via Managed Identity** | No stored credentials; most secure option |
| Standalone Function App plan | **Flex Consumption (FC1)** — never Linux Consumption (Y1) | Y1 is deprecated; FC1 is the modern replacement with blob-based One Deploy |

---

## Standalone Function App (Flex Consumption)

SWA managed functions handle HTTP endpoints for the hello world. A **standalone Flex Consumption Function App** is required for features that SWA managed functions cannot support:

- Timer triggers (blob cleanup, scheduled jobs)
- Background processing
- Longer-running operations

### FC1 vs Linux Consumption (Y1) — Why it matters

| | Linux Consumption (Y1) | Flex Consumption (FC1) |
|---|---|---|
| SKU | `Y1/Dynamic` | `FC1/FlexConsumption` |
| Status | **Deprecated** | Current, recommended |
| Deployment | `WEBSITE_RUN_FROM_PACKAGE` | **One Deploy** (blob-based zip) |
| `FUNCTIONS_WORKER_RUNTIME` setting | Required | **Forbidden** (causes deploy failure) |
| `AzureWebJobsStorage` | Connection string | `__accountName` (double underscore, MI auth) |
| Runtime config | App settings | `functionAppConfig.runtime` |
| Bicep/CLI creation | Works | **CLI flags silently fail — Bicep ARM only** |

### Creating FC1 Resources — Bicep is Reliable, CLI is Not

The Azure CLI flags `--sku FC1` and `--flexconsumption-location` silently fall back to Y1/Dynamic without any error (tested CLI v2.83.0). The `infra/modules/functionApp.bicep` module uses Bicep's ARM API directly, which is the equivalent of `az rest --method PUT` and creates FC1 correctly every time.

**Verify after deploy:**
```bash
az functionapp show \
  --name "oat-baseapp-func-test" \
  --resource-group "oat-baseapp-rg-test" \
  --query "{sku:properties.sku, serverFarm:properties.serverFarmId}" \
  -o json
# properties.sku must be "FlexConsumption"
# serverFarmId must NOT end with "LinuxDynamicPlan"
```

### FC1 App Settings Rules

```
✅ AzureWebJobsStorage__accountName = <storageAccountName>   (double underscore, MI auth)
✅ SQL_CONNECTION_STRING = <value>                            (if SQL needed)
✅ AI_PROJECT_ENDPOINT = <value>                             (if AI needed)

❌ FUNCTIONS_WORKER_RUNTIME     — FORBIDDEN on FC1, causes "malformed content" failure
❌ WEBSITE_RUN_FROM_PACKAGE     — Y1 only, not applicable to FC1
❌ WEBSITE_ENABLE_SYNC_UPDATE_SITE — Y1 only, not applicable to FC1
```

### Required RBAC for FC1 Function App Managed Identity

| Role | Scope | Purpose |
|---|---|---|
| `Storage Blob Data Owner` | Function App's storage account | Host lease + deployment blob read + user delegation SAS |
| `Storage Blob Data Contributor` | Blob upload storage | Read/write/delete user-uploaded blobs |
| `Cognitive Services OpenAI User` | AI Services account | Call LLM endpoints |

**Note:** RBAC propagation can take up to 10 minutes. If the app fails immediately after deploy, wait and retry.

### GitHub Actions Deployment for Standalone FC1

```yaml
- name: Deploy Functions
  uses: Azure/functions-action@v1.5.2   # Minimum version with reliable FC1 auto-detection
  with:
    app-name: oat-baseapp-func-test
    package: functions-deploy.zip       # Pre-built zip: dist/ + node_modules/ + host.json + package.json
    # No 'sku: flexconsumption' parameter — auto-detected via management API
    # No publish-profile — use azure/login (OIDC) before this step
```

**Build and package pattern (GitHub Actions):**
```yaml
- run: npm ci                          # Install all deps (typescript devDep needed for build)
- run: npm run build                   # Compile TypeScript → dist/
- run: npm ci --omit=dev              # Reinstall prod-only deps (smaller zip)
- run: zip -r ../deploy.zip dist/ node_modules/ host.json package.json
```

**Verify in Actions log — look for these exact lines:**
```
Detected function app sku: FlexConsumption   ← correct plan detected
Package deployment using One Deploy initiated.
Successfully deployed web package to Function App.
```
If you see `Detected function app sku: Consumption` → the app is on Y1, not FC1. Recreate it.

### Code Structure for Standalone FC1

```json
// package.json — specific file path required, glob patterns not supported
{ "main": "dist/index.js" }
```

```typescript
// src/index.ts — imports register functions as side effects
import './functions/hello.js';      // .js extension required for ESM
import './functions/cleanup.js';
```

```json
// tsconfig.json for ESM (recommended for standalone FC1)
{ "compilerOptions": { "module": "ES2022", "moduleResolution": "node" } }
```

> **Note for SWA managed functions** (`api/` folder): CommonJS (`"module": "commonjs"`) works fine and is simpler with Oryx. ESM is specifically recommended for standalone FC1 apps.

### Pitfall: Existing App Cannot Change Hosting Plan

If a Function App was accidentally created on Y1 and you need to move it to FC1:
1. Delete the app: `az functionapp delete --name <name> --resource-group <rg>`
2. Wait ~24h if the name is soft-deleted, or choose a new name
3. Redeploy via Bicep — the `functionApp.bicep` module creates it on FC1

### Full Pitfall Reference

See [azure-flex-consumption-deployment-guide.md](azure-flex-consumption-deployment-guide.md) for every pitfall encountered and verified fixes.
