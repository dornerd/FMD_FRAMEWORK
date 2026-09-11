# CLAUDE.md — FMD Framework (Fabric Metadata-Driven) Agent Operating Guide

Instructions for an AI agent operating the **Fabric Metadata-Driven Framework (FMD)**
deployed in this Fabric tenant, via the Fabric CLI (`fab`), direct SQL, and the Fabric REST API.

Originally forked from https://github.com/edkreuk/FMD_FRAMEWORK (MIT) as a starting point.
We are **not** tracking upstream or contributing back — this is now our own codebase built
on top of that base (see rule 8 in §2 on git remotes). The actual deployment here was done
manually (see §1), not via the repo's own setup notebook.

**Read [../CLAUDE.md](../CLAUDE.md) first** — it has the Windows/PowerShell environment
notes and the Azure PAYG cost guardrail that apply to everything below.

---

## 1. What this system is, and how it got deployed here

FMD is a metadata-driven ingestion and orchestration framework for Microsoft Fabric.

- **Control plane:** Fabric SQL Database `SQL_INTEGRATION_FRAMEWORK`, schemas
  `integration` (source/entity/connection config), `execution` (per-run queue
  tables — what's landed, what's processed), `logging` (audit trail).
- **Execution plane:** Fabric Data Pipelines + parameterized Spark notebooks,
  lakehouse-first, medallion (Bronze/Silver — Gold not deployed here yet).
- **Orchestration:** `PL_FMD_LOAD_ALL` is the real entry point (Landingzone → Bronze →
  Silver). The Fabric **Taskflow** item is cosmetic — a visual canvas, not executable.
- **Deployment:** done directly via `fab` CLI + Fabric REST API + direct SQL from
  Claude Code, replicating what `setup/NB_SETUP_FMD.ipynb` would have done — that
  notebook was never actually run. See `../CLAUDE.md`'s "FMD_FRAMEWORK deployment"
  section for the full bug list this surfaced; several are load-bearing for §7 below.

**The core implication for you:** onboarding a new source is a *metadata* change —
rows in `integration.LandingzoneEntity`/`BronzeLayerEntity`/`SilverLayerEntity`, plus a
`Connection`/`DataSource` if the source type is new. You do not hand-author pipeline
JSON or notebook logic unless a bug forces it (§7 has the confirmed bug list; check
there before assuming new pipeline code is needed).

### Current deployment (verify before trusting — see §2 rule 1)

| Item | Value |
|---|---|
| Domain | `INTEGRATION` |
| Workspace `INTEGRATION CODE (D)` | `a429833b-6390-466d-ac86-f4e3e181eb21` |
| Workspace `INTEGRATION DATA (D)` | `48daabd7-7e4a-4cf2-a5bb-b431fef8d533` |
| Workspace `INTEGRATION CONFIG` | `0d449e17-4ebc-49a1-9cb6-bcf67f5e9644` |
| SQL DB `SQL_INTEGRATION_FRAMEWORK` | item id `28197400-a126-4966-be2f-cd274424ba86` |
| SQL server FQDN | `icb7yrp2pncehdkvzfe3x5hoze-c6peidn4j2quthfwxt3h6xuwiq.database.fabric.microsoft.com` |
| Capacity | Trial (`FTL4`) — see §7 for its Spark concurrency limit |
| Service principal | `SP-FMD-FABRIC-PIPELINES`, secret in Key Vault `kv-fmd-framework` |

These GUIDs will go stale the moment anyone redeploys or adds an environment — re-derive
via `fab get`/SQL rather than trusting this table blindly if anything doesn't line up.

---

## 2. Non-negotiable rules

1. **Verify state before writing, every session.** GUIDs and row IDs above and in your
   own memory can go stale. Re-check with `fab get`/`fab ls` or a `SELECT` before using
   a cached value in a write.
2. **Prefer the tooling pipeline over hand-written SQL** where one exists — see §5.1.
   `PL_TOOLING_POST_ASQL_TO_FMD` is deployed but untested end-to-end here; try it, don't
   assume it works from the name alone.
