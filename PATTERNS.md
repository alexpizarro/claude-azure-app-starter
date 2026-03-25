# Claude Azure Starter — Architecture Patterns

This file is the Claude-consumable specification for the Claude Azure Starter template. Fetch it at the start of any new project to scaffold a fully deployable Azure web app using these proven patterns.

**Core promise:** One `git push` deploys your full stack (frontend + API + database + infrastructure) to Azure automatically.

---

## Scaffolding Checklist

When building a new project from these patterns, generate ALL of the following:

**Infrastructure & config**
- `infra/main.bicep` + `infra/modules/` + `infra/environments/{test,prod}.parameters.json`
- `infra/sql/migrations/000_migration_history.sql` + `001_create_items_table.sql`
- `.github/workflows/deploy-test.yml` + `deploy-prod.yml`
- `.gitignore` (include `api/local.settings.json` and `learnings/`)

**Frontend**
- `frontend/` — React 19 + TypeScript + Vite 6
- `frontend/public/staticwebapp.config.json` — SWA routing + security headers
- `frontend/vite.config.ts` — proxy `/api/*` → `localhost:7071` in dev

**API**
- `api/` — Azure Functions v4, Node 22, TypeScript (CommonJS output)
- `api/src/index.ts` + `api/src/functions/hello.ts` + `api/src/lib/database.ts`
- `api/local.settings.json.example` — empty string values, `__HINT_*` keys for format
- `api/package.json` — use exact versions from the Required package versions section below

**Project documentation (required — do not skip)**
- `README.md` — project-specific setup instructions
- `CLAUDE.md` — operational guide for this project (see instructions below)

### Generating CLAUDE.md for the derived project

`CLAUDE.md` is the persistent context file that enables Claude Code to work effectively in all future sessions on this project. **Always generate it.** It must contain:

1. **What this project is** — one paragraph describing the app being built
2. **Stack** — same as this document's Stack section
3. **Naming convention** — the formula table with the actual `org` and `project` values substituted
4. **Branch → deploy mapping** — with the actual GitHub repo (`{GithubOrg}/{RepoName}`) filled in
5. **Architecture decisions** — the same 5 decisions from this document
6. **SQL migration system** — identical rules and guard-clause template
7. **GitHub Actions workflow anatomy** — the same 3-step description
8. **GitHub secrets** — the same 6 secrets table
9. **API conventions** — same list, including `@types/*` rule and required package versions
10. **Local development** — same instructions
11. **Known deployment gotchas** — same table (all gotchas apply equally to derived projects)

Use the starter's `CLAUDE.md` at `https://raw.githubusercontent.com/alexpizarro/claude-azure-app-starter/main/CLAUDE.md` as the structural template. Substitute actual values throughout — do not leave `{org}`, `{project}`, or `YOUR_GITHUB_ORG` as placeholders.

---

## Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend framework | React | 19.x |
| Frontend language | TypeScript | 5.x |
| Frontend build tool | Vite | 6.x |
| Backend runtime | Azure Functions | v4 (Node.js model) |
| Backend language | Node.js | **22 LTS** |
| Backend language | TypeScript | 5.x |
| SQL client | mssql | 11.x |
| Azure AI SDK | @azure-rest/ai-inference | Latest (Azure AI Foundry — NOT direct OpenAI) |
| Azure Auth SDK | @azure/identity | Latest |
| Infrastructure as Code | Bicep | Latest (subscription-scoped) |
| CI/CD | GitHub Actions + OIDC | No client secrets |
| Hosting | Azure Static Web Apps | Free tier (includes managed functions) |
| Database | Azure SQL Serverless | GP_S_Gen5_1 (auto-pauses after 15 min) |

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
- `infra/modules/functionApp.bicep` exists for a future phase needing timer triggers or AI workloads (Flex Consumption) — see `FC1-DEPLOYMENT.md` for the full deployment guide

### 2. Azure SQL Serverless
- SKU: `GP_S_Gen5_1` — scales to 0.5 vCores when idle, auto-pauses after 15 min, 1 GB max size, locally-redundant backup
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
- Add `"engines": { "node": "22" }` to `api/package.json` to enforce the correct Node.js runtime

### Required package versions for `api/package.json`

Use these exact versions when generating the API package.json. Do not substitute older versions.

```json
{
  "engines": { "node": "22" },
  "dependencies": {
    "@azure/functions": "^4.5.0",
    "mssql": "^11.0.1"
  },
  "devDependencies": {
    "@types/mssql": "^9.1.5",
    "@types/node": "^22.0.0",
    "typescript": "^5.7.3"
  }
}
```

When adding packages for external services (Azure AI, Blob Storage, etc.), also add `@types/{package}` to devDependencies for any package that does not bundle its own `.d.ts` files.

---

