# Deploy a Copy of This App to Azure

This guide covers everything needed to take this codebase and deploy it as a new project under a different name and organization. It is written to be followed by a developer **or** by an AI coding assistant (Claude Code or similar).

---

## Overview of what this app deploys

| Layer | Technology | Azure Resource |
|---|---|---|
| Frontend | React 19 + TypeScript + Vite 6 | Azure Static Web App (Free tier) |
| API | Azure Functions v4, Node.js 22 | SWA managed functions (`api/` folder) |
| Database | SQL Server + Serverless DB | Azure SQL (`GP_S_Gen5_1`, auto-pause 15 min, 1 GB, LRS backup) |
| IaC | Bicep | Subscription-scoped deployment |
| CI/CD | GitHub Actions | OIDC — no secret rotation needed |

Two environments are deployed: **test** (push to `test` branch) and **production** (push to `production` branch).

---

## Part 1 — Rename the project

All project-specific naming follows this formula:

| Pattern | Formula | Example |
|---|---|---|
| Hyphenated (most resources) | `{org}-{project}-{component}-{env}` | `oat-baseapp-swa-test` |
| Concatenated (storage/KV — no hyphens) | `{org}{project}{component}{env}` | `oatbaseappsttest` |
| NPM packages | `{orgfull}-{project}-{component}` | `oatlatte-baseapp-api` |

Current values to replace:

| Token | Current value | Replace with |
|---|---|---|
| `{org}` — short org abbreviation | `oat` | your org abbreviation (e.g. `acme`) |
| `{project}` — short project name | `baseapp` | your project name (e.g. `myapp`) |
| `{orgfull}` — full org name for npm | `oatlatte` | your full org name (e.g. `acmecorp`) |
| `{Orgfull}` — title-case org name | `Oatlatte` | your org name title-cased (e.g. `Acmecorp`) |
| `{RepoName}` — GitHub repo name | `AzureAppArchitecture` | your new repo name (e.g. `my-app`) |
| `{GithubOrg}` — GitHub org or username | `alexpizarro` | your GitHub org or username |

### Instructions for AI assistant (Claude Code)

Paste the following prompt into Claude Code after cloning the repo. Fill in your values before running.

```
I need to rename this Azure project. Replace the following tokens across all files:

Current → New:
- "oat-baseapp" → "{org}-{project}"           (hyphenated resources)
- "oatbaseapp"  → "{org}{project}"             (concatenated, no hyphens: storage/KV)
- "oatlatte-baseapp" → "{orgfull}-{project}"   (npm package names)
- "oatlatte"    → "{orgfull}"                  (npm descriptions, comments)
- "Oatlatte"    → "{Orgfull}"                  (title-case: npm descriptions, comments)
- "oat-baseapp-github" → "{org}-{project}-github"  (service principal names in README)

Files to update (in this order):

1. infra/main.bicep
   - var org = 'oat'        →  var org = '{org}'
   - var project = 'baseapp' →  var project = '{project}'
   (All resource names derive from these two variables — no other changes needed in this file)

2. api/package.json
   - "name": "oatlatte-baseapp-api"  →  "name": "{orgfull}-{project}-api"
   - "description": "Azure Functions API — Oatlatte Base App"  →  update description

3. frontend/package.json
   - "name": "oatlatte-baseapp-frontend"  →  "name": "{orgfull}-{project}-frontend"

4. api/local.settings.json.example
   - "oat-baseapp-sql-test.database.windows.net"  →  "{org}-{project}-sql-test.database.windows.net"
   - "oat-baseapp-sqldb-test"  →  "{org}-{project}-sqldb-test"

5. README.md
   Replace all occurrences of:
   - "oat-baseapp-rg-test" → "{org}-{project}-rg-test"
   - "oat-baseapp-rg-prod" → "{org}-{project}-rg-prod"
   - "oat-baseapp-swa-test" → "{org}-{project}-swa-test"
   - "oat-baseapp-swa-prod" → "{org}-{project}-swa-prod"
   - "oat-baseapp-sql-test" → "{org}-{project}-sql-test"
   - "oat-baseapp-sql-prod" → "{org}-{project}-sql-prod"
   - "oat-baseapp-sqldb-test" → "{org}-{project}-sqldb-test"
   - "oat-baseapp-sqldb-prod" → "{org}-{project}-sqldb-prod"
   - "oat-baseapp-github-test" → "{org}-{project}-github-test"
   - "oat-baseapp-github-prod" → "{org}-{project}-github-prod"
   - "Oatlatte/AzureAppArchitecture" → "{GithubOrg}/{RepoName}"
   - GITHUB_ORG="Oatlatte" → GITHUB_ORG="{GithubOrg}"
   - REPO="AzureAppArchitecture" → REPO="{RepoName}"

6. ARCHITECTURE.md — apply same replacements as README.md plus:
   - "oatbaseappsttest" → "{org}{project}sttest"
   - "oatbaseappstprod" → "{org}{project}stprod"
   - "oatbaseappstblobtest" → "{org}{project}stblobtest"
   - "oatbaseappstblobprod" → "{org}{project}stblobprod"
   - "oatbaseappkvtest" → "{org}{project}kvtest"
   - "oatbaseappkvprod" → "{org}{project}kvprod"
   - "oatbaseappaisvctest" → "{org}{project}aisvctest"
   - "oatbaseappaisvcprod" → "{org}{project}aisvcprod"
   - "oatlatte-baseapp/" → "{orgfull}-{project}/"    (repo folder name in directory diagram)

7. After editing package.json files, regenerate lock files:
   cd api && npm install && cd ..
   cd frontend && npm install && cd ..

Verify: after all replacements, run:
   grep -r "oat-baseapp\|oatbaseapp\|oatlatte\|Oatlatte" . \
     --include="*.ts" --include="*.tsx" --include="*.bicep" \
     --include="*.json" --include="*.md" --include="*.yml" \
     --exclude-dir=node_modules --exclude-dir=.git
The result should be empty (zero matches).
```

