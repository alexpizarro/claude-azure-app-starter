# CLAUDE.md — Claude Azure App Starter

This file gives Claude Code (or any AI assistant) the project context needed to work effectively on this codebase. Read it before making changes.

> **Before making any commits:** Always run `git branch --show-current` to confirm which branch is active. All development work and commits should target `main`. Never commit directly to `test` or `production` — those branches are deployment triggers and should only receive changes via `git merge main`. Committing to the wrong branch can trigger unintended deployments.

---

## What this project is

A **hello-world starter template** for Azure projects. The goal is a single git push from any device (including iOS) deploys everything to Azure automatically.

**Live deployments:** *(set after your first deployment)*
- Test: `https://<your-swa-name>.azurestaticapps.net`
- Production: `https://<your-swa-name>.azurestaticapps.net`

**Stack:** React 19 + TypeScript (frontend) | Azure Functions v4 + Node.js 22 (API) | Azure SQL Serverless (database) | Bicep (IaC) | GitHub Actions (CI/CD)

---

## Claude Code operating mode

Users of projects built from this template are often **not Azure experts**. When a task requires Azure CLI (`az`) or GitHub CLI (`gh`) commands:

- **Run them directly** using the Bash tool — do not paste commands and ask the user to run them manually
- Only ask the user for values Claude cannot determine from context: subscription IDs, passwords, client IDs, repo names, org names
- If a value is not yet known, retrieve it first with a query command (`az account show --query id -o tsv`, `az ad sp show --id ... --query id -o tsv`) before asking the user
- This applies to: creating service principals, granting role assignments, creating federated credentials, setting GitHub secrets, and querying Azure resource state

---

## Naming convention

**Formula:** `{org}-{project}-{component}-{env}` (hyphens) | `{org}{project}{component}{env}` (no hyphens for storage/KV)

| Token | Value |
|---|---|
| `{org}` | your org abbreviation (e.g. `myorg`) |
| `{project}` | your project name (e.g. `myapp`) |
| `{env}` | `test` or `prod` |

| Resource | Test name | Prod name |
|---|---|---|
| Resource Group | `{org}-{project}-rg-test` | `{org}-{project}-rg-prod` |
| Static Web App | `{org}-{project}-swa-test` | `{org}-{project}-swa-prod` |
| SQL Server | `{org}-{project}-sql-test` | `{org}-{project}-sql-prod` |
| SQL Database | `{org}-{project}-sqldb-test` | `{org}-{project}-sqldb-prod` |
| GitHub SPs | `{org}-{project}-github-test` | `{org}-{project}-github-prod` |

**Single source of truth for naming:** `infra/main.bicep` lines `var org` and `var project`. All Azure resource names are computed from these two variables — changing them here changes everything.

---

## Branch → deploy mapping

| Branch | Deploys to | Triggered by |
|---|---|---|
| `main` | Nothing — local dev only | — |
| `test` | `{org}-{project}-rg-test` | Push to `test` branch |
| `production` | `{org}-{project}-rg-prod` | Push to `production` branch |

**To deploy:** merge `main` into the target branch and push:
```bash
git checkout test && git merge main && git push origin test
git checkout production && git merge main && git push origin production
git checkout main
```

---

## Architecture decisions

### SWA managed functions (not standalone Function App)
The API lives in `api/` and is deployed by `Azure/static-web-apps-deploy@v1` as a SWA managed function. This means:
- **Single deploy action** handles frontend + API together
- **Free tier** — no separate Function App plan needed
- **Limitation:** HTTP triggers only (no timer triggers, no queue triggers)
- The `infra/modules/functionApp.bicep` module exists for future phases requiring FC1 (Flex Consumption) standalone functions with timer triggers or AI workloads

### SQL — serverless tier, password auth
- SKU: `GP_S_Gen5_1` (General Purpose Serverless, Gen5, 1 vCore)
- **Auto-pauses after 60 minutes** of inactivity — first request after pause takes 30–60 seconds
- **minCapacity: 0.5** — scales down to 0.5 vCores when idle
- Password auth (`sqladmin` user) — Key Vault integration is a future phase
- Connection string is set as a SWA app setting via Bicep (never logged or output)

### SWA location is NOT `australiaeast`
Azure Static Web Apps do not support `australiaeast` as a billing region. The SWA is deployed to `eastasia` (closest supported region). All other resources use `australiaeast`. This is hardcoded in `infra/main.bicep` as `var swaLocation = 'eastasia'`.

### OIDC auth — no secrets to rotate
Two service principals (`{org}-{project}-github-test`, `{org}-{project}-github-prod`) authenticate via OIDC federated credentials. No client secrets. Each SP is federated to its specific branch (`test` and `production` respectively) — the test SP cannot access production.