## AI Integration

**Always use Azure AI Foundry — never direct OpenAI.**

This architecture uses `@azure-rest/ai-inference` authenticated via `DefaultAzureCredential` (Managed Identity in Azure, `az login` locally). There is no API key — do not use `OPENAI_API_KEY` or the `openai` npm package.

| Setting | Value |
|---|---|
| SDK | `@azure-rest/ai-inference` + `@azure/identity` |
| Auth | `DefaultAzureCredential` — no API key |
| Default model | `gpt-4.1-mini-2025-04-14` |
| Config env var | `AI_PROJECT_ENDPOINT` (set in `local.settings.json` locally, App Settings in Azure) |

### Mock pattern for local dev (before Azure AI is provisioned)

Check for `AI_PROJECT_ENDPOINT`, not an API key:

```typescript
if (!process.env.AI_PROJECT_ENDPOINT) {
  context.warn('AI_PROJECT_ENDPOINT not set — returning mock response');
  return { status: 200, jsonBody: { result: '[MOCK] Set AI_PROJECT_ENDPOINT to get real output.' } };
}
// real Azure AI Foundry call below
```

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

**Running before Azure is provisioned**

Leave `SQL_CONNECTION_STRING` and any external API keys as empty strings in `local.settings.json`. Functions should check for key presence and return mock responses when absent — this allows the full UI to be tested before Azure resources exist.

Pattern for external service calls (example uses Azure AI — see AI Integration section):
```typescript
if (!process.env.AI_PROJECT_ENDPOINT) {
  context.warn('AI_PROJECT_ENDPOINT not set — returning mock response');
  return { status: 200, jsonBody: { result: '[MOCK] Set AI_PROJECT_ENDPOINT to get real output.' } };
}
// real Azure AI Foundry call below
```

Pattern for optional DB writes:
```typescript
if (process.env.SQL_CONNECTION_STRING) {
  await saveToDB(...);
} else {
  context.warn('SQL_CONNECTION_STRING not set — skipping DB write');
}
```

---

## Critical Gotchas

| Problem | Cause | Fix |
|---|---|---|
| `BCP258: org/project missing` | Params not set | Set `org` and `project` in both `infra/environments/*.parameters.json` |
| `LocationNotAvailableForResourceType` | SWA doesn't support `australiaeast` | `swaLocation` is hardcoded to `eastasia` — do not change |
| Functions return 500 on first request after idle | SQL serverless auto-paused | Wait 30–60 seconds and retry |
| OIDC login fails with `AADSTS70021` | Federated credential subject mismatch | Subject must be `repo:{GithubOrg}/{Repo}:ref:refs/heads/{branch}` exactly |
| Deployment token warning in Bicep linter | `listSecrets` in outputs | Suppressed with `#disable-next-line outputs-should-not-contain-secrets` — required for GitHub Actions |
| `error TS7016: Could not find a declaration file for module 'mssql'` | `@types/mssql` missing from devDependencies | Add `"@types/mssql": "^9.1.5"` to `api/package.json` devDependencies. Always check `@types/*` for packages that don't bundle their own `.d.ts` files. |
| Placeholder strings in `local.settings.json` cause cryptic errors | Non-empty placeholder strings fool `if (!value)` checks | Use `""` for all user-input values in the example file; add `__HINT_*` keys for format documentation |
| `Multiple files found matching pattern *.sql` | `azure/sql-action@v2.3` accepts only a single file — fails with >1 migration | Use a `sqlcmd` bash loop instead — see workflow files for the complete pattern |
| `sqlcmd: No such file or directory` (exit 127) | `sqlcmd` is not pre-installed on `ubuntu-latest` (ubuntu-24.04 as of 2026) | Install `mssql-tools18` explicitly via the Microsoft apt repo at the start of the migration step |
| `gpg: cannot open '/dev/tty': No such device or address` | `gpg --dearmor` without `--batch` tries to open a TTY in headless CI | Use `gpg --batch --yes --dearmor` and pipe output through `sudo tee` — never `sudo gpg -o /path` |
| New function returns 404 after successful deploy | Function file not imported in `api/src/index.ts` — routes are not auto-discovered | Add `import './functions/{name}'` to `api/src/index.ts`; the file compiling and deploying is not sufficient |

---

## Full Documentation

- `CLAUDE.md` — Operational guide for this specific project
- `ARCHITECTURE.md` — Full design document with diagrams
- `DEPLOY.md` — Step-by-step guide for deploying a renamed copy
- `FC1-DEPLOYMENT.md` — Flex Consumption (FC1) Function App deployment guide with pitfalls and verified fixes (use when standalone functions with timer triggers, queues, or AI workloads are needed beyond SWA managed functions)
- `README.md` — Setup instructions and Claude Code starter prompt