3. **Never compose raw `INSERT`/`UPDATE` text against `integration.*` tables.** Always
   go through the matching `sp_Upsert*` stored procedure (§5.2) — they're idempotent by
   natural key, hand-written SQL isn't.
4. **This deployment has one environment: dev.** There is no test/prod allowlist to
   check yet. If someone asks you to add one, that's new deployment work, not a config
   change — treat it accordingly (new workspaces, new capacity decision, likely a
   second `environments` block).
5. **Connection creation is possible and documented (§8) — but treat it as a security
   action, not a routine one.** Creating a connection embeds a real credential (SP
   secret or storage key) into the tenant. Confirm with the user before creating a new
   one, same bar as any other credential-handling action.
6. **Retry budget: 2.** Then stop and report with the actual error text. Fabric's
   pipeline-level errors are frequently misleading (§7) — a third blind retry burns
   Trial-capacity Spark quota for no benefit. Escalate to reading `logging.*` (§6)
   instead of retrying again.
7. **Log what you ran.** Every `fab`/SQL write command you execute should be visible in
   your own tool-call transcript (it already is, by construction) — don't summarize a
   write away in prose without showing the actual command.
8. **Never push to `origin` or `myfork`.** `origin` is the third-party upstream
   (`edkreuk/FMD_FRAMEWORK`) — we do not push there, ever. `myfork`
   (`dornerd/FMD_FRAMEWORK`) was the original fork remote, but we're no longer tracking
   upstream or contributing back to it — we're building our own thing on top of this
   codebase. Local commits on `main` are fine; if/when this needs its own remote home,
   that's a deliberate decision to make with the user (new repo, likely in Azure DevOps
   alongside `Fabric_Agent`), not a default `git push`.

---

## 3. Prerequisites (verify, don't assume)

- `fab` CLI — already installed and authenticated as the interactive user
  (`gustavo@balutechn.onmicrosoft.com`) for manual/session-driven work. Check with
  `fab auth status`. This is a **user identity**, not a service principal — fine for
  Claude-Code-in-the-loop work, not for a truly unattended scheduled agent.
- For direct SQL writes: Node `mssql` package (`npm install mssql --no-save` in this
  directory if `node_modules/mssql` isn't already present) — see §6 for why the
  `fabric-sql` MCP isn't sufficient on its own.
- `az` CLI logged into the `balutechn` tenant — used to mint the AAD access token
  (`--resource https://api.fabric.microsoft.com`) that both `fab` and direct SQL need.
- No commit-pinning concern here — this is a full local clone, not a live `git pull`
  dependency. If re-cloning upstream, note it has no tagged releases; pin a commit.

---

## 4. Tool tiers

### 4.1 Read-only — always allowed, no confirmation

| Command | Use |
|---|---|
| `fab ls <path>` | Inventory workspaces/items |
| `fab get <item> -f -q <jmespath>` | Read item properties (id, server FQDN, etc.) |
| `fab exists <path>` | Idempotency check before any create/import |
| `SELECT` via direct `mssql` connection (§6) | Config, queue, and `logging.*` reads |

Start every task here. Build a picture of actual state before proposing anything.

### 4.2 Write — allowed against an approved plan, not silently

| Command | Notes |
|---|---|
| `fab import <path> -i <local dir> -f` | Redeploying a pipeline/notebook. **Check §7 bug 8 first** if the target is one of the 10 pipelines that deployed clean on the very first pass — their local files may be stale placeholder-GUID copies. |
| `fab create <path> -P <params>` | Always precede with `fab exists`. |
| Pipeline trigger via REST (`POST .../jobs/instances?jobType=Pipeline`) | Then poll — see §6. |
| `EXEC integration.sp_Upsert*` / `EXEC execution.sp_Upsert*` (§5.2) | Scoped to these schemas only. No ad-hoc DDL. |

### 4.3 Forbidden — never do these, redirect to the user instead

- `fab del`/`fab rm` on anything you didn't create in the current task, without
  explicit confirmation first.
- Capacity assignment, reassignment, or creation (billable — Azure PAYG guardrail in
  `../CLAUDE.md`).
- Any Azure RBAC / IAM role assignment (`az role assignment create` and similar) — the
  Claude Code classifier already blocks this by design; don't work around it, ask the
  user to run the command themselves (this happened during initial deployment — it's
  expected behavior, not a bug).
- Tenant admin portal settings.
- DDL against `SQL_INTEGRATION_FRAMEWORK` outside the already-deployed
  `integration`/`execution`/`logging` schemas.
- Writing through the `fabric-sql` MCP server — it's bound to the read-only SQL
  analytics endpoint and will reject DML outright (§6).

---

## 5. The config contract

### 5.1 Preferred path: bulk onboarding

For Azure SQL sources, the framework ships `PL_TOOLING_POST_ASQL_TO_FMD`, intended to
extract table metadata from the source and populate the config tables in bulk. It's
deployed (`fab exists "INTEGRATION CODE (D).Workspace/PL_TOOLING_POST_ASQL_TO_FMD.DataPipeline"`)
but has not been run end-to-end in this deployment — the two source types actually
proven here (§7 context) were registered by hand via §5.2. Try the tooling pipeline
first for a genuine bulk ASQL onboarding job; verify what it actually produces in
`integration.*` before trusting it, and update this note with the result.

### 5.2 Fallback / current path: direct stored-procedure upsert

This is the path actually used and verified in this deployment. One call per layer,
in order — each returns the row's own ID via a follow-up `SELECT` (not `OUTPUT`, see
`../CLAUDE.md` bug 1):

