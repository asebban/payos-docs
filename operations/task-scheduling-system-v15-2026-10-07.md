Created: 2026-10-02
Last updated: 2026-10-07
Version: 15

# Task Scheduling System (`payos-job-scheduler`)

Operator/developer guidance for running TSS: a Quartz-backed, clustered scheduler that registers and fires tenant JavaScript job scripts on a cron schedule, reading PayOS's own tenant/platform database configuration rather than duplicating it.

Like `payos-service-notification`, this is its **own deployable artifact** with its own entry point — it does not implement `IServer`/`ServerProvider` and is not booted from a bundle's `service-adapters-dir`. Unlike the notification daemon, it can also run **embedded inside `payos-runtime`** instead of as a separate process — see [Deployment modes](#deployment-modes) below. For the class-level design (every class, how they relate, the full execution-flow walkthrough), see the living [class diagram](../architecture/job-scheduler-class-diagram-v18-2026-10-07.md) and [ADR-0009](../architecture/adr/0009-task-scheduling-system.md); this page is the operational "how do I configure and run it" companion.

## Deployment modes

| | Standalone (`StandaloneMain`) | Embedded (`RuntimeIntegrationHook`) |
| --- | --- | --- |
| Entry point | `ma.s2m.payos.scheduler.bootstrap.StandaloneMain.main(...)` | `RuntimeIntegrationHook.start(...)`, called by `payos-runtime` during its own startup |
| Process | Its own JVM, independently scalable from `payos-runtime`'s replica count | Shares `payos-runtime`'s JVM and lifecycle |
| `jobs/` and script files resolve against | `--scheduler-base-dir <path>` (defaults to the process's working directory) | `PayOSConfig.getRuntimeBaseDirectory()` — the host's own `--bundle-path` |
| Tenant/platform database settings come from | A PayOS config file this module loads itself (see [Tenant and platform database resolution](#tenant-and-platform-database-resolution)) | The host's already-loaded `PayOSConfig.settings`/`PayOSConfig.getPlatformDatabaseService()` — no file is re-read |
| A job-registration failure (bad tenant config, etc.) | **Propagates** — fails the whole startup. TSS is this process's only job, so fail-fast is correct. | **Logged, not propagated** — TSS is a secondary subsystem inside `payos-runtime` and must not take the whole runtime down over a problem scoped to TSS's own jobs |
| TSS's own Quartz schema DB (`tss-config.json`) | Resolved via `StandaloneConfigLoader`'s three-tier model (below) | Same — this part doesn't change between modes |

Both modes delegate all scheduler construction and job registration to the same `SchedulerBootstrap`/`TenantDatabaseRegistryLoader` code, so the two deployment modes cannot drift apart in behavior.

## Building

```bash
cd payos-job-scheduler
mvn clean package
```

This produces `target/payos-job-scheduler-<version>.jar`: a self-contained, runnable uber-jar (`maven-shade-plugin`, same pattern as `payos-service-notification`), `Main-Class: ma.s2m.payos.scheduler.bootstrap.StandaloneMain`, every dependency bundled — GraalVM polyglot/JS, Hibernate, Quartz, HikariCP, the H2/PostgreSQL JDBC drivers, and `dynamic-database-service`'s `IDatabaseServiceFactory` SPI provider. No separate classpath assembly is needed to run it.

## Starting the service (standalone)

```bash
java -jar payos-job-scheduler-<version>.jar --scheduler-base-dir /opt/payos-tss
```

`--scheduler-base-dir` is optional; omitted, it defaults to the process's current working directory. Everything else TSS reads (`tss-config.json`, `jobs/`, job scripts, the PayOS config file) is resolved relative to this one directory unless overridden individually (see below).

### Stopping

`StandaloneMain` registers a JVM shutdown hook, so a normal `SIGTERM` triggers `SchedulerBootstrap.shutdown(true)` — it waits for any in-flight job executions to finish before the process exits. Avoid `kill -9`/`SIGKILL` where possible — it skips the shutdown hook entirely and can leave a job mid-execution.

## Configuration

### TSS's own database (`tss-config.json`)

This is **TSS's own isolated schema** — the Quartz `QRTZ_*` tables and `tss_job_execution` — never a tenant or platform database. Resolved through `StandaloneConfigLoader`'s three-tier model, in priority order: environment variable → system property of the same name → a field in `tss-config.json`, nested under the matching block below. `TSS_CONFIG_FILE` (env var/system property) locates the JSON file itself; it defaults to `tss-config.json` in the scheduler base directory.

| Env var / property | JSON field (under `"database"`) | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_DB_URL` | `dbUrl` | *(required)* | JDBC URL for TSS's own schema. |
| `TSS_DB_USER` | `dbUser` | *(none)* | Database username. |
| `TSS_DB_PASSWORD` | `dbPassword` | *(none)* | Database password. |
| `TSS_DB_DRIVER` | `dbDriver` | *(required)* | JDBC driver class. |
| `TSS_DB_VENDOR` | `dbVendor` | *(generic ANSI SQL)* | `postgresql` or `oracle` — selects Quartz's `DriverDelegate` (`DriverDelegateResolver`). Any other value (including PostgreSQL) resolves to Quartz's generic `StdJDBCDelegate`. |
| `TSS_DB_SCHEMA` | `dbSchema` | *(none)* | Schema name, if your deployment uses a non-default one. |
| `TSS_DB_MAX_POOL_SIZE` | `dbMaxPoolSize` | `5` | Connection pool size for TSS's own `DataSource`. |

### Scheduler identity and sizing

Same three-tier resolution, under the `"scheduler"` block.

| Env var / property | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_SCHEDULER_INSTANCE_NAME` | `schedulerInstanceName` | `TSS-Cluster` | Quartz scheduler instance name — every clustered node sharing one `jobs/` registry must use the **same** name. |
| `TSS_THREAD_COUNT` | `threadCount` | `5` | Quartz worker thread pool size. |
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `maxConcurrentExecutionsPerTenant` | `0` (disabled) | Caps how many of one tenant's executions may be in progress at once, cluster-wide — the simplest possible "quota rejection" enforcement (FR21). This is the **global default**, applied to every tenant that doesn't declare its own `maxConcurrentExecutions` in its `jobs.json` (see below) — a tenant that genuinely needs a different number can override it there. A platform-level job is never subject to this (there's no tenant to meter). Exceeding it rejects the execution outright — see [Execution history](#execution-history-tss_job_execution) below for the `QUOTA_REJECTED` outcome this produces. |

### Jobs location

Also resolved via the three-tier model, under the `"jobs"` block — but these two values are themselves resolved **relative to `--scheduler-base-dir`**, not the current working directory, when they're relative paths.

| Env var / property | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_JOBS_ROOT_DIR` | `rootDir` | `jobs` | Directory `JobConfigLoader` walks — one subdirectory per tenant, each with its own `jobs.json`. |
| `TSS_PAYOS_CONFIG_FILE` | `payosConfigFile` | `payos.json` | The PayOS config file TSS reads tenant/platform database settings from (standalone mode only — see below). |

A `jobs/` root that doesn't exist, or that contains no job entries at all, is not an error — the scheduler starts normally with nothing registered.

### Example `tss-config.json`

The three blocks above — `database`, `scheduler`, `jobs` — all live in this one file (the JSON-file tier of the three-tier resolution; any of these fields can instead come from an environment variable or system property, which takes priority over the file).

```json
{
  "database": {
    "dbUrl": "jdbc:postgresql://db.internal:5432/tss",
    "dbUser": "tss_svc",
    "dbDriver": "org.postgresql.Driver",
    "dbVendor": "postgresql",
    "dbSchema": "tss",
    "dbMaxPoolSize": 10
  },
  "scheduler": {
    "schedulerInstanceName": "TSS-Cluster",
    "threadCount": 10
  },
  "jobs": {
    "rootDir": "jobs",
    "payosConfigFile": "payos.json"
  }
}
```

`dbPassword` is deliberately omitted here — set it via the `TSS_DB_PASSWORD` environment variable instead of writing it into this file. Unlike the kernel's `bootstrap.json`, TSS's own config loader does not expand `${...}`-style placeholders inside the JSON file, so a literal secret is the only thing a JSON `dbPassword` field would ever contain; the environment variable avoids that entirely, and also wins priority over the file either way. `rootDir` and `payosConfigFile` are shown here at their defaults — omit the whole `jobs` block entirely if you don't need to override either.

### `jobs/<tenantId>/jobs.json`

One file per tenant directory under the jobs root. A tenant routinely declares several jobs; only `implementation.type: "script"` + `trigger.type: "cron"` entries are loaded today (other combinations are Growth-phase and are skipped, not rejected, so a file mixing both kinds of entry still loads its supported ones).

```json
{
  "maxConcurrentExecutions": 3,
  "jobs": [
    {
      "id": "reconciliation-daily",
      "implementation": {
        "type": "script",
        "script": "jobs/reconciliation.js",
        "mappingFiles": ["/opt/payos-tss/model/Invoice.hbm.xml"]
      },
      "trigger": {
        "type": "cron",
        "expression": "0 0 2 * * ?"
      },
      "parallelizable": false
    }
  ]
}
```

- `maxConcurrentExecutions` — optional, top-level, sibling to `jobs`. Overrides `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT`'s global default for this one tenant specifically. Absent, or `0`/negative, means "no override — use the global default." Tenant-level operational policy, not a per-job setting — declared once here, never in `custom-jobs.json`.
- `id` — the job's declared identifier (used in logs; the job's actual unique key is `(tenantId, scriptPath)`, not `id`).
- `implementation.script` — the script path, resolved at execution time against the **shared script pool** (see below) — not against this tenant's own directory.
- `implementation.mappingFiles` — optional array of Hibernate mapping files this job's script needs on its tenant's `$DB` session. The tenant's effective mapping-file set is the **union** across all of that tenant's jobs.
- `trigger.expression` — a standard Quartz cron expression.
- `parallelizable` — parsed and stored, not yet enforced (every job currently schedules as parallel-safe).

### `jobs/<tenantId>/custom-jobs.json` — tenant-scoped overrides and additions

A second, optional file in the same tenant directory as `jobs.json`, for a tenant integrator's own jobs — either a brand-new job, or an override of an editor-provided `jobs.json` entry at the same `scriptPath`. Loaded after `jobs.json` and merged into the same per-tenant job map, so a matching `scriptPath` is replaced; a new one is simply added. Same entry schema and filtering rules as `jobs.json`, with one addition: every entry **must** declare a `tenant` field matching the enclosing directory exactly — a mismatch, or an absent field, causes that entry to be rejected (logged, skipped), never silently trusted or defaulted.

```json
{
  "jobs": [
    {
      "id": "tenant-override-reconciliation",
      "implementation": {
        "type": "script",
        "script": "jobs/reconciliation.js",
        "mappingFiles": ["/opt/payos-tss/model/Invoice.hbm.xml"]
      },
      "tenant": "acme-corp",
      "trigger": {
        "type": "cron",
        "expression": "0 30 2 * * ?"
      },
      "parallelizable": false
    }
  ]
}
```

In this example, if tenant `acme-corp`'s `jobs.json` already declares a job at `jobs/reconciliation.js`, this entry replaces it entirely (different cron expression, in this case); if it doesn't, this entry is simply added as a new job. A `custom-jobs.json` entry can never reach another tenant's job or a platform-level job, even when it declares the exact same literal `scriptPath` — the merge happens entirely within this one tenant's own job map, and `(tenantId-or-"platform", scriptPath)`, not `scriptPath` alone, is what Quartz actually keys on (see [The shared script pool](#the-shared-script-pool) below).

A tenant directory with no `custom-jobs.json` at all is not an error — same as a missing `jobs.json`.

### `jobs/platform-jobs.json` — platform-level jobs

A single file sitting directly under the `jobs/` root — **not** nested in any tenant directory. A platform-level job has no tenant owner; it's identified by `JobDefinition.tenantId() == null`, a plain null check, never a `"platform"` string sentinel (that literal is used only for the Quartz job group/log display — see below — and never flows back into the job's actual tenant field). Same entry schema as a tenant's `jobs.json` (`JobConfigLoader` shares its entry-parsing code between the two), same filtering rules (only `implementation.type: "script"` + `trigger.type: "cron"` loaded today).

```json
{
  "jobs": [
    {
      "id": "reference-data-refresh",
      "implementation": {
        "type": "script",
        "script": "jobs/refresh-reference-data.js",
        "mappingFiles": ["/opt/payos-tss/model/Currency.hbm.xml"]
      },
      "trigger": {
        "type": "cron",
        "expression": "0 0 3 * * ?"
      },
      "parallelizable": false
    }
  ]
}
```

A platform-level job's script gets **no `$DB` global at all** (there is no tenant to open a session for — `typeof $DB === 'undefined'`) and a **read+write** `$PlatformDB` (`PlatformJobDatabaseBinding`): writing its own platform data — a shared reference-data refresh, as above — is exactly what a platform-level job is for. Compare a tenant job's `$PlatformDB`, which is **read-only** (`PlatformDatabaseBinding`, no `save`/`update`/`delete`/`deleteById`/`executeUpdate` method at all) — a tenant job reading cross-tenant platform data is outside its own tenant's scope, so it can look but never write. Scheduled under the Quartz job group `"platform"` (`IJobConfigSpec.PLATFORM_LABEL`, the one shared constant `JobConfigLoader` and `SchedulerBootstrap` both reference) instead of a tenant id, purely so logs/`JobKey` lookups stay readable rather than falling into Quartz's own generic `"DEFAULT"` group.

A `jobs/` root with no `platform-jobs.json` at all is not an error — same as a tenant directory with no `jobs.json`.

### The shared script pool

`scriptLoader` (the function that turns a `scriptPath` into script source) has no `tenantId` parameter, so script files live in **one shared location**, keyed only by `scriptPath` — not nested under each tenant's own directory. In production this is `FileSystemScriptLoader`, which resolves `scriptPath` against `--scheduler-base-dir` directly: `implementation.script: "jobs/reconciliation.js"` reads `<scheduler-base-dir>/jobs/reconciliation.js`. Different tenants can schedule the exact same `scriptPath` — they read the identical file, just on their own cron schedule and against their own tenant's `$DB` (the `(scriptPath, tenantId)` Quartz `JobKey` is what keeps the two executions independent). A missing script file is not a startup error — it fails that specific job execution with a diagnostic naming the path, the first time it's triggered.

### Tenant and platform database resolution

TSS does **not** duplicate PayOS's tenant/database configuration model — it reads the real PayOS config and reuses the kernel's own resolution logic.

**Standalone mode:** `TenantDatabaseRegistryLoader.loadPayOSSettings` reads the file at `TSS_PAYOS_CONFIG_FILE` (default `payos.json`, resolved relative to `--scheduler-base-dir`) the same way `ConfigLoader.loadServerConfig` does in the kernel: that file is read only for its `configDir` key (default `"."`, relative to the file's own directory) — the actual settings (`database-service`, `multitenancy`, `applications`, `platform-database-service`) live in that directory's `*.json` files, merged together (every `*.json` file except `custom.json`, then `custom.json` last so it can override anything). This means tenant definitions are routinely found in a file separate from `payos.json` itself, exactly as in a real PayOS deployment. TSS deliberately does **not** call `ConfigLoader.loadServerConfig` itself — that method also runs kernel-boot side effects (service-adapter loading, secret-service/encryption-key initialization, `applications` loading) TSS must not repeat.

**Embedded mode:** no file is read at all — `RuntimeIntegrationHook` reuses `PayOSConfig.settings` and `PayOSConfig.getPlatformDatabaseService()` directly, since the host (`payos-runtime`) has already loaded them.

In both modes, each tenant's connection is resolved via `DatabaseServiceInitializer.resolveTenantDatabaseConnections` — the exact same database-declaration/tenant-reference/default-fallback rules the kernel itself applies at boot (a named `database-service.name` reference, or a fallback to the root default database). Only tenants with at least one job under `jobs/` are resolved into TSS's own registry; a tenant with jobs but no matching `multitenancy.tenants` entry in the PayOS config fails startup with a diagnostic naming the tenant.

`$PlatformDB` is resolved the same way in both modes now — standalone mode calls the kernel's own, unmodified `DatabaseServiceInitializer.initializePlatform(...)`; embedded mode reuses the host's already-initialized platform service. In both cases, a `platform-database-service` block that isn't declared simply means `$PlatformDB` is not bound for that execution — not an error.

Yes — `platform-database-service` is its own top-level block in one of PayOS's `configDir` `*.json` files (the same files `multitenancy`/`database-service` live in, not `tss-config.json`), with a `configuration` sub-block (same key names as a tenant `database-service`'s own `configuration` block) plus a required `mapping-files` array — `initializePlatform` fails startup with a diagnostic if the block is declared but `mapping-files` resolves to zero entries.

```json
{
  "platform-database-service": {
    "configuration": {
      "url": "jdbc:postgresql://db.internal:5432/payos_platform",
      "username": "platform_svc",
      "driver_class": "org.postgresql.Driver",
      "dialect": "org.hibernate.dialect.PostgreSQLDialect",
      "ddl-auto": "validate",
      "schema": "platform",
      "max-pool-size": 10
    },
    "mapping-files": ["model/Currency.hbm.xml", "model/Country.hbm.xml"]
  }
}
```

`password` is omitted here for the same reason as `tss-config.json`'s `dbPassword` above — but this file is read through the kernel's own `ConfigLoader.loadConfigByDirectory`, which (unlike TSS's own `tss-config.json` loader) runs `EnvVarResolver.resolve(...)` on every file it loads, expanding `${VAR}`-style tokens (including the `${VAR:-default}`/`${file:/path}`/etc. forms it supports). A real deployment typically sets `"password": "${PLATFORM_DB_PASSWORD}"` here rather than a literal secret. Every `mapping-files` entry is a Hibernate mapping file for the platform's own reference-data entities (e.g. shared currency/country tables) — see [Known limitation: relative file paths](#known-limitation-relative-file-paths) below if using a relative path here.

TSS never names the concrete database-service implementation anywhere in its own code: `TenantDatabaseRegistryLoader` produces `Map<String, DatabaseTenantDefinition>` (a plain, framework-neutral `payos-foundation` type), and the actual `IDatabaseService` is built from it via `ServiceLoader<IDatabaseServiceFactory>` — the same SPI mechanism, and the same provider (`dynamic-database-service`'s `DynamicDatabaseServiceFactory`), the kernel itself uses.

### Known limitation: relative file paths

Two things resolve relative paths against the JVM's **current working directory**, not `--scheduler-base-dir` or any PayOS base directory — a pre-existing kernel behavior TSS inherits unchanged, not something introduced by this module: a relative `jdbc:h2:file:...` URL, and a relative Hibernate mapping-file path declared without an `applications`-style base path (which includes every `implementation.mappingFiles` entry in `jobs.json`, and `platform-database-service.mapping-files`). Use **absolute** paths for both in any deployment where the process's working directory isn't guaranteed to be the scheduler base directory — this is the only reliable option today.

## Execution history (`tss_job_execution`)

TSS records every job execution in a table named `tss_job_execution`, in the same isolated schema as the Quartz `QRTZ_*` tables above (`TSS_DB_URL`/`tss-config.json`'s `database` block) — never a tenant or platform database. It reuses Quartz's own already-running connection pool rather than opening a second one.

Like Quartz's own schema, **TSS never creates this table at runtime.** Apply the checked-in DDL once, before starting TSS against a fresh schema: `src/main/resources/db/migration/tss_job_execution.sql` (shipped in the module's source tree, not in the runnable jar — copy it out, or keep a checkout of `payos-job-scheduler` around for this one file). It is plain, portable ANSI SQL and runs against PostgreSQL, Oracle, or H2 without modification.

Most executions are written in **two phases**, the same row throughout (keyed by `fire_instance_id`, never a second row for the same execution): a row is inserted the instant the execution starts, and updated in place when it completes. A **misfire** (Quartz detected a trigger whose fire time passed without the job ever starting) or a **quota rejection** (the tenant's `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` limit was already reached) never starts at all, so each is written as a single, already-terminal row instead — `started_at` and `completed_at` are identical, since there is no real duration to measure.

A node crash mid-execution does **not** leave that attempt's row permanently `completed_at IS NULL`. If the job is picked up by Quartz's own clustered recovery on a surviving node (always on for every job), that recovery execution is a **separate row** with its own `fire_instance_id`, never merged into the original — and the moment it starts, it closes out its crashed predecessor's still-open row with outcome `ABANDONED`, so the predecessor doesn't sit open forever and doesn't keep counting against the tenant's concurrency quota indefinitely. Querying by `(tenant_id, script_path)` (below) shows every attempt, including the `ABANDONED` one.

| Column | Meaning |
| --- | --- |
| `fire_instance_id` | Primary key — Quartz's own unique id for this one firing (a synthetic id for a misfire, which Quartz never actually fires). |
| `job_id` | The job definition's declared `id` (not unique by itself — see `jobs.json`'s own `id` note above). |
| `tenant_id` | `NULL` for a platform-level job. |
| `script_path` | The job's script path. |
| `node` | Which cluster node ran this execution, or detected the misfire (`Scheduler.getSchedulerInstanceId()` — the per-node id, not the shared `TSS_SCHEDULER_INSTANCE_NAME`). |
| `started_at` | Set on insert. |
| `completed_at` | `NULL` while the execution is still in progress — this is what "currently executing" means. Set immediately (equal to `started_at`) for a misfire/quota rejection. |
| `outcome` | `NULL` until completion; then `SUCCESS`, `SCRIPT_ERROR`, `MISFIRE`, `QUOTA_REJECTED`, or `ABANDONED` (a crashed node's row, closed out by the recovery execution that replaced it). |
| `failure_message` | Set only when `outcome = 'SCRIPT_ERROR'`. |

TSS has no network API for this (or anything else — see the shared script pool note above): the table itself is the query interface. Examples an operator or support engineer would run directly against TSS's database:

```sql
-- Everything currently executing, cluster-wide (not just on the node you happen to be connected through)
SELECT * FROM tss_job_execution WHERE completed_at IS NULL;

-- A specific job's full history, newest first — (tenant_id, script_path) is the job's real unique
-- key (the same one Quartz's own JobKey uses), not the non-unique job_id
SELECT * FROM tss_job_execution
WHERE tenant_id = 'acme-corp' AND script_path = 'jobs/reconciliation.js'
ORDER BY started_at DESC
LIMIT 20;

-- Every misfire or quota rejection in the last day, across all tenants
SELECT * FROM tss_job_execution
WHERE outcome IN ('MISFIRE', 'QUOTA_REJECTED') AND started_at > CURRENT_TIMESTAMP - INTERVAL '1' DAY;
```

## Metrics

Every finished execution also emits into PayOS's existing metrics infrastructure (`PayOSMetrics` — the same `IMetricsRecorder`/Micrometer-backed facade every other PayOS component already uses) — TSS adds no metrics-specific tooling of its own. Query TSS's job-execution metrics (logically `tss.job.execution.count`/`tss.job.execution.duration` — see below for the actual exposed names) through the exact same backend/dashboard you already use for every other PayOS metric, no separate system. Full tag/field shape and worked examples: [developer guide](../developer/task-scheduler-guide-v11-2026-10-08.md#11-metrics).

**Embedded mode:** nothing to configure — the host's own `payos-server-http` `/metrics` endpoint already scrapes TSS's samples (same JVM, same static registry).

**Standalone mode:** set `TSS_METRICS_HTTP_PORT` to a positive port (default `0` = disabled) to start a dedicated Prometheus scrape HTTP server on that port — standalone TSS has no HTTP server of its own otherwise, so without this, job-execution metrics are recorded but genuinely unreachable from outside the process. Point your Prometheus (or compatible) scraper at `http://<standalone-host>:<port>/metrics` as its own target. See the [developer guide §11.1](../developer/task-scheduler-guide-v11-2026-10-08.md#111-how-these-metrics-actually-get-exposed--embedded-vs-standalone) for the full explanation.

**A misconfigured port never stops the scheduler from starting.** If `TSS_METRICS_HTTP_PORT` points at a port already in use (or otherwise fails to bind), TSS logs a warning and continues running with job scheduling fully intact — just without metrics exposure for that run. Check the startup logs for `Failed to start the standalone metrics HTTP server` if `/metrics` isn't reachable and you expected it to be.

**`tenant_id` as a Prometheus label depends on deployment mode.** The underlying setting (`-Dpayos.metrics.tenant-tags-enabled`) is a PayOS-wide JVM flag, off by default for every component.
- **Standalone:** `StandaloneMain` sets it itself at startup (unless you've already set it yourself, in which case your own choice is respected). Safe to do unconditionally, since standalone TSS owns 100% of its own JVM — this can only ever affect TSS's own metrics.
- **Embedded:** `RuntimeIntegrationHook` deliberately does **not** touch this flag — in that mode TSS shares `payos-runtime`'s own JVM with the kernel's API metrics, connectors, and audit trail, and deciding that host's metrics-cardinality policy isn't TSS's call to make. `tenant_id` appears on TSS's metrics in embedded mode only if the host has already enabled the flag itself. If you run TSS embedded and want `tenant_id` on your TSS dashboards, set `-Dpayos.metrics.tenant-tags-enabled=true` on `payos-runtime`'s own JVM — the same setting then applies to every PayOS metric in that process, TSS's included.

The exact metric names as they actually appear in `/metrics` (not the logical names used elsewhere in this doc) are shown in the [developer guide's worked example](../developer/task-scheduler-guide-v11-2026-10-08.md#what-youll-actually-see-at-metrics).

## Clustering

Multiple TSS nodes sharing the same `TSS_SCHEDULER_INSTANCE_NAME` and pointed at the same TSS database form a Quartz cluster: job registration is idempotent (`registerScriptJobs` skips a `JobKey` that's already registered, logging it as expected rather than an error), and only one node fires a given trigger at a time. Point every node's `jobs/` root (standalone) or bundle (embedded) at the same content — a clustered node that registers a *different* `jobs.json` than its peers results in whichever node registers first owning that job's trigger.

## Logging

The daemon uses slf4j with logback; no `logback.xml` ships by default. Job registration logs the number of jobs and tenants registered at startup; each execution logs its `jobId`/`tenantId`/`scriptPath` on both success and failure paths.

**Sensitive-data masking — always on, nothing to configure.** Before a script error's message reaches `tss_job_execution.failure_message` or a log line, TSS masks sensitive-looking `key: value`/`key=value` fragments inside it (the same canonical denylist — `password`, `token`, `ssn`, `cardNumber`, etc. — every other PayOS component uses). This is deliberately best-effort, not a guarantee: a message with no recognizable key prefix at all (an unlabeled bare secret) passes through unmasked. Full behavior and worked examples: [developer guide §12](../developer/task-scheduler-guide-v11-2026-10-08.md#12-sensitive-data-masking).

## Next

- [Class diagram](../architecture/job-scheduler-class-diagram-v18-2026-10-07.md) — every class, how they relate, and the full execution-flow walkthrough (registration through a fired trigger through script execution through cleanup).
- [ADR-0009](../architecture/adr/0009-task-scheduling-system.md) — the architectural decision behind TSS's design.
- [Configuration: Task Scheduling System](../configuration/task-scheduling-system-v4-2026-10-07.md) — the config-key reference only, for a quick lookup without the full narrative.
- [Developer guide: Task Scheduling System](../developer/task-scheduler-guide-v11-2026-10-08.md) — the full developer guide: configuring TSS and tenant jobs, writing script jobs, the `context`/`$DB`/`$PlatformDB` surface, and the same execution-history/misfire/quota model from a job script author's point of view.
- [notification-service.md](notification-service.md) — the other standalone-daemon precedent this guide's structure follows.
- [deployment.md](deployment.md) — `payos-runtime`'s own deployment model, relevant to TSS's embedded mode.
