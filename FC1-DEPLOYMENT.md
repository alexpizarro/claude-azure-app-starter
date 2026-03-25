# Azure Flex Consumption Function App — Deployment Guide

A practical guide for deploying Node.js Azure Functions (v4 programming model, TypeScript) to
a **Flex Consumption (FC1)** plan via GitHub Actions. This document captures every pitfall
encountered and the exact commands that resolved them.

---

## Table of Contents

1. [Background: Why This Is Hard](#1-background-why-this-is-hard)
2. [Prerequisites](#2-prerequisites)
3. [Step 1 — Create a Storage Account](#3-step-1--create-a-storage-account)
4. [Step 2 — Create the FC1 App Service Plan](#4-step-2--create-the-fc1-app-service-plan)
5. [Step 3 — Create the Function App on the FC1 Plan](#5-step-3--create-the-function-app-on-the-fc1-plan)
6. [Step 4 — Configure App Settings](#6-step-4--configure-app-settings)
7. [Step 5 — Configure CORS](#7-step-5--configure-cors)
8. [Step 6 — Grant Managed Identity Roles](#8-step-6--grant-managed-identity-roles)
9. [Step 7 — Create a Service Principal for GitHub Actions](#9-step-7--create-a-service-principal-for-github-actions)
10. [Step 8 — Set GitHub Secrets](#10-step-8--set-github-secrets)
11. [Step 9 — Structure Your Code Correctly](#11-step-9--structure-your-code-correctly)
12. [Step 10 — The GitHub Actions Workflow](#12-step-10--the-github-actions-workflow)
13. [Step 11 — Verify the Deployment](#13-step-11--verify-the-deployment)
14. [Pitfalls & Lessons Learned](#14-pitfalls--lessons-learned)

---

## 1. Background: Why This Is Hard

Azure Flex Consumption is a distinct hosting model from the older Linux Consumption plan
(SKU `Y1/Dynamic`). They look identical in the portal UI but behave completely differently:

| | Regular Linux Consumption | Flex Consumption |
|---|---|---|
| SKU | `Y1/Dynamic` | `FC1/FlexConsumption` |
| Deployment method | `WEBSITE_RUN_FROM_PACKAGE` | **One Deploy** (blob-based) |
| `FUNCTIONS_WORKER_RUNTIME` setting | **Required** | **Forbidden** |
| Kudu SCM | Standard | Different endpoint |
| `az functionapp create --flexconsumption-location` | Silently falls back to Y1 | Should create FC1 |

**The CLI flag `--flexconsumption-location` does not reliably create a Flex Consumption plan
(tested with Azure CLI v2.83.0).** It silently places the app on the shared
`AustraliaEastLinuxDynamicPlan` (Y1/Dynamic) without any error. The only reliable approach is
to create the FC1 App Service Plan first via ARM REST API, then create the Function App via ARM
REST API pointing to that plan.

---

## 2. Prerequisites

- Azure CLI installed and logged in: `az login`
- GitHub CLI installed: `gh auth login`
- Node.js 20+ with `npm`
- TypeScript project using `@azure/functions` v4 (v4 programming model)
- A GitHub repository

Set these shell variables once — all commands below reference them:

```bash
SUBSCRIPTION_ID="your-subscription-id"
RESOURCE_GROUP="your-resource-group"
LOCATION="australiaeast"               # or your region
STORAGE_ACCOUNT="yourstorageaccount"   # globally unique, lowercase, 3-24 chars, no hyphens
FUNCTION_APP_NAME="your-function-app"
FC_PLAN_NAME="your-fc-plan"
GITHUB_REPO="owner/repo-name"
```

---

## 3. Step 1 — Create a Storage Account

Flex Consumption requires a storage account for blob-based deployment and host leases.

```bash
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2

# Create the deployment blob container
az storage container create \
  --name "app-package-${FUNCTION_APP_NAME}" \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login
```

> **Note:** The container name `app-package-<function-app-name>` is the convention used by the
> Azure Functions runtime for Flex Consumption deployments.

---

## 4. Step 2 — Create the FC1 App Service Plan

**Do NOT use `az appservice plan create --sku FC1`** — it may silently fail or create the wrong
plan type. Use the ARM REST API directly:

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/serverfarms/${FC_PLAN_NAME}?api-version=2023-12-01" \
  --body "{
    \"location\": \"${LOCATION}\",
    \"kind\": \"functionapp\",
    \"sku\": {
      \"name\": \"FC1\",
      \"tier\": \"FlexConsumption\",
      \"family\": \"FC\",
      \"size\": \"FC1\"
    },
    \"properties\": {
      \"reserved\": true
    }
  }"
```

Verify it was created correctly:

```bash
az appservice plan show \
  --name "$FC_PLAN_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "{name:name, sku:sku, status:status}" \
  -o json
```

Expected output — confirm `sku.name = FC1` and `sku.tier = FlexConsumption`:

```json
{
  "name": "your-fc-plan",
  "sku": {
    "family": "FC",
    "name": "FC1",
    "size": "FC1",
    "tier": "FlexConsumption"
  },
  "status": null
}
```

---

## 5. Step 3 — Create the Function App on the FC1 Plan

**Do NOT use `az functionapp create`** — it ignores `--plan` and places the app on the shared
dynamic plan. Use the ARM REST API directly:

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/sites/${FUNCTION_APP_NAME}?api-version=2023-12-01" \
  --body "{
    \"location\": \"${LOCATION}\",
    \"kind\": \"functionapp,linux\",
    \"identity\": { \"type\": \"SystemAssigned\" },
    \"properties\": {
      \"serverFarmId\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/serverfarms/${FC_PLAN_NAME}\",
      \"functionAppConfig\": {
        \"deployment\": {
          \"storage\": {
            \"type\": \"blobContainer\",
            \"value\": \"https://${STORAGE_ACCOUNT}.blob.core.windows.net/app-package-${FUNCTION_APP_NAME}\",
            \"authentication\": { \"type\": \"SystemAssignedIdentity\" }
          }
        },
        \"scaleAndConcurrency\": {
          \"maximumInstanceCount\": 100,
          \"instanceMemoryMB\": 2048
        },
        \"runtime\": {
          \"name\": \"node\",
          \"version\": \"22\"
        }
      }
    }
  }"
```

Verify the app is on the correct plan:

```bash
az functionapp show \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "{name:name, sku:properties.sku, serverFarm:properties.serverFarmId}" \
  -o json
```

Confirm `properties.sku = "FlexConsumption"` and `serverFarmId` ends with your FC plan name,
**not** `AustraliaEastLinuxDynamicPlan`.

---

## 6. Step 4 — Configure App Settings

**Do NOT set `FUNCTIONS_WORKER_RUNTIME`** — it is forbidden on Flex Consumption and causes
deployment failures. Runtime is specified in `functionAppConfig.runtime` (done in Step 3).

For managed-identity-based storage authentication, use `AzureWebJobsStorage__accountName`
(double underscore) instead of a connection string:

```bash
az functionapp config appsettings set \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --settings \
    "AzureWebJobsStorage__accountName=${STORAGE_ACCOUNT}" \
    "YOUR_OTHER_SETTING=value" \
    "ANOTHER_SETTING=value"
```

> **Note:** `WEBSITE_RUN_FROM_PACKAGE` and `WEBSITE_ENABLE_SYNC_UPDATE_SITE` are for regular
> Consumption only — do not set these on Flex Consumption apps.

---

## 7. Step 5 — Configure CORS

`az functionapp cors add` returns `Bad Request` on Flex Consumption apps. Use the ARM REST API:

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/sites/${FUNCTION_APP_NAME}/config/web?api-version=2023-12-01" \
  --body '{
    "properties": {
      "cors": {
        "allowedOrigins": [
          "https://your-frontend.azurestaticapps.net",
          "https://www.yourdomain.com",
          "http://localhost:5173"
        ],
        "supportCredentials": false
      }
    }
  }' \
  --query "properties.cors" \
  -o json
```

---

## 8. Step 6 — Grant Managed Identity Roles

After the ARM PUT, the response includes the app's managed identity `principalId`. Capture it:

```bash
PRINCIPAL_ID=$(az functionapp show \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "identity.principalId" \
  -o tsv)

STORAGE_ID=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --query id \
  -o tsv)

# Required for host lease management and deployment blob access
az role assignment create \
  --role "Storage Blob Data Owner" \
  --assignee "$PRINCIPAL_ID" \
  --scope "$STORAGE_ID"

# Required if using storage queues
az role assignment create \
  --role "Storage Queue Data Contributor" \
  --assignee "$PRINCIPAL_ID" \
  --scope "$STORAGE_ID"
```

> **Important:** RBAC propagation can take up to 10 minutes. If the app fails to start
> immediately after deployment, wait and retry.

---

## 9. Step 7 — Create a Service Principal for GitHub Actions

> **Note on auth approach:** The commands below use `--sdk-auth` / `AZURE_CREDENTIALS` (JSON
> secret). This approach is **deprecated** by Microsoft and requires a client secret that must
> be rotated. The preferred approach used in this project's CI/CD template is **OIDC federated
> credentials** — no client secret, no rotation. See `DEPLOY.md` Part 4 for the OIDC setup
> pattern. Use the OIDC approach for new projects.

If you are setting up a standalone FC1 Function App outside the main Bicep template and need a
quick service principal, the `--sdk-auth` approach still works:

```bash
az ad sp create-for-rbac \
  --name "your-app-github-deploy" \
  --role Contributor \
  --scopes "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}" \
  --sdk-auth \
  2>/dev/null | python3 -c "
import json, sys
data = sys.stdin.read()
start = data.find('{')
print(json.dumps(json.loads(data[start:]), indent=2))
"
```

> **Critical:** The `2>/dev/null` suppresses a `WARNING:` line that Azure CLI prepends to the
> output. If this WARNING text is included in the JSON secret, GitHub Actions authentication
> will fail silently. The Python snippet strips any leading non-JSON text.

The output is a JSON object like:

```json
{
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "...",
  "activeDirectoryEndpointUrl": "...",
  "resourceManagerEndpointUrl": "...",
  ...
}
```

Also grant the service principal `Storage Blob Data Contributor` on the storage account so it
can upload the deployment zip:

```bash
SP_APP_ID=$(az ad sp list --display-name "your-app-github-deploy" --query "[0].appId" -o tsv)

az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee "$SP_APP_ID" \
  --scope "$STORAGE_ID"
```

> **User Access Administrator required when Bicep creates role assignments:** If your Bicep
> deploys FC1 with a Managed Identity and creates `Microsoft.Authorization/roleAssignments`
> inline, the SP also needs `User Access Administrator` at the RG scope — `Contributor` alone
> will fail with 403. See `DEPLOY.md` Part 4 for the grant command.

---

## 10. Step 8 — Set GitHub Secrets

```bash
# Paste the full JSON from Step 7 when prompted
gh secret set AZURE_CREDENTIALS --repo "$GITHUB_REPO"

# The public URL of your function app
gh secret set VITE_AZURE_FUNCTION_URL \
  --body "https://${FUNCTION_APP_NAME}.azurewebsites.net" \
  --repo "$GITHUB_REPO"
```

---

## 11. Step 9 — Structure Your Code Correctly

### package.json

- `"type": "module"` — required for ESM
- `"main": "dist/index.js"` — **must be a specific file, not a glob pattern**. Glob patterns
  like `"dist/functions/*.js"` are not reliably supported across runtimes.
- Do **not** add `FUNCTIONS_WORKER_RUNTIME` to app settings — runtime is declared in
  `functionAppConfig.runtime`.

```json
{
  "name": "your-functions",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "npm run build && func start"
  },
  "dependencies": {
    "@azure/functions": "^4.5.0"
  },
  "devDependencies": {
    "@types/node": "^22",
    "typescript": "~5.8"
  }
}
```

### src/index.ts — Entry Point

Create a single entry point that imports all function files as side effects. The v4 programming
model registers functions when the module is imported.

```typescript
// src/index.ts
// Side-effect imports register all Azure Functions with the runtime
import "./functions/myHttpFunction.js";
import "./functions/myQueueFunction.js";
```

> **Note:** Use `.js` extensions in import paths even though the source files are `.ts`.
> TypeScript compiles to `.js` and Node.js ESM requires the extension.

> **Critical — every new function must be imported here:** Functions are **not auto-discovered**
> by filename. If you add `newFeature.ts` but forget to add `import "./functions/newFeature.js"`
> to `index.ts`, the route will return 404 after a successful deploy. The file compiling and
> deploying is not sufficient — the import is what calls `app.http(...)` and registers the
> route with the runtime.

### tsconfig.json

```json
{
  "compilerOptions": {
    "module": "ES2022",
    "moduleResolution": "node",
    "target": "ES2022",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### host.json

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "functionTimeout": "00:05:00"
}
```

### Deployment zip contents

The zip must contain:

```
dist/          ← compiled JS output
node_modules/  ← production dependencies only (no devDeps)
host.json
package.json
```

Do **not** include `src/`, `tsconfig.json`, `.env`, or `local.settings.json`.

---

## 12. Step 10 — The GitHub Actions Workflow

```yaml
name: Deploy Azure Functions

on:
  push:
    branches:
      - your-production-branch
    paths:
      - "azure-functions/**"

env:
  AZURE_FUNCTIONAPP_NAME: your-function-app  # the inkoff-mb-functions equivalent
  AZURE_RESOURCE_GROUP: your-resource-group

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    name: Build and Deploy Functions

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
          cache-dependency-path: azure-functions/package-lock.json

      # Install ALL deps (including devDeps) so TypeScript compiler is available
      - name: Install dependencies
        working-directory: azure-functions
        run: npm ci

      - name: Build TypeScript
        working-directory: azure-functions
        run: npm run build

      # Reinstall production-only deps to keep zip size minimal
      - name: Prune dev dependencies
        working-directory: azure-functions
        run: npm ci --omit=dev

      # Package: compiled dist/ + node_modules + host.json + package.json
      - name: Package for deployment
        working-directory: azure-functions
        run: |
          zip -r ../functions-deploy.zip \
            dist/ \
            node_modules/ \
            host.json \
            package.json

      - name: Log in to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # Pin to v1.5.2+ — this version auto-detects Flex Consumption via the
      # management API when using service principal auth (AZURE_CREDENTIALS).
      # It then uses One Deploy instead of WEBSITE_RUN_FROM_PACKAGE.
      # No 'sku: flexconsumption' parameter is needed with SP auth.
      - name: Deploy to Flex Consumption
        uses: Azure/functions-action@v1.5.2
        with:
          app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
          package: functions-deploy.zip
```

### Key workflow notes

- **`cache-dependency-path`**: The `package-lock.json` must exist and be committed to the repo.
  Run `npm install` locally first if it doesn't exist.
- **Build then prune**: Install all deps (including `typescript` devDep) to build, then
  reinstall with `--omit=dev` before packaging. This keeps the zip small.
- **`Azure/functions-action@v1.5.2`**: This is the minimum version with reliable Flex
  Consumption One Deploy auto-detection when using service principal auth.
- **No `sku: flexconsumption` parameter**: When authenticating with `AZURE_CREDENTIALS`
  (service principal), the action detects the SKU automatically via the management API.
  Adding `sku: flexconsumption` explicitly can interfere with auto-detection.

---

## 13. Step 11 — Verify the Deployment

### Check the GitHub Actions log

After the push, look for these exact lines in the "Deploy to Flex Consumption" step:

```
Detected function app sku: FlexConsumption        ← confirms correct plan detected
Package deployment using One Deploy initiated.    ← confirms correct deploy method
Successfully deployed web package to Function App.← success
```

If you see `Detected function app sku: Consumption` instead of `FlexConsumption`, the app is
on the wrong plan — go back to Step 3.

### Verify functions are registered

```bash
az functionapp function list \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  -o table
```

This should list all your functions with their invoke URLs.

### Verify the plan

```bash
az functionapp show \
  --name "$FUNCTION_APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query "{sku:properties.sku, serverFarm:properties.serverFarmId}" \
  -o json
```

`properties.sku` must be `"FlexConsumption"`.

---

## 14. Pitfalls & Lessons Learned

### Pitfall 1 — `az functionapp create --flexconsumption-location` silently fails

**Symptom:** The command succeeds with no error but the app ends up on
`AustraliaEastLinuxDynamicPlan` (SKU `Y1/Dynamic`).

**Root cause:** CLI v2.83.0 does not reliably provision Flex Consumption via this flag.

**Fix:** Create the FC1 App Service Plan first via `az rest --method PUT`, then create the app
via `az rest --method PUT` with `serverFarmId` pointing to that plan (see Steps 2 and 3).

---

### Pitfall 2 — ARM REST PUT on existing app doesn't change the hosting plan

**Symptom:** You PUT `functionAppConfig` properties on an existing app. The app accepts the
config but `serverFarmId` still points to the old dynamic plan.

**Fix:** Delete the app and recreate it. The hosting plan cannot be changed on an existing app.
If the app name is soft-deleted (Azure holds it for ~24h), use a new name.

---

### Pitfall 3 — `az appservice plan create --sku FC1` silently fails

**Symptom:** The command returns no error, but the plan is "Not Found" when queried.

**Fix:** Use `az rest --method PUT` with the explicit ARM body (see Step 2).

---

### Pitfall 4 — `AZURE_CREDENTIALS` has WARNING text prepended

**Symptom:** `az ad sp create-for-rbac ... | gh secret set` stores a secret that starts with
`WARNING: ...` before the JSON. GitHub Actions authentication fails silently.

**Fix:** Strip stderr before piping:

```bash
az ad sp create-for-rbac ... 2>/dev/null | python3 -c "
import json, sys
data = sys.stdin.read()
start = data.find('{')
print(json.dumps(json.loads(data[start:]), indent=2))
" | gh secret set AZURE_CREDENTIALS
```

---

### Pitfall 5 — `FUNCTIONS_WORKER_RUNTIME` causes "malformed content" deployment failure

**Symptom:** Deployment fails with "malformed content" or the action logs `"language: None
(V1 function app)"`.

**Root cause:** On Flex Consumption, runtime is declared in `functionAppConfig.runtime`. Setting
`FUNCTIONS_WORKER_RUNTIME` in app settings conflicts with this.

**Fix:** Remove `FUNCTIONS_WORKER_RUNTIME` from app settings. Never set it on Flex Consumption
apps.

---

### Pitfall 6 — `az functionapp cors add` returns `Bad Request` on Flex Consumption

**Symptom:** The CLI command fails with `Operation returned an invalid status 'Bad Request'`.

**Fix:** Set CORS via the ARM REST API (see Step 5).

---

### Pitfall 7 — `"main": "dist/functions/*.js"` glob pattern not resolved

**Symptom:** Functions deploy but are not registered. The runtime cannot find the entry point.

**Fix:** Use a concrete file path: `"main": "dist/index.js"`, and create `src/index.ts` that
imports each function file as a side effect.

---

### Pitfall 8 — Missing `package-lock.json` breaks CI cache step

**Symptom:** GitHub Actions fails at "Set up Node.js" because `cache-dependency-path` points to
a non-existent file.

**Fix:** Run `npm install` locally in the `azure-functions/` directory and commit the resulting
`package-lock.json`.

---

### Pitfall 9 — Publish profile auth fails with Kudu 401 on Flex Consumption

**Symptom:** Using `publish-profile` auth with `Azure/functions-action` fails with HTTP 401.

**Fix:** Switch to service principal auth:
1. Use `azure/login@v2` with `creds: ${{ secrets.AZURE_CREDENTIALS }}`
2. Remove `publish-profile` from the `Azure/functions-action` step
3. Pin to `Azure/functions-action@v1.5.2` or later

---

### Summary checklist

- [ ] FC1 plan created via `az rest --method PUT` (not CLI `az appservice plan create`)
- [ ] Function app created via `az rest --method PUT` with explicit `serverFarmId`
- [ ] `az functionapp show` confirms `properties.sku = "FlexConsumption"`
- [ ] `FUNCTIONS_WORKER_RUNTIME` is NOT in app settings
- [ ] `AzureWebJobsStorage__accountName` (double underscore) used instead of connection string
- [ ] Managed identity has `Storage Blob Data Owner` on the storage account
- [ ] Service principal has `Contributor` on resource group + `Storage Blob Data Contributor` on storage
- [ ] `AZURE_CREDENTIALS` secret contains clean JSON (no `WARNING:` text)
- [ ] `package.json` has `"main": "dist/index.js"` (not a glob)
- [ ] `src/index.ts` entry point imports all function files as side effects
- [ ] `package-lock.json` is committed to the repo
- [ ] Workflow uses `Azure/functions-action@v1.5.2` with no `sku:` parameter
- [ ] GitHub Actions log confirms `"Detected function app sku: FlexConsumption"`
- [ ] GitHub Actions log confirms `"Package deployment using One Deploy initiated."`
