# Claude Azure Starter

A scaffolding starter for building and deploying web apps to Azure with Claude Code. One `git push` deploys your full stack — React frontend, Azure Functions API, Azure SQL database, and all infrastructure — automatically.

**Stack:** React 19 + TypeScript · Azure Functions v4 (Node 22) · Azure SQL Serverless · Bicep IaC · GitHub Actions OIDC

**Target audience:** Personal projects, MVPs, hackathons, and small team prototypes. See [Enterprise readiness](#enterprise-readiness) at the bottom of this file if you are evaluating this for a production workload.

See [PATTERNS.md](PATTERNS.md) for the Claude-consumable architecture spec, [DEPLOY.md](DEPLOY.md) for the full step-by-step deployment guide, and [ARCHITECTURE.md](ARCHITECTURE.md) for the full design document.

---

## How to use this

**Option A — Let Claude build your app (recommended):**
Open Claude Code in an empty folder, paste the prompt below, and Claude fetches the architecture spec and builds your project from scratch. Then follow the one-time setup below to configure Azure and GitHub.

**Option B — Clone and rename:**
Fork this repo (or click "Use this template"), clone it locally, and follow [DEPLOY.md](DEPLOY.md) to rename the placeholders and configure Azure. Skip straight to the one-time setup below once renaming is done.

---

## Using with Claude Code

Open Claude Code in an **empty folder** where you want your project to live, then paste this prompt with your values filled in:

```
I want to build [describe your app].

Use the Claude Azure Starter architecture patterns. First, fetch and read:
https://raw.githubusercontent.com/alexpizarro/claude-azure-app-starter/main/PATTERNS.md

My project naming:
- org: myorg       (short org/company prefix, used in Azure resource names)
- project: myapp   (short app name, used in Azure resource names)
- GitHub repo: YOUR_GITHUB_ORG/YOUR_REPO_NAME

Build: [what you want — e.g. "a landing page with a waitlist form and email capture",
        "a task manager with CRUD API and user-facing list view"]
```

Replace `YOUR_GITHUB_ORG`, `YOUR_REPO_NAME`, `myorg`, `myapp`, and both `[...]` sections with your actual values.

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

> **Option B users (clone/fork):** Your repo already exists — skip Step 1 and go straight to Step 2.

### 1. Create the GitHub repository

*(Option A only — skip if you cloned or forked)*

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

---

## Enterprise readiness

This starter is intentionally free-tier-first and optimised for speed over operational maturity. Before using it for external-facing or enterprise workloads, you should understand the following gaps.

### Who this is for

| Suitable | Not suitable |
|---|---|
| Personal projects and side projects | Applications with external or untrusted users |
| MVPs and hackathon prototypes | Regulated industries (finance, healthcare, government) |
| Internal tools with a small trusted team | High-traffic or high-availability production workloads |
| Learning Azure and CI/CD patterns | Multi-team enterprise environments with compliance requirements |

### Known gaps

**Security**
- All API endpoints use `authLevel: 'anonymous'` — any internet user can read and write data. There is no authentication or authorisation configured.
- SQL Server has `publicNetworkAccess: Enabled` with AllowAllAzureIPs. There is no VNet, no private endpoint.
- The API connects to SQL using a password (stored in GitHub Secrets). Enterprise standard is Managed Identity with no password.
- GitHub Actions service principals are granted `Contributor` at subscription scope — over-permissioned relative to least-privilege best practice.
- No Web Application Firewall (WAF), no Azure Front Door, no DDoS protection.
- No Key Vault — the SQL password is injected at deploy time, not retrieved from Key Vault at runtime, and there is no automated rotation.

**Reliability**
- SQL Serverless auto-pauses after 15 minutes of inactivity. The first request after a pause takes 30–60 seconds — unacceptable for external-facing production workloads.
- Azure Static Web Apps Free tier has no SLA and caps bandwidth at 100 GB/month.
- Single-region deployment with no geo-redundancy or failover.
- No retry logic or circuit-breaker patterns in the API.

**Observability**
- Application Insights is not provisioned in the Bicep. `host.json` references it, but no resource is created. There is no centralised logging, structured logging, distributed tracing, or alerting.

**Testing and code quality**
- No automated tests (unit, integration, or end-to-end).
- No linting (ESLint not configured).
- No dependency or CVE scanning in CI.

**Operations**
- No tagging strategy beyond `environment` on the resource group.
- No cost management or budget alerts.
- No explicit rollback mechanism beyond redeploying a previous commit or Azure SQL point-in-time restore.

### Highest-priority upgrades for production

If you need to move this toward enterprise standards, address these in order:

1. **Add authentication** — enable Entra ID on the Static Web App or add API Management with JWT validation in front of the Functions.
2. **Switch to Managed Identity for SQL** — remove the SQL password; grant the Function App's system-assigned managed identity `db_datareader`/`db_datawriter` on the database.
3. **Scope SP permissions** — change the OIDC service principal role from subscription `Contributor` to the specific resource group only.
4. **Private networking** — disable `publicNetworkAccess` on the SQL Server, add a private endpoint in a VNet, and connect the Function App via VNet integration.
5. **Provision Application Insights** — add it to the Bicep and wire `APPLICATIONINSIGHTS_CONNECTION_STRING` into the Function App app settings.
6. **Upgrade SQL tier** — move from serverless (`GP_S_Gen5_1`, auto-pauses) to a provisioned vCore or DTU tier to eliminate cold starts.
7. **Add a WAF** — place Azure Front Door with a WAF policy in front of the Static Web App.
8. **Write tests** — add Vitest for the frontend and a test framework (e.g. Jest) for the API before deploying to production.
