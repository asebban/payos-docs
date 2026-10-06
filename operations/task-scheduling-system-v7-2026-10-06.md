Created: 2026-10-02
Last updated: 2026-10-05
Version: 6

# Task Scheduling System (`payos-job-scheduler`)

Operator/developer guidance for running TSS: a Quartz-backed, clustered scheduler that registers and fires tenant JavaScript job scripts on a cron schedule, reading PayOS's own tenant/platform database configuration rather than duplicating it.

Like `payos-service-notification`, this is its **own deployable artifact** with its own entry point — it does not implement `IServer`/`ServerProvider` and is not booted from a bundle's `service-adapters-dir`. Unlike the notification daemon, it can also run **embedded inside `payos-runtime`** instead of as a separate process — see [Deployment modes](#deployment-modes) below. For the class-level design (every class, how they relate, the full execution-flow walkthrough), see the living [class diagram](../architecture/job-scheduler-class-diagram-v12-2026-10-02.md) and [ADR-0009](../architecture/adr/0009-task-scheduling-system.md); this page is the operational "how do I configure and run it" companion.

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

This is **TSS's own isolated schema** — the Quartz `QRTZ_*` tables (and `tss_job_execution` in a future story) — never a tenant or platform database. Resolved through `StandaloneConfigLoader`'s three-tier model, in priority order: environment variable → system property of the same name → a field in `tss-config.json`, nested under the matching block below. `TSS_CONFIG_FILE` (env var/system property) locates the JSON file itself; it defaults to `tss-config.json` in the scheduler base directory.

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

## Clustering

Multiple TSS nodes sharing the same `TSS_SCHEDULER_INSTANCE_NAME` and pointed at the same TSS database form a Quartz cluster: job registration is idempotent (`registerScriptJobs` skips a `JobKey` that's already registered, logging it as expected rather than an error), and only one node fires a given trigger at a time. Point every node's `jobs/` root (standalone) or bundle (embedded) at the same content — a clustered node that registers a *different* `jobs.json` than its peers results in whichever node registers first owning that job's trigger.

## Logging

The daemon uses slf4j with logback; no `logback.xml` ships by default. Job registration logs the number of jobs and tenants registered at startup; each execution logs its `jobId`/`tenantId`/`scriptPath` on both success and failure paths.

## Next

- [Class diagram](../architecture/job-scheduler-class-diagram-v12-2026-10-02.md) — every class, how they relate, and the full execution-flow walkthrough (registration through a fired trigger through script execution through cleanup).
- [ADR-0009](../architecture/adr/0009-task-scheduling-system.md) — the architectural decision behind TSS's design.
- [notification-service.md](notification-service.md) — the other standalone-daemon precedent this guide's structure follows.
- [deployment.md](deployment.md) — `payos-runtime`'s own deployment model, relevant to TSS's embedded mode.
