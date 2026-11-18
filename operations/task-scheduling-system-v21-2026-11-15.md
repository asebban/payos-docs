Created: 2026-10-02
Last updated: 2026-11-15
Version: 21

# Task Scheduling System (`payos-job-scheduler`)

Operator/developer guidance for running TSS: a Quartz-backed, clustered scheduler that registers and fires tenant JavaScript job scripts on a cron schedule, reading PayOS's own tenant/platform database configuration rather than duplicating it.

Like `payos-service-notification`, this is its **own deployable artifact** with its own entry point — it does not implement `IServer`/`ServerProvider` and is not booted from a bundle's `service-adapters-dir`. Unlike the notification daemon, it can also run **embedded inside `payos-runtime`** instead of as a separate process — see [Deployment modes](#deployment-modes) below. For the class-level design (every class, how they relate, the full execution-flow walkthrough), see the living [class diagram](../architecture/job-scheduler-class-diagram-v24-2026-11-15.md) and [ADR-0009 v5](../architecture/adr/0009-task-scheduling-system-v5-2026-11-13.md); this page is the operational "how do I configure and run it" companion.

## Deployment modes

| | Standalone (`StandaloneMain`) | Embedded (`RuntimeIntegrationHook`) |
| --- | --- | --- |
| Entry point | `ma.s2m.payos.scheduler.bootstrap.StandaloneMain.main(...)` | `RuntimeIntegrationHook.start(...)`, called by `payos-runtime` during its own startup |
| Process | Its own JVM, independently scalable from `payos-runtime`'s replica count | Shares `payos-runtime`'s JVM and lifecycle |
| `jobs/` and script files resolve against | `--scheduler-base-dir <path>` (defaults to the process's working directory) | `PayOSConfig.getRuntimeBaseDirectory()` — the host's own `--bundle-path` |
| Tenant/platform database settings come from | A PayOS config file this module loads itself (see [Tenant and platform database resolution](#tenant-and-platform-database-resolution)) | The host's already-loaded `PayOSConfig.settings`/`PayOSConfig.getPlatformDatabaseService()` — no file is re-read |
| A job-registration failure (bad tenant config, etc.) | **Propagates** — fails the whole startup. TSS is this process's only job, so fail-fast is correct. | **Logged, not propagated** — TSS is a secondary subsystem inside `payos-runtime` and must not take the whole runtime down over a problem scoped to TSS's own jobs |
| TSS's own Quartz schema DB (`tss-config.json`) | Resolved via `StandaloneConfigLoader`'s three-tier model (below) | Same — this part doesn't change between modes |
| Jobs hot reload and the admin HTTP endpoint | `TSS_HOT_RELOAD_ENABLED` (on by default), `TSS_ADMIN_HTTP_PORT` (off by default) | `task-scheduler.scheduler.hotReloadEnabled`, `task-scheduler.admin.httpPort` — same behavior, the endpoint secured by the host's own `security` configuration |

Both modes delegate all scheduler construction and job registration to the same `SchedulerBootstrap`/`TenantDatabaseRegistryLoader` code, so the two deployment modes cannot drift apart in behavior.

## Building

```bash
cd payos-job-scheduler
mvn clean package
```

This produces two jars side by side:

- `target/payos-job-scheduler-<version>-standalone.jar` — the self-contained, runnable uber-jar (`maven-shade-plugin`, `standalone` classifier), `Main-Class: ma.s2m.payos.scheduler.bootstrap.StandaloneMain`, every dependency bundled — GraalVM polyglot/JS, Hibernate, Quartz, HikariCP, the H2/PostgreSQL JDBC drivers, `dynamic-database-service`'s `IDatabaseServiceFactory` SPI provider, `tenant-config-loader-database`'s `IConfigLoader` provider (inactive unless configured — see [Tenant and platform database resolution](#tenant-and-platform-database-resolution)), and the `mappings/PayosQueueOutbox.hbm.xml` mapping resource from `payos-server-queue`. No separate classpath assembly is needed to run it. It also carries the DDL scripts to apply before the first start (Quartz's `org/quartz/impl/jdbcjobstore/tables_<vendor>.sql` and `db/migration/tss_job_execution.sql`).
- `target/payos-job-scheduler-<version>.jar` — the plain jar (TSS's own classes only, not runnable), the artifact `payos-runtime` depends on for embedded mode, so the runtime's own shade step bundles shared libraries exactly once.

## Starting the service (standalone)

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss
```

`run` (start the service) is the default command, so it may be omitted: `java -jar <jar> run` is equivalent. `--scheduler-base-dir` is optional; omitted, it defaults to the process's current working directory. `tss-config.json`, `jobs/`, job scripts and the PayOS config file are resolved relative to this one directory unless overridden individually (see below). Relative H2 `file:` URLs and mapping-file paths are **not**: they resolve against the process's working directory — start TSS from its base directory, or use absolute paths for those.

The complete start-up procedure — prerequisites from A to Z, database scripts, exact commands for Linux, Windows and an extra JDBC driver, expected log lines, and every command-line option — is in the [TSS start-up and command-line guide](tss-startup-and-cli-guide-v2-2026-11-13.md) (French).

### Stopping

`StandaloneMain` registers a JVM shutdown hook, so a normal `SIGTERM` triggers `SchedulerBootstrap.shutdown(true)` — it waits for any in-flight job executions to finish before the process exits. Avoid `kill -9`/`SIGKILL` where possible — it skips the shutdown hook entirely and can leave a job mid-execution.

## Configuration

### TSS's own database (`tss-config.json`)

This is **TSS's own isolated schema** — the Quartz `QRTZ_*` tables and `tss_job_execution` — never a tenant or platform database. Resolved through `StandaloneConfigLoader`'s three-tier model, in priority order: environment variable → system property of the same name → a field in `tss-config.json`, nested under the matching block below. `TSS_CONFIG_FILE` (env var/system property) locates the JSON file itself; it defaults to `tss-config.json`, and a relative value resolves against the **scheduler base directory** (`--scheduler-base-dir`), like `jobs/`. Builds before 2026-11-09 resolved it against the working directory; a file found only there is still used, with a `WARN` asking to move it. When no file is found, TSS logs `No TSS config file at '<path>'` and takes every value from the environment and system properties — so the start-up then fails on `Missing required configuration 'TSS_DB_URL'` unless that comes from the environment.

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
| `TSS_MAX_PARALLEL_WORKERS_PER_JOB` | `maxParallelWorkersPerJob` | `4` | Ceiling on the worker threads one `$Parallel.forEach` call may use inside a single execution. Each worker opens its own tenant session and may also read the (read-only) `$PlatformDB`, so size the tenant and platform connection pools for up to `threadCount × maxParallelWorkersPerJob` concurrent connections per node. |
| `TSS_METRICS_HTTP_PORT` | `metricsHttpPort` | `0` (disabled) | Standalone only — a positive port starts a dedicated Prometheus `GET /metrics` server. See [Metrics](#metrics). |

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

One file per tenant directory under the jobs root. A tenant routinely declares several jobs; only `implementation.type: "script"` entries are loaded. Each entry's `triggers` array accepts any mix of `cron`/`interval`/`once`/`manual`/`queue`/`dependency` — full schema and worked examples in the [developer guide](../developer/task-scheduler-guide-v20-2026-11-18.md#trigger-types). The former single `trigger: {...}` object is no longer accepted: an entry still using it is rejected as a whole, with a warning naming it, while every other entry in the same file still loads.

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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 2 * * ?" }],
      "parallelizable": false
    }
  ]
}
```

- `maxConcurrentExecutions` — optional, top-level, sibling to `jobs`. Overrides `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT`'s global default for this one tenant specifically. Absent, or `0`/negative, means "no override — use the global default." Tenant-level operational policy, not a per-job setting — declared once here, never in `custom-jobs.json`.
- `id` — the job's declared identifier (used in logs; the job's actual unique key is `(tenantId, scriptPath)`, not `id`).
- `implementation.script` — the script path, resolved at execution time against the **shared script pool** (see below) — not against this tenant's own directory.
- `implementation.mappingFiles` — optional array of Hibernate mapping files this job's script needs on its tenant's `$DB` session. The tenant's effective mapping-file set is the **union** across all of that tenant's jobs. Each entry is first looked up as a file on disk; when no such file exists, it is loaded as a **classpath resource** — from the standalone jar or the JARs of the PayOS `service-adapters-dir` in standalone mode, from the runtime's classpath and the same `service-adapters-dir` in embedded mode. A mapping shipped inside a connector, such as `mappings/PayosAuditEvent.hbm.xml`, is therefore declared by its resource path. The same rule applies to the `platform-database-service.mapping-files` of the PayOS configuration.
- `triggers` — array of triggers, any mix of `cron`/`interval`/`once`/`manual`/`queue`/`dependency`; absent or empty means the job only ever runs when triggered by hand. See [Event-driven triggers](#event-driven-triggers) below for how `manual`/`queue`/`dependency` behave operationally, cluster-wide.
- `arguments` — optional free-form JSON object handed to the script as `context.getArguments()`. Stored with the job in the TSS database and kept up to date by reconciliation, so an edited value reaches every node through hot reload or `sync`, without a restart (see [Changing jobs without a restart](#changing-jobs-without-a-restart)). `${...}` placeholders are not expanded in `jobs.json` — keep secrets out of it. See the [developer guide §6.1](../developer/task-scheduler-guide-v20-2026-11-18.md#61-passing-arguments-to-a-script-arguments).
- `parallelizable` — `false` schedules the job as `ExclusiveScriptJob` (`@DisallowConcurrentExecution`): this `(tenant, script)` job never runs twice at once anywhere in the cluster. A trigger firing during a running execution waits for it; if it waits beyond Quartz's misfire threshold (60 s) it is recorded as `MISFIRE` and fired once to catch up. `true` (the default) imposes no such restriction.

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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 30 2 * * ?" }],
      "parallelizable": false
    }
  ]
}
```

In this example, if tenant `acme-corp`'s `jobs.json` already declares a job at `jobs/reconciliation.js`, this entry replaces it entirely (different cron expression, in this case); if it doesn't, this entry is simply added as a new job. A `custom-jobs.json` entry can never reach another tenant's job or a platform-level job, even when it declares the exact same literal `scriptPath` — the merge happens entirely within this one tenant's own job map, and `(tenantId-or-"platform", scriptPath)`, not `scriptPath` alone, is what Quartz actually keys on (see [The shared script pool](#the-shared-script-pool) below).

A tenant directory with no `custom-jobs.json` at all is not an error — same as a missing `jobs.json`.

### `jobs/platform-jobs.json` — platform-level jobs

A single file sitting directly under the `jobs/` root — **not** nested in any tenant directory. A platform-level job has no tenant owner; it's identified by `JobDefinition.tenantId() == null`, a plain null check, never a `"platform"` string sentinel (that literal is used only for the Quartz job group/log display — see below — and never flows back into the job's actual tenant field). Same entry schema as a tenant's `jobs.json` (`JobConfigLoader` shares its entry-parsing code between the two), same filtering rule (only `implementation.type: "script"` loaded). A platform job's `queue` trigger has one distinct behavior worth knowing here: it sees **every** message on its destination regardless of `tenantId` metadata, unlike a tenant job's `queue` trigger (default-deny on a missing/different `tenantId`) — see [Event-driven triggers](#event-driven-triggers) below.

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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 3 * * ?" }],
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

**Standalone mode:** `TenantDatabaseRegistryLoader.loadPayOSSettings` reads the file at `TSS_PAYOS_CONFIG_FILE` (default `payos.json`, resolved relative to `--scheduler-base-dir`) the same way `ConfigLoader.loadServerConfig` does in the kernel: that file is read only for its `configDir` key (default `"."`, relative to the file's own directory) — the actual settings (`database-service`, `multitenancy`, `applications`, `platform-database-service`) live in that directory's `*.json` files, merged together (every `*.json` file except `custom.json`, then `custom.json` last so it can override anything). This means tenant definitions are routinely found in a file separate from `payos.json` itself, exactly as in a real PayOS deployment. TSS deliberately does **not** call `ConfigLoader.loadServerConfig` itself — that method also runs kernel-boot side effects (service-adapter loading, secret-service/encryption-key initialization, `applications` loading) TSS must not repeat. It does, however, run the one step of it that can change which tenants exist: every SPI-discovered `IConfigLoader` is invoked on the merged settings, each contribution deep-merged on top (the loader wins on conflict), a failing loader logged and skipped — exactly the kernel's own contract, so a given configuration yields the same tenant set in standalone TSS as in the runtime.

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

### Running standalone on its own node — avoiding a copy of the tenant configuration

A standalone node typically has no `payos-runtime` on it. TSS never contacts a runtime process — it only reads the configuration files described above from its own disk — so the node needs a subset of the PayOS configuration directory. Two ways to keep that subset from drifting away from the runtime's:

1. **Declare tenants in the database (recommended).** With `tenant-config-loader-database`, `multitenancy.tenants` comes from a registry table in the platform database, read through the same `IConfigLoader` step the runtime runs. The loader is already bundled in the standalone jar; the node then only needs the platform connection and the loader's column mapping, both stable — no tenant list to copy:

   ```json
   {
     "platform-database-service": {
       "configuration": {
         "url": "jdbc:postgresql://db.internal:5432/payos_platform",
         "username": "platform_svc",
         "password": "${PLATFORM_DB_PASSWORD}",
         "driver_class": "org.postgresql.Driver"
       },
       "mapping-files": ["/opt/payos-tss/model/Currency.hbm.xml"]
     },
     "tenant-config-loader": { "enabled": true, "table": "tenant_registry" }
   }
   ```

   Tenants declared in JSON (if any remain) are merged alongside the registry's; a registry row wins on conflict. Column names and every optional field are documented in [tenant-config-loader-database](../developer/tenant-config-loader-database-v1-2026-10-22.md).
2. **Mount the same configuration directory** on both nodes (a Kubernetes ConfigMap/Secret mounted into both pods, a shared volume, or one configuration artifact deployed by the same pipeline) instead of maintaining two copies.

Either way, TSS resolves tenants **once, at startup**: a tenant added to the registry or the shared files is picked up on the next TSS start — hot reload and `sync` never schedule a brand-new tenant's jobs before then (see [Safety rules](#safety-rules)). Things to know before relying on this:

- **Secrets:** use `${VAR}`, `${VAR:?message}` or `${file:/run/secrets/...}` placeholders in these files — they are expanded (see the `platform-database-service` note above). The PayOS **secret service is not initialized** in standalone TSS, so a value the runtime would fetch from it is not resolved here; provide it through an environment variable or a mounted file.
- **Encrypted configuration files are not supported in standalone mode.** Files encrypted with `payosv2-packer` cannot be decrypted, because standalone TSS never initializes the editor encryption key: such a file is logged as failing to load and skipped — if it holds `multitenancy`, startup then fails with a "tenant not declared" diagnostic. Ship these files unencrypted (with secrets as placeholders) on a TSS node.
- **Every declared tenant must be valid.** Tenant connections are resolved and validated for every tenant under `multitenancy.tenants` in one pass, including tenants with no jobs — a broken declaration blocks startup, as it does for the runtime.
- **The PayOS `service-adapters-dir` is loaded, as in the runtime.** The JARs in the directory named by `service-adapters-dir` (resolved against the PayOS config file's own directory when relative; `PAYOS_SERVICE_ADAPTERS_DIR` or the `service-adapters-dir` system property take priority, as in the kernel) supply their `IConfigLoader` and `IDatabaseServiceFactory` providers, and the Hibernate mapping files they ship as classpath resources resolve when a tenant or the platform database lists them — e.g. `mappings/PayosAuditEvent.hbm.xml` from `payos-database-audit-trail`. `mappings/PayosQueueOutbox.hbm.xml`, which the runtime gets from `payos-server-queue`, is bundled in the standalone jar itself.

### Known limitation: relative file paths

Two things resolve relative paths against the JVM's **current working directory**, not `--scheduler-base-dir` or any PayOS base directory — a pre-existing kernel behavior TSS inherits unchanged, not something introduced by this module: a relative `jdbc:h2:file:...` URL, and a relative Hibernate mapping-file path declared without an `applications`-style base path (which includes every `implementation.mappingFiles` entry in `jobs.json`, and `platform-database-service.mapping-files`). Use **absolute** paths for both in any deployment where the process's working directory isn't guaranteed to be the scheduler base directory — this is the only reliable option today.

## Changing jobs without a restart

The `jobs/` files are the source of truth for what is scheduled. One reconciliation step compares them with the jobs Quartz holds in the TSS database and applies only the differences. It runs at every node startup, on hot reload when `jobs/` changes, and on demand through `sync` (command line or HTTP, see [Administration](#administration-manual-trigger-sync-and-list)).

| Change in the jobs files | What reconciliation does |
| --- | --- |
| New job | Scheduled. |
| A temporal trigger's (`cron`/`interval`/`once`) definition changed | That one Quartz trigger is replaced; the next fire time follows the new definition. Every other trigger of the job is untouched. |
| An event-driven trigger (`manual`/`queue`/`dependency`) added, removed or its definition changed | Written to the job's stored trigger list immediately; each node starts/stops it in memory at its own next event-trigger refresh (default every 30 s, `eventTriggerRefreshSeconds`) — see [Event-driven triggers](#event-driven-triggers). |
| `parallelizable`, `id` or `arguments` changed | The job definition is replaced in place; its triggers and next fire time are kept. |
| Job removed (from `jobs.json`, `custom-jobs.json` or `platform-jobs.json`, or its whole tenant directory removed) | Deleted from the scheduler, with a `WARN` log line: `Job '<tenant>/<script>' is no longer declared in any jobs file — deleted from the scheduler`. An execution already running finishes normally. |
| Nothing | Nothing is touched: triggers keep their next fire time and misfire state. |

Since `arguments` are stored with the job in the TSS database, an edited value reaches every node of the cluster as soon as one node has reconciled.

### Safety rules

- **An unreadable `jobs.json` or `platform-jobs.json`** (invalid JSON, typically a file caught mid-write) abandons the whole pass: the scheduler is left exactly as it was, an error is logged, and the next change retries. At startup, standalone mode still fails fast on it, as before.
- **An unreadable `custom-jobs.json`** only lets that tenant's missing jobs be added: none of its existing jobs is updated or deleted until the file reads cleanly again, so a broken override file never unschedules custom jobs.
- **A missing `jobs/` directory** (wrong `TSS_JOBS_ROOT_DIR`, volume not mounted) is never read as "no jobs": nothing is deleted.
- **A brand-new tenant** (no job of its own in the scheduler yet) is only scheduled when the nodes restart, because a node opens a tenant's database at startup: hot reload and `sync` report its jobs as skipped (`new tenant: restart the scheduler nodes to open its database`). A changed `mappingFiles` is likewise logged and applied at the next restart. Restart every node (rolling restart) so each one serves the new tenant.
- **An invalid cron expression** fails that job only: it is reported and logged, its previous trigger stays in force, and every other job is reconciled normally.

### New tenants: directories created from `jobs/template.json`

Before every reconciliation (startup, hot reload, `sync`), TSS creates `jobs/<tenantId>/` for each tenant declared under `multitenancy.tenants` in the PayOS configuration that has none yet, with a `jobs.json` made from `jobs/template.json` when that file exists — every `{{tenantId}}` in it replaced by the tenant's id (see [configuration §`jobs/template.json`](../configuration/task-scheduling-system-v11-2026-11-15.md#jobstemplatejson--model-for-new-tenants)) — or an empty directory otherwise. The tenant list is re-read each time: standalone nodes and the `sync` command reload the PayOS config file (`TSS_PAYOS_CONFIG_FILE`, including config loaders such as a database tenant registry); embedded mode reads the host's live configuration. Existing tenant directories and files are never modified, an invalid `template.json` is never copied, and nothing is created when the jobs root is missing. Typical sequence for a new tenant: declare it in PayOS, run `sync` (or wait for the next reload) to create its directory and adjust its `jobs.json` if needed, then restart the nodes — its jobs are scheduled at that restart, like any new tenant. A tenant removed from the PayOS configuration keeps its directory: remove it yourself, otherwise the next startup refuses to register jobs with a "not declared under multitenancy.tenants" diagnostic (standalone: startup fails; embedded: the scheduler runs with no job registered).

### Hot reload

On by default (`TSS_HOT_RELOAD_ENABLED`, embedded: `task-scheduler.scheduler.hotReloadEnabled`). TSS uses the same `ConfigWatcher` as PayOS's own configuration hot reload: it watches `jobs/` and every tenant directory present at startup, waits about 2 s after the last change (an editor save or a multi-file copy produces a burst of events), then re-reads the files and reconciles. Quartz is never stopped: executions in progress are not interrupted and no trigger is missed. Hot reload stays off when no job at all was declared at startup.

File-system notifications are not delivered everywhere: a network share (NFS, SMB) modified from another host, or a Kubernetes ConfigMap update (swapped through its `..data` link, never reported as a `.json` change), go unnoticed — the same limitation as PayOS's own hot reload. Run `sync` after such a change.

Every node watches and reconciles on its own; two nodes applying the same change is harmless, the second one finds nothing left to do. All nodes must therefore see the same `jobs/` content: a node with an outdated copy would, when it restarts or reloads, reconcile the cluster back to its version.

## Event-driven triggers

Unlike a temporal trigger (`cron`/`interval`/`once`), which Quartz itself tracks centrally with no per-node state, a `manual`/`queue`/`dependency` trigger is started and stopped **in memory, independently on every node** — each node's own `EventTriggerRuntime` compares what's declared against what it has already started, at startup, right after this node's own local reconciliation, and then every `eventTriggerRefreshSeconds` (default 30 s, `TSS_EVENT_TRIGGER_REFRESH_SECONDS`). A change made on, or seen first by, one node (a hot-reloaded edit, another node's `sync`) therefore reaches *this* node within that same window, not instantly — budget for it when timing a rollout that adds, removes or edits an event-driven trigger. If a trigger fails to start on a node (the broker is unreachable, say), that failure is logged and the next refresh retries it automatically — it is never a fatal error for the node.

**`manual`** needs nothing running to be ready — it's a passive target for the CLI/HTTP operations below, documented there.

**`queue`** opens TSS's own connection to a message broker — see [Configuration §`queue`](../configuration/task-scheduling-system-v11-2026-11-15.md#queue-block--tsss-own-broker-connection) for `TSS_QUEUE_TYPE`/`HOST`/`PORT`/`ADAPTERS_DIR`. This connection is entirely TSS's own, never the host PayOS application's own queue server (`payos-server-queue`) or a notification connector's — the same way TSS's database is its own isolated schema, never a tenant's. Absent `TSS_QUEUE_TYPE`, no queue client is connected at all, and a declared `queue` trigger then simply never starts (logged, retried at every subsequent refresh) until it's configured. Three operational points worth knowing before relying on `queue` triggers in a cluster:
- **One destination, one shared consumer group, cluster-wide.** Several of your nodes subscribing to the same destination share its message load between them rather than each receiving every message — this is what keeps a single message from firing the job more than once across the cluster. The flip side: **two different jobs that both declare a `queue` trigger on the same destination also share that destination's messages between them**, each message reaching only one of the two jobs, never both. Give each job its own destination unless you specifically want several jobs splitting one stream's load.
- **Tenant visibility is fixed.** A tenant job's `queue` trigger only fires for a message whose metadata carries a matching `tenantId`; a platform job's fires for every message on the destination regardless of `tenantId` — its one deliberate cross-tenant integration point.
- **The *initial* broker connection (standalone mode) used to be a node-level failure, not just a trigger-level one — fixed 2026-11-15.** The paragraph above (line 309) describes a declared trigger failing to *start* once the scheduler is already running — always non-fatal. Opening TSS's own connection the very first time, at node startup, was a separate code path in standalone mode (`StandaloneMain`) that did **not** have the same tolerance: an unreachable broker there used to kill the whole standalone process before it ever registered a single job, `cron`/`interval`/`once`/`manual`/`dependency` included, none of which have anything to do with `queue`. Confirmed with a real broker and fixed to match embedded mode's own "secondary subsystem" rule (`RuntimeIntegrationHook`, always non-fatal): a connection failure at node startup is now logged the same way, in both modes, and the node starts normally with every other trigger type working — only `queue` triggers stay off until connectivity is restored.
- **`destination` naming is connector-specific — confirmed against a real broker, 2026-11-15.** `queue-service-nats`'s `NatsQueueClient` only accepts letters, digits, `_` and `-` in `destination` (it is used directly as the JetStream stream name, which cannot contain `.`): a dotted, subject-hierarchy name otherwise idiomatic for NATS (`payments.settled`) makes the trigger fail to start (logged, retried — never fatal, same as any other start failure above) until renamed `payments-settled`-style.

**`dependency`** needs `db/migration/tss_dependency.sql` applied to TSS's own database beforehand (same procedure as `tss_job_execution.sql`, [start-up guide §3](tss-startup-and-cli-guide-v2-2026-11-13.md#3-la-base-de-données-de-tss)) — without it, every upstream job completion is logged as an error and never counted, and the downstream job never fires. The lock that keeps a multi-node cluster from firing the downstream job twice, or never, when two upstream jobs finish on different nodes at nearly the same instant is a row-level `SELECT ... FOR UPDATE` on TSS's own database, held only for the instant it takes to record one completion and check whether every upstream has now finished — never held while the downstream job is actually fired. See the [class diagram §17](../architecture/job-scheduler-class-diagram-v24-2026-11-15.md) for the full mechanics. Two things this does **not** clean up by itself: a completion recorded while the database is briefly unreachable is logged as an error and never counted (that upstream's completion is simply lost for this round), and a downstream job or trigger that's later removed leaves its lock/state rows behind, inert but never deleted.

## Administration: manual trigger, sync and list

Three operations, available both on the command line and as an HTTP endpoint, and audited the same way through either channel.

| Operation | Effect |
| --- | --- |
| `trigger` | Fires one job once, now, outside its cron schedule, on whichever node picks it up. Every rule of a scheduled run applies: exclusivity for a `parallelizable: false` job, tenant quota, execution history, metrics — tagged `triggerType=manual` instead of `cron`. The cron schedule itself is unaffected. |
| `sync` | Re-reads `jobs/` and runs the reconciliation above. Returns its report: `added`, `rescheduled`, `updated`, `deleted`, `skipped`, `failed`. |
| `list` | The scheduled jobs: tenant, script, id, `parallelizable`, next and previous fire time (the earliest/latest across the job's own temporal triggers), and its full `triggers` array (each trigger's id, type, declared definition, and its own next/previous fire time when it's a temporal one — `null` for an event-driven one, which has no schedule of its own). |

### Command line

The standalone jar takes a command after its options (`run`, the default, starts the service):

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss list --tenant acme
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss trigger --tenant acme --script jobs/acme/daily-report.js
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss trigger --platform --script jobs/compliance-export.js
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss sync
```

The command reads `tss-config.json` like the service and connects to the TSS database through a Quartz instance that is never started: it never fires a job and never joins the cluster, so it can run on any host with database access while the nodes are running — including against an embedded deployment, which shares the same TSS database. Against an embedded deployment, point the command at that deployment's database **and** scheduler name, because Quartz scopes every row by scheduler name: set `TSS_DB_URL` (and credentials) to the `task-scheduler.database` values and `TSS_SCHEDULER_INSTANCE_NAME` to `task-scheduler.scheduler.schedulerInstanceName`. With another name, `list` returns no job and `trigger` answers "unknown job". Access is controlled by the TSS database credentials. `sync` reconciles against the `jobs/` files of the host it runs on. The result is printed as JSON; exit codes: `0` success, `1` a `sync` that failed on some jobs, `2` usage error, `3` unknown job. Each command is audited under the operating-system user name.

### HTTP endpoint

Enabled by `TSS_ADMIN_HTTP_PORT` (embedded: `task-scheduler.admin.httpPort`), on its own port in both modes so it can be firewalled separately from business traffic. Like hot reload, it stays off when no job was declared at startup.

| Request | Body | Response |
| --- | --- | --- |
| `GET /tss/admin/jobs[?tenantId=acme]` | — | `200` `{"jobs": [...]}` |
| `POST /tss/admin/jobs/trigger` | `{"tenantId": "acme", "scriptPath": "jobs/acme/daily-report.js"}` — `tenantId` omitted for a platform job | `202` triggered, `404` unknown job, `400` no `scriptPath` |
| `POST /tss/admin/sync` | — | `200` with the reconciliation report |

```bash
curl -X POST https://tss-admin.internal:8095/tss/admin/jobs/trigger \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: acme" -H "Content-Type: application/json" \
  -d '{"tenantId": "acme", "scriptPath": "jobs/acme/daily-report.js"}'
```

**Authentication.** Every call needs `Authorization: Bearer <access token>`; without it the answer is `401`, never a login redirect. The token is verified by PayOS's own security service, exactly as for a Bearer call to the runtime: signature against the identity provider's published keys, issuer, expiry. The audience is deliberately not checked by that service (so that `client_credentials` tokens work), which makes the TSS roles the real gate: grant them only to the accounts and clients that need them. In a multi-tenant setup, send `X-Tenant-Id` so the token is verified against that tenant's realm; without it, the global `security` configuration applies. Bearer tokens are validated by the `nimbus` security provider, the PayOS default.

**Authorization.**

| Token role | Allowed |
| --- | --- |
| `tss-platform-admin` (`TSS_ADMIN_PLATFORM_ROLE`) | Every operation, on every tenant and on platform jobs. |
| `tss-tenant-admin` (`TSS_ADMIN_TENANT_ROLE`) | `trigger` and `list` on its own tenant only — the tenant derived from the token's issuer realm, which the request cannot forge. Never `sync`, never a platform job. |
| Neither | `403`. |

**Standalone mode** reads the `security` configuration from the PayOS config file (`TSS_PAYOS_CONFIG_FILE`), so the node must be able to reach the identity provider. The session store is forced to `memory` for TSS, since the endpoint only serves stateless Bearer calls and a `redis` store module is not bundled in the standalone jar.

### Audit trail

Every `trigger` and `sync`, from either channel, is written through PayOS's audit facade (`AuditLogger`), so it lands wherever the platform's audit trail goes:

| Event | `result` | Content |
| --- | --- | --- |
| `TSS_JOB_TRIGGERED` | `SUCCESS`, `NOT_FOUND`, `FAILURE` | `userId` (token principal or OS user), `tenantId`, `resourceId` (`<tenant or platform>/<scriptPath>`), `channel` (`http` or `cli`) |
| `TSS_JOBS_SYNCHRONIZED` | `SUCCESS`, `PARTIAL`, `FAILURE` | `userId`, `channel` |
| `TSS_ADMIN_DENIED` | `DENIED` | a tenant administrator reaching outside its tenant: `userId`, its `tenantId`, the `operation` and `targetTenantId` |

PayOS's own authorization records (granted or denied by role) are written as well. A manually triggered execution's `tss_job_execution` row looks like any other — the table has no trigger-type column; the audit trail and the `triggerType` metric tag tell manual runs apart.

## Execution history (`tss_job_execution`)

TSS records every job execution in a table named `tss_job_execution`, in the same isolated schema as the Quartz `QRTZ_*` tables above (`TSS_DB_URL`/`tss-config.json`'s `database` block) — never a tenant or platform database. It reuses Quartz's own already-running connection pool rather than opening a second one.

Like Quartz's own schema, **TSS never creates this table at runtime.** Apply the checked-in DDL once, before starting TSS against a fresh schema: `db/migration/tss_job_execution.sql`, shipped inside the standalone jar (extract it with `jar xf <jar> db/migration/tss_job_execution.sql`) and in the module's source tree under `src/main/resources/`. See the [start-up guide §3](tss-startup-and-cli-guide-v2-2026-11-13.md#3-la-base-de-données-de-tss) for the commands per database, and for the Quartz scripts, whose PostgreSQL and Oracle versions drop every `QRTZ_*` table first. It is plain, portable ANSI SQL and runs against PostgreSQL, Oracle, or H2 without modification.

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

Every finished execution also emits into PayOS's existing metrics infrastructure (`PayOSMetrics` — the same `IMetricsRecorder`/Micrometer-backed facade every other PayOS component already uses) — TSS adds no metrics-specific tooling of its own. Query TSS's job-execution metrics (logically `tss.job.execution.count`/`tss.job.execution.duration` — see below for the actual exposed names) through the exact same backend/dashboard you already use for every other PayOS metric, no separate system. Full tag/field shape and worked examples: [developer guide](../developer/task-scheduler-guide-v20-2026-11-18.md#12-metrics).

**Embedded mode:** nothing to configure — the host's own `payos-server-http` `/metrics` endpoint already scrapes TSS's samples (same JVM, same static registry).

**Standalone mode:** set `TSS_METRICS_HTTP_PORT` to a positive port (default `0` = disabled) to start a dedicated Prometheus scrape HTTP server on that port — standalone TSS has no HTTP server of its own otherwise, so without this, job-execution metrics are recorded but genuinely unreachable from outside the process. Point your Prometheus (or compatible) scraper at `http://<standalone-host>:<port>/metrics` as its own target. See the [developer guide §12.1](../developer/task-scheduler-guide-v20-2026-11-18.md#121-how-these-metrics-actually-get-exposed--embedded-vs-standalone) for the full explanation.

**A misconfigured port never stops the scheduler from starting.** If `TSS_METRICS_HTTP_PORT` points at a port already in use (or otherwise fails to bind), TSS logs a warning and continues running with job scheduling fully intact — just without metrics exposure for that run. Check the startup logs for `Failed to start the standalone metrics HTTP server` if `/metrics` isn't reachable and you expected it to be.

**`tenant_id` as a Prometheus label depends on deployment mode.** The underlying setting (`-Dpayos.metrics.tenant-tags-enabled`) is a PayOS-wide JVM flag, off by default for every component.
- **Standalone:** `StandaloneMain` sets it itself at startup (unless you've already set it yourself, in which case your own choice is respected). Safe to do unconditionally, since standalone TSS owns 100% of its own JVM — this can only ever affect TSS's own metrics.
- **Embedded:** `RuntimeIntegrationHook` deliberately does **not** touch this flag — in that mode TSS shares `payos-runtime`'s own JVM with the kernel's API metrics, connectors, and audit trail, and deciding that host's metrics-cardinality policy isn't TSS's call to make. `tenant_id` appears on TSS's metrics in embedded mode only if the host has already enabled the flag itself. If you run TSS embedded and want `tenant_id` on your TSS dashboards, set `-Dpayos.metrics.tenant-tags-enabled=true` on `payos-runtime`'s own JVM — the same setting then applies to every PayOS metric in that process, TSS's included.

The exact metric names as they actually appear in `/metrics` (not the logical names used elsewhere in this doc) are shown in the [developer guide's worked example](../developer/task-scheduler-guide-v20-2026-11-18.md#what-youll-actually-see-at-metrics).

## Clustering

Multiple TSS nodes sharing the same `TSS_SCHEDULER_INSTANCE_NAME` and pointed at the same TSS database form a Quartz cluster: only one node fires a given trigger at a time, and every node reconciles the shared scheduler against its own `jobs/` files at startup and on hot reload (see [Changing jobs without a restart](#changing-jobs-without-a-restart)). Reconciliation is idempotent, so nodes with identical files find nothing left to change after the first one. Point every node's `jobs/` root (standalone) or bundle (embedded) at the same content — a node with *different* files would reconcile the whole cluster to its own version (adding, changing or deleting jobs) when it starts or reloads. A shared volume, or one artifact deployed by the same pipeline, avoids this.

## Logging

The daemon uses slf4j with logback; no `logback.xml` ships by default. Job registration logs the number of jobs and tenants registered at startup; every reconciliation logs one summary line (`Job reconciliation (STARTUP|LIVE): added=[...], rescheduled=[...], ...`), and each deletion of an undeclared job its own `WARN`; each execution logs its `jobId`/`tenantId`/`scriptPath` on both success and failure paths.

**Sensitive-data masking — always on, nothing to configure.** Before a script error's message reaches `tss_job_execution.failure_message` or a log line, TSS masks sensitive-looking `key: value`/`key=value` fragments inside it (the same canonical denylist — `password`, `token`, `ssn`, `cardNumber`, etc. — every other PayOS component uses). This is deliberately best-effort, not a guarantee: a message with no recognizable key prefix at all (an unlabeled bare secret) passes through unmasked. Full behavior and worked examples: [developer guide §13](../developer/task-scheduler-guide-v20-2026-11-18.md#13-sensitive-data-masking).

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Standalone startup fails with `MappingNotFoundException: Mapping (RESOURCE) not found : mappings/<name>.hbm.xml` | A tenant or the platform database lists a mapping that is neither a file on disk nor a resource TSS can see. A mapping shipped in a connector (e.g. `mappings/PayosAuditEvent.hbm.xml` in `payos-database-audit-trail`) needs the PayOS configuration to declare that connector's directory as `service-adapters-dir`, as for the runtime (see [Running standalone on its own node](#running-standalone-on-its-own-node--avoiding-a-copy-of-the-tenant-configuration)). TSS versions built before 2026-11-06 did not load `service-adapters-dir` at all and failed this way on any connector-provided mapping, and on `mappings/PayosQueueOutbox.hbm.xml`. |
| The embedded TSS never starts, and no TSS line appears in the runtime log | The `task-scheduler` block is not part of the loaded PayOS configuration. The runtime reads only the `*.json` files directly in its configuration directory (`configDir`), not in its subdirectories: a `task-scheduler.json` placed in, say, `scheduler/` is ignored. Move the block to a file at the root of `configDir`, and check `task-scheduler.enabled` is `true`. |
| After a start-up exception (an unresolvable mapping, an invalid tenant, an unreachable database), the process does not exit | Known defect: Quartz's worker threads are created before job registration and are not stopped when it fails, so the JVM stays alive. Stop it (`Ctrl+C`), fix the cause, start again; a service manager should watch for the `TSS standalone scheduler started` log line rather than rely on the process exiting. |
| `list` shows no job, or `trigger` answers "unknown job", although jobs run | The command reaches another scheduler: a different TSS database, or the same database under a different scheduler name. See [Command line](#command-line) for running it against an embedded deployment. |
| Every pass of a reconciliation reports `failed=[platform/...]` with a `NullPointerException` | A defect of the job reconciliation introduced on 2026-11-03 and fixed on 2026-11-06 (`JobReconciler`): a platform job already in the scheduler could not be reconciled, so its cron, `parallelizable` or `arguments` changes were never applied, while it kept running on its previous definition. Rebuild or upgrade TSS (and `payos-runtime` for embedded mode). |

## Next

- [Class diagram](../architecture/job-scheduler-class-diagram-v24-2026-11-15.md) — every class, how they relate, and the full execution-flow walkthrough (registration through a fired trigger through script execution through cleanup).
- [ADR-0009](../architecture/adr/0009-task-scheduling-system.md) — the architectural decision behind TSS's design.
- [Configuration: Task Scheduling System](../configuration/task-scheduling-system-v11-2026-11-15.md) — the config-key reference only, for a quick lookup without the full narrative.
- [Developer guide: Task Scheduling System](../developer/task-scheduler-guide-v20-2026-11-18.md) — the full developer guide: configuring TSS and tenant jobs, writing script jobs, the `context`/`$DB`/`$PlatformDB`/`$Parallel` surface, and the same execution-history/misfire/quota model from a job script author's point of view.
- [notification-service.md](notification-service.md) — the other standalone-daemon precedent this guide's structure follows.
- [deployment.md](deployment.md) — `payos-runtime`'s own deployment model, relevant to TSS's embedded mode.