### Manual rename (if not using an AI assistant)

Run these search-and-replace commands from the repo root. Substitute your values for `{org}`, `{project}`, `{orgfull}`, `{Orgfull}`, `{GithubOrg}`, `{RepoName}`:

```bash
# Set your values
ORG="acme"           # short org abbreviation
PROJECT="myapp"      # short project name
ORGFULL="acmecorp"   # full org name (npm)
ORGFULL_TITLE="Acmecorp"  # title-case org name
GITHUB_ORG="your-github-username"
REPO="your-repo-name"

# --- Core infrastructure (two-variable change) ---
sed -i '' "s/var org = 'oat'/var org = '$ORG'/" infra/main.bicep
sed -i '' "s/var project = 'baseapp'/var project = '$PROJECT'/" infra/main.bicep

# --- NPM package names ---
sed -i '' "s/oatlatte-baseapp-api/$ORGFULL-$PROJECT-api/" api/package.json
sed -i '' "s/oatlatte-baseapp-frontend/$ORGFULL-$PROJECT-frontend/" frontend/package.json
sed -i '' "s/Oatlatte Base App/$ORGFULL_TITLE $PROJECT/" api/package.json

# --- local.settings.json.example ---
sed -i '' "s/oat-baseapp-sql-test/$ORG-$PROJECT-sql-test/g" api/local.settings.json.example
sed -i '' "s/oat-baseapp-sqldb-test/$ORG-$PROJECT-sqldb-test/g" api/local.settings.json.example

# --- README.md ---
sed -i '' \
  -e "s/oat-baseapp/$ORG-$PROJECT/g" \
  -e "s/Oatlatte\/AzureAppArchitecture/$GITHUB_ORG\/$REPO/g" \
  -e "s/GITHUB_ORG=\"Oatlatte\"/GITHUB_ORG=\"$GITHUB_ORG\"/" \
  -e "s/REPO=\"AzureAppArchitecture\"/REPO=\"$REPO\"/" \
  README.md

# --- ARCHITECTURE.md ---
sed -i '' \
  -e "s/oat-baseapp/$ORG-$PROJECT/g" \
  -e "s/oatbaseapp/${ORG}${PROJECT}/g" \
  -e "s/oatlatte-baseapp/$ORGFULL-$PROJECT/g" \
  -e "s/oatlatte/$ORGFULL/g" \
  -e "s/Oatlatte/$ORGFULL_TITLE/g" \
  ARCHITECTURE.md

# --- Regenerate lock files ---
cd api && npm install && cd ..
cd frontend && npm install && cd ..
```

> **macOS note:** `sed -i ''` is macOS syntax. On Linux use `sed -i` (no trailing `''`).

---

## Part 2 — Prerequisites

Install these tools before proceeding:

```bash
# Azure CLI
brew install azure-cli          # macOS
# or: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli

# GitHub CLI
brew install gh                  # macOS
# or: https://cli.github.com

# Node.js 22 LTS
# https://nodejs.org or use nvm: nvm install 22

# Azure Functions Core Tools (for local dev only)
npm install -g azure-functions-core-tools@4
```

Log in to both CLIs:

```bash
az login
gh auth login
```

Confirm your Azure subscription:

```bash
az account show --query "{name:name, subscriptionId:id, tenantId:tenantId}" -o table
# If you have multiple subscriptions:
az account set --subscription "<your-subscription-id>"
```

Note your IDs — you will need them:

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Subscription: $SUBSCRIPTION_ID"
echo "Tenant:       $TENANT_ID"
```

---

## Part 3 — Create the GitHub repository

> Skip this section if you already have a repo and have cloned it.

```bash
# Create the repo (adjust visibility as needed)
gh repo create {GithubOrg}/{RepoName} --private

# If starting fresh (not cloned yet):
git init
git remote add origin https://github.com/{GithubOrg}/{RepoName}.git
git checkout -b main
git add .
git commit -m "Initial commit"
git push -u origin main
```

Create the two deployment branches:

```bash
git checkout -b test && git push -u origin test
git checkout -b production && git push -u origin production
git checkout main
```

---

## Part 4 — Create Azure Service Principals (OIDC)

Two separate service principals are created — one per environment. This isolates credentials so a test pipeline cannot access production.

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
GITHUB_ORG="{GithubOrg}"
REPO="{RepoName}"
ORG="{org}"
PROJECT="{project}"
```

### Create the test service principal

```bash
TEST_SP=$(az ad sp create-for-rbac \
  --name "$ORG-$PROJECT-github-test" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json 2>/dev/null)

CLIENT_ID_TEST=$(echo "$TEST_SP" | python3 -c "import sys,json; print(json.load(sys.stdin)['appId'])")
echo "Test client ID: $CLIENT_ID_TEST"
```

### Create the production service principal

```bash
PROD_SP=$(az ad sp create-for-rbac \
  --name "$ORG-$PROJECT-github-prod" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json 2>/dev/null)

CLIENT_ID_PROD=$(echo "$PROD_SP" | python3 -c "import sys,json; print(json.load(sys.stdin)['appId'])")
echo "Prod client ID: $CLIENT_ID_PROD"
```

### Add OIDC federated credentials

Each SP needs a federated credential tied to its branch. This replaces client secrets — nothing to rotate.

```bash
# Test SP — trusts pushes to the 'test' branch
az ad app federated-credential create \
  --id "$CLIENT_ID_TEST" \
  --parameters "{
    \"name\": \"github-test-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:$GITHUB_ORG/$REPO:ref:refs/heads/test\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

# Prod SP — trusts pushes to the 'production' branch
az ad app federated-credential create \
  --id "$CLIENT_ID_PROD" \
  --parameters "{
    \"name\": \"github-prod-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:$GITHUB_ORG/$REPO:ref:refs/heads/production\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"
```

### Grant User Access Administrator (required if Bicep creates role assignments)

If your project uses the FC1 Flex Consumption Function App (`infra/modules/functionApp.bicep`), Bicep will create `Microsoft.Authorization/roleAssignments` resources (Managed Identity → Storage). The `Contributor` role does **not** include `Microsoft.Authorization/roleAssignments/write` — the deployment will fail with a 403.

Grant `User Access Administrator` at the resource group scope to both SPs. The RGs may not exist yet on a first deploy — if so, scope to the subscription and narrow later.

```bash
SUB_ID=$(az account show --query id -o tsv)

# Test SP
TEST_SP_OID=$(az ad sp show --id "$CLIENT_ID_TEST" --query id -o tsv)
az role assignment create \
  --assignee "$TEST_SP_OID" \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$ORG-$PROJECT-rg-test"

# Prod SP
PROD_SP_OID=$(az ad sp show --id "$CLIENT_ID_PROD" --query id -o tsv)
az role assignment create \
  --assignee "$PROD_SP_OID" \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$ORG-$PROJECT-rg-prod"
```

> **Skip this step** if your Bicep contains no `Microsoft.Authorization/roleAssignments` resources (i.e., you are not using FC1 or any Managed Identity RBAC assignments). The base SWA + SQL deployment does not require it.

---

## Part 5 — Generate SQL admin passwords

SQL passwords must meet Azure complexity requirements: minimum 8 characters, must contain uppercase, lowercase, number, and symbol.

