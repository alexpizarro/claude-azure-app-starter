# Claude Azure Starter — Architecture Patterns

This file is the Claude-consumable specification for the Claude Azure Starter template. Fetch it at the start of any new project to scaffold a fully deployable Azure web app using these proven patterns.

**Core promise:** One `git push` deploys your full stack (frontend + API + database + infrastructure) to Azure automatically.

---

## Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend | React + TypeScript + Vite | React 19, TS 5, Vite 6 |
| API | Azure Functions v4, Node.js | Node 22 LTS |
| Database | Azure SQL Serverless | Gen5, 1 vCore |
| IaC | Bicep | Subscription-scoped |
| CI/CD | GitHub Actions + OIDC | No client secrets |
| Hosting | Azure Static Web Apps (Free tier) | Includes managed functions |

---

## Naming Formula

```
{org}-{project}-{component}-{env}
```

| Token | Description | Example |
|---|---|---|
| `{org}` | Short org/company prefix | `acme` |
| `{project}` | Short app name | `taskapp` |
| `{component}` | Resource type | `rg`, `swa`, `sql`, `sqldb` |
| `{env}` | Environment | `test`, `prod` |

**Single source of truth:** `infra/environments/{env}.parameters.json` — set `org` and `project` once and all Azure resource names cascade from there.

### Resource name examples (org=acme, project=taskapp)

| Resource | Name |
|---|---|
| Resource Group | `acme-taskapp-rg-test` |
| Static Web App | `acme-taskapp-swa-test` |
| SQL Server | `acme-taskapp-sql-test` |
| SQL Database | `acme-taskapp-sqldb-test` |
| GitHub SPs | `acme-taskapp-github-test` |

---

## Repository Structure

```
.
├── .github/workflows/
│   ├── deploy-test.yml       # Triggers on push to 'test' branch
│   └── deploy-prod.yml       # Triggers on push to 'production' branch
├── infra/
│   ├── main.bicep            # Root — subscription-scoped. Naming params here.
│   ├── modules/
│   │   ├── resourceGroup.bicep
│   │   ├── staticWebApp.bicep    # SWA + SQL_CONNECTION_STRING app setting
│   │   ├── sqlServer.bicep       # SQL Server + serverless DB
│   │   └── functionApp.bicep     # FC1 standalone Function App (future use)
│   ├── environments/
│   │   ├── test.parameters.json  # Set org + project here for test
│   │   └── prod.parameters.json  # Set org + project here for prod
│   └── sql/migrations/           # Versioned, idempotent SQL scripts
├── frontend/                 # React + TypeScript + Vite
│   ├── src/
│   │   ├── App.tsx
│   │   └── services/api.ts
│   └── public/staticwebapp.config.json
├── api/                      # Azure Functions v4 — Node.js + TypeScript
│   ├── src/
│   │   ├── index.ts              # Entry point — import all functions here
│   │   ├── functions/
│   │   │   ├── hello.ts          # GET /api/hello — health check
│   │   │   ├── getItems.ts       # GET /api/items
│   │   │   └── createItem.ts     # POST /api/items
│   │   └── lib/
│   │       └── database.ts       # mssql connection pool
│   └── tsconfig.json             # CommonJS output — required for SWA managed functions
└── PATTERNS.md               # This file
```

---

## Key Architectural Decisions

### 1. SWA Managed Functions (not standalone Function App)
The `api/` folder is deployed by Azure Static Web Apps as a managed function — **frontend and API deploy together in one action**.
- Free tier, no separate Function App plan
- HTTP triggers only (no timers, no queues)
- `infra/modules/functionApp.bicep` exists for a future phase needing timer triggers or AI workloads (Flex Consumption)

### 2. Azure SQL Serverless
- SKU: `GP_S_Gen5_1` — scales to 0.5 vCores when idle, auto-pauses after 60 min
- Password auth (`sqladmin` user) — connection string injected as SWA app setting by Bicep at deploy time
- **Never** logged, output, or committed — password comes from GitHub secret at deploy time only

### 3. OIDC Auth — No Secrets to Rotate
Two service principals (`{org}-{project}-github-test`, `{org}-{project}-github-prod`) authenticate via OIDC federated credentials. Each SP is federated to its specific branch only — test SP cannot access prod.

### 4. Bicep Params — JSON Format Required
Parameter files are `.parameters.json` (not `.bicepparam`). This is intentional — JSON params can be combined with inline `--parameters key=value` for the SQL password. `.bicepparam` files cannot be supplemented this way (causes BCP258 error).

