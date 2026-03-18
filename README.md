# Claude Azure Starter

A production-ready reference architecture for building and deploying web apps to Azure with Claude Code. One `git push` deploys your full stack — React frontend, Azure Functions API, Azure SQL database, and all infrastructure — automatically.

**Stack:** React 19 + TypeScript · Azure Functions v4 (Node 22) · Azure SQL Serverless · Bicep IaC · GitHub Actions OIDC

See [PATTERNS.md](PATTERNS.md) for the Claude-consumable architecture spec, and [ARCHITECTURE.md](ARCHITECTURE.md) for the full design document.

---

## Using with Claude Code

Copy this prompt into Claude Code to start a new project using these architecture patterns:

```
I want to build [describe your app].

Use the Claude Azure Starter architecture patterns. First, fetch and read:
https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/claude-azure-starter/main/PATTERNS.md

My project naming:
- org: myorg       (short org/company prefix, used in Azure resource names)
- project: myapp   (short app name, used in Azure resource names)
- GitHub repo: YOUR_GITHUB_ORG/YOUR_REPO_NAME

Build: [what you want — e.g. "a landing page with a waitlist form and email capture",
        "a task manager with CRUD API and user-facing list view"]
```

Replace `YOUR_GITHUB_USERNAME`, `YOUR_GITHUB_ORG`, `YOUR_REPO_NAME`, `myorg`, `myapp`, and both `[...]` sections with your actual values.

> **Tip:** Enable the "Template repository" toggle in your GitHub repo Settings to add a "Use this template" button.

---

## Resource naming

Resources are named using the formula `{org}-{project}-{component}-{env}`. Set `org` and `project` once in `infra/environments/*.parameters.json` — all Azure resource names cascade from there.

| Resource | Example (org=acme, project=taskapp) |
|---|---|
| Resource Group | `acme-taskapp-rg-test` / `acme-taskapp-rg-prod` |
| Static Web App | `acme-taskapp-swa-test` / `acme-taskapp-swa-prod` |
| SQL Server | `acme-taskapp-sql-test` / `acme-taskapp-sql-prod` |
| SQL Database | `acme-taskapp-sqldb-test` / `acme-taskapp-sqldb-prod` |

---

## One-time setup

### 1. Create the GitHub repository

```bash
GITHUB_ORG="YOUR_GITHUB_ORG"
REPO="YOUR_REPO_NAME"

gh repo create $GITHUB_ORG/$REPO --public
git init
git remote add origin https://github.com/$GITHUB_ORG/$REPO.git
git checkout -b main
git add .
git commit -m "Initial commit"
git push -u origin main
```

Create the deployment branches:

```bash
git checkout -b test && git push -u origin test
git checkout -b production && git push -u origin production
git checkout main
```

### 2. Set your project names

Edit both parameter files and replace `myorg` and `myapp` with your values:

- `infra/environments/test.parameters.json`
- `infra/environments/prod.parameters.json`

### 3. Create Azure Service Principals (OIDC — no secret rotation needed)

```bash
SUBSCRIPTION_ID="<your-subscription-id>"
ORG="myorg"
PROJECT="myapp"
GITHUB_ORG="YOUR_GITHUB_ORG"
REPO="YOUR_REPO_NAME"

# Test SP
az ad sp create-for-rbac \
  --name "$ORG-$PROJECT-github-test" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json
# Note the appId (clientId) from the output

# Prod SP
az ad sp create-for-rbac \
  --name "$ORG-$PROJECT-github-prod" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json
# Note the appId (clientId) from the output
```

Add OIDC federated credentials to each SP:

```bash
# Test SP — replace CLIENT_ID_TEST with appId from above
az ad app federated-credential create \
  --id "<CLIENT_ID_TEST>" \
  --parameters '{
    "name": "github-test-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG/$REPO"':ref:refs/heads/test",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Prod SP
az ad app federated-credential create \
  --id "<CLIENT_ID_PROD>" \
  --parameters '{
    "name": "github-prod-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG/$REPO"':ref:refs/heads/production",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### 4. Add GitHub Actions Secrets

Go to **GitHub → Repository → Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|---|---|
| `AZURE_TENANT_ID` | `az account show --query tenantId -o tsv` |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_CLIENT_ID_TEST` | `appId` of the test SP |
| `AZURE_CLIENT_ID_PROD` | `appId` of the prod SP |
| `SQL_ADMIN_PASSWORD_TEST` | Strong password (min 8 chars, mixed case + number + symbol) |
| `SQL_ADMIN_PASSWORD_PROD` | Strong password for prod |

---

## Trigger a deployment

```bash
# Deploy to test
git checkout test && git merge main && git push origin test

# Deploy to production
git checkout production && git merge main && git push origin production

# Return to main
git checkout main
```

GitHub Actions runs three steps: deploy infrastructure (Bicep) → run SQL migrations → build and deploy app.

Watch progress at **GitHub → Actions**.

---

## Local development

```bash
# Install dependencies
cd api && npm install && cd ..
cd frontend && npm install && cd ..

# Configure local API settings
cp api/local.settings.json.example api/local.settings.json
# Edit SQL_CONNECTION_STRING with your test environment password
```

```bash
# Terminal 1 — Functions API (port 7071)
cd api && npm start

# Terminal 2 — React dev server (port 5173, proxies /api to 7071)
cd frontend && npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

---

## Project structure

```
.
├── .github/workflows/
│   ├── deploy-test.yml       # Triggers on push to 'test' branch
│   └── deploy-prod.yml       # Triggers on push to 'production' branch
├── infra/
│   ├── main.bicep            # Root Bicep — subscription-scoped
│   ├── modules/              # Bicep modules (resourceGroup, staticWebApp, sqlServer)
│   ├── environments/         # Per-environment parameter files (set org + project here)
│   └── sql/migrations/       # Versioned SQL migration scripts
├── frontend/                 # React + TypeScript + Vite
├── api/                      # Azure Functions v4 — Node.js + TypeScript
├── PATTERNS.md               # Claude-consumable architecture spec
└── ARCHITECTURE.md           # Full architecture design document
```

---

## Tech versions

| Technology | Version |
|---|---|
| React | 19.x |
| TypeScript | 5.x |
| Vite | 6.x |
| Azure Functions SDK | 4.x (`@azure/functions`) |
| Node.js (Functions runtime) | 22 LTS |
| mssql | 11.x |
| Bicep | Latest |
| GitHub Actions | ubuntu-latest |