```sql
-- 1. Connection (once per distinct external system/credential)
EXEC integration.sp_UpsertConnection
    @ConnectionGuid = '<fabric connection id>',  -- from fab/REST connection creation
    @Name = 'CON_...', @Type = '<ADLS|ONELAKE|SQL|SFTP|FTP|ORACLE|AZURESQLMI|ADF|NOTEBOOK>',
    @IsActive = 1;
-- @Type drives routing in PL_FMD_LOAD_LANDINGZONE's switch — must match one of these
-- literal values (case-insensitive), or the entity silently falls into the default
-- "unknown data source" Fail branch.

-- 2. Data source (the container/filesystem/namespace within that connection)
EXEC integration.sp_UpsertDataSource
    @ConnectionId = <internal int, look up by ConnectionGuid>, @DataSourceId = 0,
    @Name = '<container/filesystem/db name>', @Namespace = '<short tag>',
    @Type = '<TYPE>_01',  -- e.g. 'ADLS_01', 'ONELAKE_TABLES_01' — must match what the
                           -- COMMAND_* pipeline's own Lookup filters on
    @Description = '...', @IsActive = 1;

-- 3. Landingzone entity (one row per source table/file)
EXEC integration.sp_UpsertLandingzoneEntity
    @LandingzoneEntityId = 0, @DataSourceId = <from step 2>,
    @LakehouseId = <internal id of LH_DATA_LANDINGZONE>,
    @SourceSchema = '...', @SourceName = '...', @SourceCustomSelect = '',
    @FileName = '...', @FilePath = '...', @FileType = 'csv|parquet|...',
    @IsIncremental = 0, @IsIncrementalColumn = '', @CustomNotebookName = '', @IsActive = 1;

-- 4. Bronze entity
EXEC integration.sp_UpsertBronzeLayerEntity
    @BronzeLayerEntityId = 0, @LandingzoneEntityId = <from step 3>,
    @Schema = '...', @Name = '...', @FileType = 'Delta',
    @LakehouseId = <internal id of LH_BRONZE_LAYER>, @PrimaryKeys = '...', @IsActive = 1;

-- 5. Silver entity
EXEC integration.sp_UpsertSilverLayerEntity
    @SilverLayerEntityId = 0, @BronzeLayerEntityId = <from step 4>,
    @LakehouseId = <internal id of LH_SILVER_LAYER>,
    @Name = '...', @Schema = '...', @FileType = 'delta', @IsActive = 1;
```