```bash
SQL_PASSWORD_TEST=$(openssl rand -base64 20 | tr -dc 'A-Za-z0-9!@#$%' | head -c 20)
SQL_PASSWORD_PROD=$(openssl rand -base64 20 | tr -dc 'A-Za-z0-9!@#$%' | head -c 20)
echo "Test SQL password:  $SQL_PASSWORD_TEST"
echo "Prod SQL password:  $SQL_PASSWORD_PROD"
```

**Save these passwords** — you will need the test password for local development.

---

## Part 6 — Set GitHub Actions secrets

Set all six secrets. These commands use the shell variables from the steps above.

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

gh secret set AZURE_TENANT_ID        --body "$TENANT_ID"
gh secret set AZURE_SUBSCRIPTION_ID  --body "$SUBSCRIPTION_ID"
gh secret set AZURE_CLIENT_ID_TEST   --body "$CLIENT_ID_TEST"
gh secret set AZURE_CLIENT_ID_PROD   --body "$CLIENT_ID_PROD"
gh secret set SQL_ADMIN_PASSWORD_TEST --body "$SQL_PASSWORD_TEST"
gh secret set SQL_ADMIN_PASSWORD_PROD --body "$SQL_PASSWORD_PROD"
```

Verify all six are set:

```bash
gh secret list
```

Expected output shows all six secrets with recent update timestamps.

---

## Part 7 — Install npm dependencies

The lock files must exist before the SWA action builds the project:

```bash
cd api && npm install && cd ..
cd frontend && npm install && cd ..
```

Commit the lock files if they are new:

```bash
git add api/package-lock.json frontend/package-lock.json
git commit -m "Add npm lock files"
git push origin main
```

---

## Part 8 — Deploy

Merge `main` into both deployment branches and push. GitHub Actions handles everything from here.

> **If your project uses Azure AI:** Verify the model version is available in your target region before deploying. Model version availability varies by region and changes over time.
> ```bash
> az cognitiveservices model list \
>   --location australiaeast \
>   --query "[?model.name=='gpt-4.1-mini'].{version:model.version, status:model.lifecycleStatus}" \
>   -o table
> ```
> The default model version in this template is `2025-04-14` (verified for `australiaeast`). Update your Bicep model deployment resource if the version shown differs.

```bash
# Deploy to test
git checkout test && git merge main && git push origin test

# Deploy to production
git checkout production && git merge main && git push origin production

# Return to main
git checkout main
```

Watch the runs:

```bash
gh run list --limit 4
gh run watch <run-id>
```

Each deployment run does three things in order:
1. **Deploy Infrastructure (Bicep)** — creates/updates the resource group, SQL server + database, and Static Web App
2. **Run SQL Migrations** — runs all `.sql` files in `infra/sql/migrations/` against the database
3. **Deploy to Azure Static Web Apps** — Oryx builds the React frontend and TypeScript Functions, then deploys both

A successful run takes approximately 3–5 minutes.

---

## Part 9 — Verify the deployment

### Get the deployed URLs

```bash
# Test environment
az staticwebapp list --resource-group {org}-{project}-rg-test \
  --query "[].defaultHostname" -o tsv

# Production environment
az staticwebapp list --resource-group {org}-{project}-rg-prod \
  --query "[].defaultHostname" -o tsv
```

### Test the API endpoints

```bash
# Replace with your actual SWA hostname
SWA_HOST="<your-swa-hostname>.azurestaticapps.net"

# Hello endpoint (no database)
curl https://$SWA_HOST/api/hello

# Items endpoint (requires SQL connection)
curl https://$SWA_HOST/api/items
```

Expected responses:

```json
// /api/hello
{ "message": "Hello from Azure Functions!", "timestamp": "...", "environment": "Development" }

// /api/items
{ "items": [] }
```

### Verify Azure resources exist

```bash
az resource list \
  --resource-group {org}-{project}-rg-test \
  --query "[].{name:name, type:type}" \
  -o table
```

Expected resources: one `Microsoft.Web/staticSites` and one `Microsoft.Sql/servers` (which contains the database).

---

## Part 10 — Local development setup

Local development connects the API directly to test environment Azure SQL.

### Prerequisites

- Node.js 22 LTS
- Azure Functions Core Tools v4: `npm install -g azure-functions-core-tools@4`
- Azure CLI logged in: `az login`

### Configure local settings

```bash
cp api/local.settings.json.example api/local.settings.json
```

Edit `api/local.settings.json` and fill in the SQL password:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_EXTENSION_VERSION": "~4",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "SQL_CONNECTION_STRING": "Server=tcp:{org}-{project}-sql-test.database.windows.net,1433;Database={org}-{project}-sqldb-test;User Id=sqladmin;Password=YOUR_TEST_SQL_PASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}
```