### Bicep params — JSON format required
The parameter files are `.parameters.json` (not `.bicepparam`). This is intentional: JSON params files can be combined with inline `--parameters key=value` arguments (used to inject the SQL password from GitHub secrets at deploy time). `.bicepparam` files cannot be supplemented this way and cause a BCP258 error.

---

## Project structure

```
.
├── .github/workflows/
│   ├── deploy-test.yml       # Triggers on push to 'test' branch
│   └── deploy-prod.yml       # Triggers on push to 'production' branch
├── infra/
│   ├── main.bicep            # Root — subscription-scoped. ALL resource names defined here.
│   ├── modules/
│   │   ├── resourceGroup.bicep
│   │   ├── staticWebApp.bicep    # SWA + app settings (SQL_CONNECTION_STRING)
│   │   ├── sqlServer.bicep       # SQL Server + serverless DB + firewall rule
│   │   └── functionApp.bicep     # FC1 standalone Function App (future phase)
│   ├── environments/
│   │   ├── test.parameters.json  # Non-secret params for test env
│   │   └── prod.parameters.json  # Non-secret params for prod env
│   └── sql/migrations/           # Versioned SQL migration scripts
│       ├── 000_migration_history.sql   # Always first — creates tracking table
│       └── 001_create_items_table.sql  # Items table
├── frontend/                 # React + TypeScript + Vite 6
│   ├── src/
│   │   ├── App.tsx
│   │   └── services/api.ts
│   └── public/staticwebapp.config.json   # SWA routing + security headers
├── api/                      # Azure Functions v4 — Node.js 22 + TypeScript
│   ├── src/
│   │   ├── index.ts              # Entry point — imports all functions
│   │   ├── functions/
│   │   │   ├── hello.ts          # GET /api/hello — health check
│   │   │   ├── getItems.ts       # GET /api/items
│   │   │   └── createItem.ts     # POST /api/items
│   │   └── lib/
│   │       └── database.ts       # mssql connection pool
│   ├── host.json
│   ├── tsconfig.json             # CommonJS module output (required for SWA managed functions)
│   └── local.settings.json.example
├── ARCHITECTURE.md           # Full design document with diagrams
├── DEPLOY.md                 # Step-by-step guide for deploying a renamed copy
└── CLAUDE.md                 # This file
```

---

## SQL migration system

### How it works
`azure/sql-action@v2.3` runs all `*.sql` files in `infra/sql/migrations/` on **every deployment**, in alphabetical order. The action automatically adds a firewall rule for the GitHub Actions runner IP, runs the scripts, then removes it.

### Migration tracking
`000_migration_history.sql` creates `dbo.__MigrationHistory`. Every migration from `001` onward should check this table before executing:

```sql
IF NOT EXISTS (
    SELECT 1 FROM dbo.__MigrationHistory WHERE MigrationId = '00N_migration_name'
)
BEGIN
    -- Your DDL here
    INSERT INTO dbo.__MigrationHistory (MigrationId) VALUES ('00N_migration_name');
END
```

### Rules for writing migrations

1. **File naming:** `{NNN}_{description}.sql` — zero-padded number + snake_case description (e.g. `002_add_users_table.sql`)
2. **Never edit a file that has already been applied to production** — add a new migration instead
3. **Use `__MigrationHistory` guard for new migrations** — `001` uses a direct table existence check because the Items table was already deployed before the tracking table existed; all migrations from `002` onward use the tracking guard
4. **Seed data must also be idempotent** — use `IF NOT EXISTS` or `MERGE` for inserts
5. **No rollback scripts** — roll forward only. For urgent fixes, point-in-time restore is available (7-day retention on serverless tier)

### Adding a new migration
```sql
-- NNN_describe_change.sql
IF NOT EXISTS (
    SELECT 1 FROM dbo.__MigrationHistory WHERE MigrationId = 'NNN_describe_change'
)
BEGIN
    -- DDL here (ALTER TABLE, CREATE TABLE, etc.)

    INSERT INTO dbo.__MigrationHistory (MigrationId) VALUES ('NNN_describe_change');
    PRINT 'Migration NNN_describe_change applied.';
END
ELSE
BEGIN
    PRINT 'Migration NNN_describe_change already applied — skipping.';
END
```

---

## GitHub Actions workflow anatomy

Each workflow (`deploy-test.yml`, `deploy-prod.yml`) runs a single job with three sequential steps:

**Step 1 — Deploy Infrastructure (Bicep)**
```bash
az deployment sub create \
  --name "deploy-{env}-{run_id}" \
  --location australiaeast \
  --template-file infra/main.bicep \
  --parameters @infra/environments/{env}.parameters.json \
  --parameters sqlAdminPassword="$SQL_PASSWORD"
```
Captures `swaToken`, `sqlFqdn`, `sqlDb` from Bicep outputs. Masks `swaToken` immediately with `::add-mask::`.