### 5. SWA Location is NOT australiaeast
Azure SWA does not support `australiaeast` as a billing region. SWA deploys to `eastasia`. All other resources use `australiaeast`. This is hardcoded in `infra/main.bicep` as `var swaLocation = 'eastasia'`.

---

## Branch → Deploy Mapping

| Branch | Deploys to | Notes |
|---|---|---|
| `main` | Nothing | Local dev only |
| `test` | Test resource group | Triggers deploy-test.yml |
| `production` | Prod resource group | Triggers deploy-prod.yml |

To deploy: merge `main` into `test` or `production` and push. Never commit directly to `test` or `production`.

---

## GitHub Actions Workflow (3 steps per environment)

**Step 1 — Deploy Infrastructure (Bicep)**
```bash
az deployment sub create \
  --location australiaeast \
  --template-file infra/main.bicep \
  --parameters @infra/environments/{env}.parameters.json \
  --parameters sqlAdminPassword="$SQL_PASSWORD"
```
Captures `swaToken` from output and masks it immediately.

**Step 2 — Run SQL Migrations**
`azure/sql-action@v2.3` — auto-manages firewall rules for the runner IP. Runs all `*.sql` files in `infra/sql/migrations/` alphabetically.

**Step 3 — Deploy to SWA**
`Azure/static-web-apps-deploy@v1` — Oryx builds TypeScript functions and React app, deploys both.

---

## GitHub Secrets Required (6 total)

| Secret | Scope |
|---|---|
| `AZURE_TENANT_ID` | Both envs |
| `AZURE_SUBSCRIPTION_ID` | Both envs |
| `AZURE_CLIENT_ID_TEST` | Test SP appId |
| `AZURE_CLIENT_ID_PROD` | Prod SP appId |
| `SQL_ADMIN_PASSWORD_TEST` | Never committed |
| `SQL_ADMIN_PASSWORD_PROD` | Never committed |

---

## SQL Migration Pattern

Every migration file follows this guard pattern:

```sql
-- NNN_describe_change.sql
IF NOT EXISTS (
    SELECT 1 FROM dbo.__MigrationHistory WHERE MigrationId = 'NNN_describe_change'
)
BEGIN
    -- DDL here

    INSERT INTO dbo.__MigrationHistory (MigrationId) VALUES ('NNN_describe_change');
    PRINT 'Migration NNN_describe_change applied.';
END
ELSE
BEGIN
    PRINT 'Migration NNN_describe_change already applied — skipping.';
END
```

Rules:
- File naming: `{NNN}_{description}.sql` (zero-padded, snake_case)
- `000_migration_history.sql` always runs first — creates the tracking table
- Never edit an applied migration — add a new one
- Roll forward only (no rollback scripts)

---

## API Conventions

- All functions: `authLevel: 'anonymous'`
- Routes declared in `app.http(...)` at the bottom of each function file
- `api/src/index.ts` imports all function files as side effects — add new functions here
- `database.ts` maintains a module-level connection pool — `sql.connect()` called once
- TypeScript compiles to CommonJS (`"module": "commonjs"`) — required for SWA managed functions
- `"main": "dist/index.js"` in `package.json` — must be a specific file path, not a glob

---

## Local Development

```bash
cp api/local.settings.json.example api/local.settings.json
# Fill SQL_CONNECTION_STRING with test environment values

# Terminal 1 — API on port 7071
cd api && npm start

# Terminal 2 — Frontend on port 5173 (proxies /api/* → 7071)
cd frontend && npm run dev
```

Local API connects directly to Azure test SQL (no local emulator needed).

---

## Critical Gotchas

| Problem | Cause | Fix |
|---|---|---|
| `BCP258: org/project missing` | Params not set | Set `org` and `project` in both `infra/environments/*.parameters.json` |
| `LocationNotAvailableForResourceType` | SWA doesn't support `australiaeast` | `swaLocation` is hardcoded to `eastasia` — do not change |
| Functions return 500 on first request after idle | SQL serverless auto-paused | Wait 30–60 seconds and retry |
| OIDC login fails with `AADSTS70021` | Federated credential subject mismatch | Subject must be `repo:{GithubOrg}/{Repo}:ref:refs/heads/{branch}` exactly |
| Deployment token warning in Bicep linter | `listSecrets` in outputs | Suppressed with `#disable-next-line outputs-should-not-contain-secrets` — required for GitHub Actions |

---

## Full Documentation

- `CLAUDE.md` — Operational guide for this specific project
- `ARCHITECTURE.md` — Full design document with diagrams
- `DEPLOY.md` — Step-by-step guide for deploying a renamed copy
- `README.md` — Setup instructions and Claude Code starter prompt