Replace `YOUR_TEST_SQL_PASSWORD` with the value from Part 5.

### Run locally

```bash
# Terminal 1 — Azure Functions (port 7071)
cd api && npm start

# Terminal 2 — React dev server (port 5173, proxies /api/* → port 7071)
cd frontend && npm run dev
```

Open http://localhost:5173.

---

## Troubleshooting

### Deployment fails at "Deploy Infrastructure (Bicep)"

**BCP258 — parameter missing from params file**

Symptom:
```
ERROR: BCP258: The following parameters are declared in the Bicep file but are missing an assignment in the params file: "sqlAdminPassword".
```

Cause: The workflow uses `--parameters @infra/environments/test.parameters.json` (JSON format with `@` prefix). The `.parameters.json` format allows inline `--parameters key=value` to supplement it. The older `.bicepparam` format does not — every required parameter without a default must be in the `.bicepparam` file itself.

Fix: Ensure the parameter files in `infra/environments/` are `.parameters.json` (not `.bicepparam`), and that the workflows use `@infra/environments/test.parameters.json` with the `@` prefix.

---

**SWA location not available**

Symptom:
```
"code": "LocationNotAvailableForResourceType",
"message": "The provided location 'australiaeast' is not available for resource type 'Microsoft.Web/staticSites'."
```

Cause: Azure Static Web Apps only support specific billing regions. `australiaeast` is not one of them.

Fix: In `infra/main.bicep`, the `swaLocation` variable is hardcoded to `eastasia`. If you're deploying to a different Azure region and want to change this, valid SWA locations are: `westus2`, `centralus`, `eastus2`, `westeurope`, `eastasia`.

```bicep
var swaLocation = 'eastasia'   // Change this if needed
```

---

**SQL migrations fail — firewall error**

Symptom: `Cannot open server requested by the login` or firewall-related error in the SQL Migrations step.

Cause: The `azure/sql-action@v2.3` action automatically adds and removes a firewall rule for the GitHub Actions runner IP. If the SQL server firewall rule `AllowAllAzureIPs` (0.0.0.0–0.0.0.0) is missing, Azure-internal services cannot connect.

Fix: The firewall rule is created by Bicep in `infra/modules/sqlServer.bicep`. Re-run the Bicep deployment to restore it.

---

**GitHub Actions OIDC login fails**

Symptom: `ClientAuthenticationFailed` or `AADSTS70021` in the Azure Login step.

Causes and fixes:

| Cause | Fix |
|---|---|
| Wrong `subject` in federated credential | The subject must exactly match `repo:{GithubOrg}/{RepoName}:ref:refs/heads/{branch}`. Re-create the credential. |
| Wrong client ID secret | Verify `AZURE_CLIENT_ID_TEST` matches the `appId` of the test SP: `az ad sp list --display-name "{org}-{project}-github-test" --query "[].appId" -o tsv` |
| SP doesn't have Contributor on subscription | Re-assign: `az role assignment create --assignee $CLIENT_ID_TEST --role Contributor --scope /subscriptions/$SUBSCRIPTION_ID` |

---

**Functions return 500 on /api/items**

Cause: SQL connection string is wrong or the SQL server is paused (auto-pause after 15 min of inactivity).

The first request after a pause takes 30–60 seconds while the serverless database resumes. Subsequent requests are fast.

To check if this is a connection string issue, read the function logs:

```bash
# Get recent logs from the SWA functions
az staticwebapp show \
  --name {org}-{project}-swa-test \
  --resource-group {org}-{project}-rg-test \
  --query "properties.defaultHostname" -o tsv
```

Then check GitHub Actions logs for the Bicep step output to confirm `SQL_CONNECTION_STRING` was set correctly.

---

## Everyday deploy workflow

After the one-time setup, all future deployments are a single merge + push:

```bash
# Make changes on main, then:
git checkout test && git merge main && git push origin test
# Watch GitHub Actions, then when ready:
git checkout production && git merge main && git push origin production
git checkout main
```

That's it. GitHub Actions handles infra + migrations + frontend + API in a single run.