**Step 2 — Run SQL Migrations**
`azure/sql-action@v2.3` — auto-manages firewall rules for the runner IP. Runs all `*.sql` files in `infra/sql/migrations/`.

**Step 3 — Deploy to Azure Static Web Apps**
`Azure/static-web-apps-deploy@v1` — Oryx builds the TypeScript Functions and React app, then deploys both. Uses the masked `swaToken` from Step 1.

---

## GitHub secrets (6 required)

| Secret | Description |
|---|---|
| `AZURE_TENANT_ID` | Shared across both envs |
| `AZURE_SUBSCRIPTION_ID` | Shared across both envs |
| `AZURE_CLIENT_ID_TEST` | `appId` of `{org}-{project}-github-test` SP |
| `AZURE_CLIENT_ID_PROD` | `appId` of `{org}-{project}-github-prod` SP |
| `SQL_ADMIN_PASSWORD_TEST` | SQL admin password for test (never committed) |
| `SQL_ADMIN_PASSWORD_PROD` | SQL admin password for prod (never committed) |

---

## API conventions

- All functions use `authLevel: 'anonymous'`
- HTTP methods and routes declared in `app.http(...)` call at the bottom of each file
- `api/src/index.ts` imports all function files as side effects — **every new function file must be added here or its route will return 404**. The file compiling and deploying successfully is not sufficient; the import is what registers the route with the runtime.
- The `database.ts` module maintains a module-level connection pool (`sql.connect()` called once)
- TypeScript compiles to CommonJS (`"module": "commonjs"` in `tsconfig.json`) — required for SWA managed functions
- `"main": "dist/index.js"` in `package.json` — must be a specific file path, not a glob
- When adding an npm package that does not bundle its own `.d.ts` files, add `@types/{package}` to `api/package.json` devDependencies (e.g. `@types/mssql`)

---

## Local development

Local API connects directly to test environment Azure SQL (not a local emulator).

```bash
cp api/local.settings.json.example api/local.settings.json
# Fill in SQL_CONNECTION_STRING with the test environment password

# Terminal 1
cd api && npm start          # Functions on port 7071

# Terminal 2
cd frontend && npm run dev   # React on port 5173, proxies /api/* → 7071
```

`az login` is required before starting locally (DefaultAzureCredential not currently used, but good habit).

Leave unconfigured service keys as empty strings in `local.settings.json`. Functions should return mock responses when keys are absent — this allows UI testing before Azure is provisioned. Never use non-empty placeholder strings (e.g. `"sk-YOUR_KEY"`) in example files — they fool JavaScript truthiness checks and produce confusing runtime errors.

---

## Known deployment gotchas

| Problem | Cause | Fix |
|---|---|---|
| `BCP258: sqlAdminPassword missing` | Using `.bicepparam` instead of `.parameters.json` | Keep parameter files as `.parameters.json`; use `@` prefix in `--parameters @file.json` |
| `LocationNotAvailableForResourceType` for SWA | `australiaeast` not supported for `Microsoft.Web/staticSites` | `swaLocation` is hardcoded to `eastasia` in `main.bicep` — do not change to match other resources |
| `listSecrets` output warning | Bicep linter flags secrets in outputs | Suppressed with `#disable-next-line outputs-should-not-contain-secrets` — token must be output for GitHub Actions |
| Functions return 500 on first request after idle | SQL serverless auto-paused | Wait 30–60 seconds and retry — database is resuming |
| OIDC login fails with `AADSTS70021` | Federated credential subject mismatch | Subject must be `repo:{GithubOrg}/{Repo}:ref:refs/heads/{branch}` exactly |
| Bicep `roleAssignments` step fails with 403 | OIDC SP only has `Contributor` — FC1 Managed Identity RBAC requires `Microsoft.Authorization/roleAssignments/write` | Grant `User Access Administrator` at RG scope: `az role assignment create --assignee <SP_OID> --role "User Access Administrator" --scope /subscriptions/{sub}/resourceGroups/{rg}` (both test and prod SPs) |
| New API function returns 404 after deploy | Function file not imported in `api/src/index.ts` — routes are not auto-discovered | Add `import './functions/{name}'` to `api/src/index.ts` |

---

## Guiding principles

1. **Simple git push > everything else** — if a change makes deployment harder, reconsider it
2. **Free tier > Consumption > fixed cost** — use the cheapest option that meets the need
3. **Two environments, identical infrastructure** — test and prod use the same Bicep, different parameter files
4. **Secrets never in code** — SQL password injected at deploy time from GitHub secrets; never in param files or outputs
5. **Idempotent everything** — Bicep deployments, SQL migrations, and seed data must all be safe to run multiple times