Internal lakehouse ids: look up once per session with
`SELECT LakehouseId, Name FROM integration.Lakehouse` — don't hardcode, they're
tenant/deployment-specific.

**Default a new entity active (`@IsActive = 1`) only after you've confirmed the
source data actually exists at the path/table you registered.** An entity registered
against a nonexistent source doesn't error — it just fails at Bronze copy time and
you'll be back in §6/§7 debugging something that was never real to begin with.

---

## 6. Observability loop

`logging.*` is what makes this a closed loop instead of fire-and-forget:

| Table | Contains |
|---|---|
| `logging.PipelineExecution` | Start/end/fail per pipeline run, by `PipelineName` + `EntityLayer` |
| `logging.CopyActivityExecution` | Per-copy-activity detail, including real byte/row counts and the underlying connector's own error |
| `logging.NotebookExecution` | Declared but not actually called anywhere in the deployed pipelines — don't rely on it; notebook failures show up in `PipelineExecution` instead |

Queue/state tables worth checking directly:
- `execution.PipelineLandingzoneEntity` / `execution.PipelineBronzeLayerEntity` —
  `IsProcessed` flag gates whether the next layer picks the row up. `../CLAUDE.md`
  bug 5 (Bronze's flag always `False`, causing Silver to reprocess every entity every
  run) is fixed as of the `NB_FMD_LOAD_LANDING_BRONZE` update — verify with
  `SELECT * FROM execution.vw_LoadToSilverLayer` returning 0 rows right after a clean run.

**Use a direct `mssql` connection for all of this, not the `fabric-sql` MCP.** The MCP
is bound to the read-only SQL analytics endpoint — it can lag several minutes behind
real writes and will flatly reject any DML. Pattern that works (no ODBC driver, no
admin rights needed):

```js
const sql = require('mssql');
const pool = await sql.connect({
  server: '<SQL server FQDN from §1>', port: 1433,
  database: '<database name from `fab get ... -q properties.databaseName`>',
  authentication: { type: 'azure-active-directory-access-token',
                     options: { token: /* az account get-access-token --resource https://api.fabric.microsoft.com */ } },
  options: { encrypt: true }, connectionTimeout: 30000
});
```

Standard loop:

```
plan → verify state (§4.1) → trigger pipeline via REST →
  poll job instance status until terminal (NotStarted/InProgress → Completed/Failed) →
    Completed → query logging.PipelineExecution + the target lakehouse's Files/Tables
                 to confirm real data moved, not just a clean status code
    Failed    → read logging.PipelineExecution's LogData for that PipelineName/EntityLayer;
                if still vague ("inner activity failed"), trigger the FAILING SUB-PIPELINE
                directly (not the parent) to get its own, more specific failureReason —
                this is how every bug in §7 was actually diagnosed
```

**Never report success from a clean pipeline status alone.** Bug 6 in `../CLAUDE.md`
is a pipeline that reports `Completed` while silently losing the entity's data —
confirmed only by checking the actual lakehouse `Files`/`Tables` path.

---

## 7. Failure playbook

| Symptom | First check |
|---|---|
| Pipeline fails in under ~15s, before any real Spark/copy work could have happened | Almost certainly a config/binding error, not a data error — check the exact pipeline JSON for stale placeholder GUIDs or the `workspaceId` sentinel (`../CLAUDE.md` bugs 4, 8) before assuming the source data is the problem |
| `fab import` fails with `RequestValidationFailed: User does not have access to the connection used in the Pipeline` | Not a permissions problem — stale local file with placeholder GUIDs. Reapply `node ../fmd_replace_ids.js <PipelineName>` (see `../CLAUDE.md` bug 8) before reimporting |
| Notebook activity fails with `Failed to get workspace details` / `errorCode 2461` | `../CLAUDE.md` bug 3 — pre-create the notebook it's trying to auto-generate |
| Pipeline reports `Completed`, but the target lakehouse path is empty | `../CLAUDE.md` bug 6 (hardcoded sink `container` field) or bug 2 (demo seeder Delta-vs-file mismatch) — check which source type it is |
| `Data Manipulation Language (DML) statements are not supported for this table type` | You're on the `fabric-sql` MCP's read-only endpoint — switch to direct `mssql` (§6) |
| `System cancelled the Spark session due to statement execution failures` with no further detail | Trigger the notebook directly (not via the pipeline) with the same params to get the real Spark exception in its own job status |
| `[TooManyRequestsForCapacity]` / HTTP 430 | Trial capacity's Spark concurrency limit — wait ~60-90s between pipeline runs that both use Spark, don't retry immediately |
| Item already exists on `fab create` | You skipped `fab exists`. Stop, re-inventory. |

---

## 8. Connection creation — actually works, here's the verified recipe

Contrary to generic community reports of `fab create .connections` being unreliable
for credentialed connection types, **creating connections via the Fabric REST API
directly works reliably** and is how every connection in this deployment was made.
`fab create .connections/...` only works for the no-credential `WorkspaceIdentity`
types (`FabricDataPipelines`, `Notebook`); anything needing a real credential
(`FabricSql`, `AzureDataLakeStorage`, etc.) needs `POST /v1/connections` directly:

```js
// FabricSql via service principal — see credentialType options per type via
// GET /v1/connections/supportedConnectionTypes?showAllCreationMethods=true
{
  "connectivityType": "ShareableCloud",
  "displayName": "CON_...",
  "connectionDetails": { "type": "FabricSql", "creationMethod": "FabricSql.Contents", "parameters": [] },
  "privacyLevel": "Organizational",
  "credentialDetails": {
    "singleSignOnType": "None", "connectionEncryption": "NotEncrypted", "skipTestConnection": false,
    "credentials": { "credentialType": "ServicePrincipal", "tenantId": "...", "servicePrincipalClientId": "...", "servicePrincipalSecret": "..." }
  }
}
```

```js
// AzureDataLakeStorage via account key
{
  "connectivityType": "ShareableCloud", "displayName": "CON_...",
  "connectionDetails": { "type": "AzureDataLakeStorage", "creationMethod": "AzureDataLakeStorage",
    "parameters": [{"dataType":"Text","name":"server","value":"https://<account>.dfs.core.windows.net"},
                    {"dataType":"Text","name":"path","value":"<container>"}] },
  "privacyLevel": "Organizational",
  "credentialDetails": { "singleSignOnType": "None", "connectionEncryption": "NotEncrypted", "skipTestConnection": false,
    "credentials": { "credentialType": "Key", "key": "<storage account key>" } }
}
```

After creating, grant every OTHER principal that needs to use it (a service principal,
a workspace's managed identity) a role explicitly — the creator already gets one
automatically (confirmed: re-granting yourself returns `409
ConnectionRoleAssignmentAlreadyExists`), but nobody else does:
`POST /v1/connections/{id}/roleAssignments` with
`{"role":"User","principal":{"id":"<sp or identity object id>","type":"ServicePrincipal|User"}}`.

If `fab import` on a pipeline referencing this connection fails with `User does not
have access to the connection used in the Pipeline`, don't assume it's a role-assignment
problem first — check `../CLAUDE.md` bug 8 (stale placeholder GUIDs in the local file)
before touching roles. That misleading error is what stale content produces, not a
missing grant.

Still gate actually *creating* a new connection behind user confirmation per rule 5 —
this section is "here's how," not "do this unprompted."

---

## 9. Output expectations

- Report discrepancies between intended and actual state explicitly — a pipeline that
  "succeeds" with no data moved is a discrepancy, not a success (§6).
- Lead with the result. No preamble.
- Before any write in tier 4.2, state what you're about to run and why — the config
  diff (rows before/after) or the exact command — before executing it.
- If a request is ambiguous about which environment/entity it targets, ask rather than
  guessing — there's no test/prod allowlist here to catch a wrong guess (rule 4).
