Created: 2026-10-06
Last updated: 2026-11-09
Version: 9

# Task Scheduling System configuration

**Not a `bootstrap.json`/`PayOSConfig` block in standalone mode, unlike every other page in this directory** (embedded mode reads a `task-scheduler` block of `payos.json` instead — see [Embedded mode](#embedded-mode--the-task-scheduler-block-of-payosjson)). `payos-job-scheduler` ("TSS") is its own standalone deployable with its own, independent configuration pipeline (`tss-config.json`, resolved by its own `StandaloneConfigLoader`) — it never reads `bootstrap.json` and its keys are not in `IConfigSpec`, they're in TSS's own `IJobConfigSpec`. This page exists so TSS's config surface is still discoverable from here; the full operational narrative (deployment modes, clustering, tenant/platform database resolution) lives in [operations/task-scheduling-system-v19-2026-11-09.md](../operations/task-scheduling-system-v19-2026-11-09.md).

Every key in the `database`/`scheduler`/`jobs`/`admin` blocks below resolves across three tiers, in priority order: **environment variable** → **system property** (same name) → a **field in `tss-config.json`**, nested under the block named in that table. `jobs.json`/`custom-jobs.json`/`platform-jobs.json` are a separate, declarative document format (TSS spec §7.2) — not part of this three-tier resolution at all; they're read directly from disk by `JobConfigLoader`.

| Key | Default | Purpose |
| --- | --- | --- |
| `TSS_CONFIG_FILE` | `tss-config.json` | Locates the JSON config file itself. An absolute path is used as is; a relative one, like the default, resolves against `--scheduler-base-dir` (builds before 2026-11-09 used the working directory, still accepted as a fallback with a `WARN`). A file that is not found is logged (`No TSS config file at '<path>'`) and every value must then come from the environment or system properties. Only the env var / system property tiers apply here — there's no third, file-based tier for the file's own location. |

## `database` block — TSS's own schema

TSS's own isolated schema (Quartz's `QRTZ_*` tables plus `tss_job_execution`) — never a tenant or platform database; see [database-service.md](database-service.md) for those.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_DB_URL` | `dbUrl` | *(required)* | JDBC URL. |
| `TSS_DB_USER` | `dbUser` | *(none)* | Username. |
| `TSS_DB_PASSWORD` | `dbPassword` | *(none)* | Password — set via the environment variable, not the file. Unlike the kernel's `bootstrap.json`, TSS's loader does not expand `${...}` placeholders, so a literal `dbPassword` in the file would be a literal secret on disk; the env var also wins priority either way. |
| `TSS_DB_DRIVER` | `dbDriver` | *(required)* | JDBC driver class, e.g. `org.postgresql.Driver`. |
| `TSS_DB_VENDOR` | `dbVendor` | *(generic ANSI SQL)* | `postgresql` or `oracle` — selects Quartz's `DriverDelegate`. Any other value (including unset) resolves to Quartz's generic `StdJDBCDelegate`. |
| `TSS_DB_SCHEMA` | `dbSchema` | *(none)* | Schema name, if your deployment uses a non-default one. |
| `TSS_DB_MAX_POOL_SIZE` | `dbMaxPoolSize` | `5` | Connection pool size for TSS's own pool. Also the pool `JobExecutionRepository`/`TenantConcurrencyQuotaGuard` reuse — see the class diagram, §13.4. |

## `scheduler` block — identity, sizing, and quota

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_SCHEDULER_INSTANCE_NAME` | `schedulerInstanceName` | `TSS-Cluster` | Every clustered node sharing one `jobs/` registry must use the **same** name. |
| `TSS_THREAD_COUNT` | `threadCount` | `5` | Quartz worker thread pool size — how many job executions can run concurrently on this one node. |
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `maxConcurrentExecutionsPerTenant` | `0` (disabled) | **Global default** for the per-tenant concurrency quota (how many of one tenant's executions may be in progress at once, cluster-wide, across every node). `0` or negative means no tenant is ever rejected on this basis. A tenant can override this for itself via `jobs.json`'s own `maxConcurrentExecutions` field (below). Platform-level jobs are never subject to it — there's no tenant to meter. Exceeding it produces the `QUOTA_REJECTED` outcome — see [developer/task-scheduler-guide-v16-2026-11-09.md](../developer/task-scheduler-guide-v16-2026-11-09.md). |
| `TSS_MAX_PARALLEL_WORKERS_PER_JOB` | `maxParallelWorkersPerJob` | `4` | Ceiling on how many worker threads one `$Parallel.forEach` call (tenant-scoped job scripts only) may use to fan a single execution's own batch out. A different axis from the quota above: that one bounds how many whole executions run at once; this one bounds how much *one* execution can fan out internally. See [developer/task-scheduler-guide-v16-2026-11-09.md §4](../developer/task-scheduler-guide-v16-2026-11-09.md#4-parallel--processing-a-batch-concurrently). Since workers can also read the (read-only) `$PlatformDB`, up to `threadCount × maxParallelWorkersPerJob` platform reads may be in flight at once — size `platform-database-service.configuration.max-pool-size` (HikariCP default `10`) accordingly. |
| `TSS_METRICS_HTTP_PORT` | `metricsHttpPort` | `0` (disabled) | **Standalone mode only.** `0`/negative never starts anything — embedded mode never reads this key at all. A positive value starts a tiny Prometheus scrape HTTP server on that port, serving `GET /metrics` — see [developer/task-scheduler-guide-v16-2026-11-09.md §12](../developer/task-scheduler-guide-v16-2026-11-09.md#12-metrics) for why standalone mode needs this and embedded mode doesn't. |
| `TSS_HOT_RELOAD_ENABLED` | `hotReloadEnabled` | `true` | Watches `jobs/` (with the same `ConfigWatcher` PayOS uses for its own configuration) and, about 2 s after the last change, reconciles the scheduler against the edited files — new jobs, changed cron expressions, `parallelizable` and `arguments`, removed jobs — without a restart. `false` means changes are applied only at startup or by the `sync` command. Embedded mode: `task-scheduler.scheduler.hotReloadEnabled`. See [operations §Changing jobs without a restart](../operations/task-scheduling-system-v19-2026-11-09.md#changing-jobs-without-a-restart). |

## `jobs` block — file locations

Both values are resolved **relative to `--scheduler-base-dir`** (standalone mode) or the embedded host's bundle path, not the process's working directory, when given as relative paths.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_JOBS_ROOT_DIR` | `rootDir` | `jobs` | Directory `JobConfigLoader` walks — one subdirectory per tenant (each with its own `jobs.json`/`custom-jobs.json`), plus this directory's own `platform-jobs.json` for platform-level jobs. |
| `TSS_PAYOS_CONFIG_FILE` | `payosConfigFile` | `payos.json` | The PayOS config file TSS reads tenant/platform database settings from — standalone mode only (embedded mode reuses the host's already-loaded `PayOSConfig.settings` directly, no file re-read). After merging that file's `configDir`, standalone TSS also runs every SPI-discovered PayOS config loader (`IConfigLoader`) the same way the runtime does — e.g. `tenant-config-loader-database`, bundled in the standalone jar and inactive unless its `tenant-config-loader` block is enabled — so tenants declared in a database registry table are seen without copying their JSON declarations. It also loads the JARs of that configuration's `service-adapters-dir` (relative to the PayOS config file's directory; `PAYOS_SERVICE_ADAPTERS_DIR` takes priority), as the runtime does: their config loaders and database factories are used, and Hibernate mapping files shipped in them as classpath resources resolve when a tenant or the platform database lists them. See [operations §Tenant and platform database resolution](../operations/task-scheduling-system-v19-2026-11-09.md#tenant-and-platform-database-resolution). |

## `admin` block — administration endpoint

The secured HTTP endpoint for manual triggering, on-demand reconciliation and job listing. Off unless a port is set. Callers authenticate with an OIDC Bearer token verified by PayOS's own security service (the `security` configuration of `payos.json`, or of the host runtime in embedded mode); the roles below decide what they may do. Embedded mode reads the same fields from `task-scheduler.admin`.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_ADMIN_HTTP_PORT` | `httpPort` | `0` (disabled) | Port of the administration endpoint (`/tss/admin/...`), its own listener in both modes so it can be firewalled separately. `0` or negative never starts it. |
| `TSS_ADMIN_PLATFORM_ROLE` | `platformRole` | `tss-platform-admin` | Token role granting every operation, on every tenant and on platform jobs, including `sync`. |
| `TSS_ADMIN_TENANT_ROLE` | `tenantRole` | `tss-tenant-admin` | Token role granting `trigger` and `list` on the caller's own tenant only — the tenant carried by the token itself, derived from its issuer realm. |

See [operations §Administration: manual trigger, sync and list](../operations/task-scheduling-system-v19-2026-11-09.md#administration-manual-trigger-sync-and-list) for the endpoints, the command-line equivalents and the audit trail.

## Full `tss-config.json` example

All four blocks above live in this one file — the JSON-file tier of the three-tier resolution. Any field shown here can instead come from an environment variable or system property, which takes priority over the file.

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
    "threadCount": 10,
    "maxConcurrentExecutionsPerTenant": 20,
    "maxParallelWorkersPerJob": 4,
    "metricsHttpPort": 9404,
    "hotReloadEnabled": true
  },
  "jobs": {
    "rootDir": "jobs",
    "payosConfigFile": "payos.json"
  },
  "admin": {
    "httpPort": 8095,
    "platformRole": "tss-platform-admin",
    "tenantRole": "tss-tenant-admin"
  }
}
```

`dbPassword` is deliberately omitted — set `TSS_DB_PASSWORD` as an environment variable instead. `rootDir`/`payosConfigFile` are shown here at their defaults; omit the whole `jobs` block if you don't need to override either.

## Embedded mode — the `task-scheduler` block of `payos.json`

When TSS runs inside `payos-runtime` (`TaskSchedulerRuntimeIntegration`), it does not read `tss-config.json` or the `TSS_*` environment variables. It reads one block of the host's own PayOS configuration, `task-scheduler`, with the same JSON field names as the standalone blocks above. Because this block is part of the PayOS configuration, `${...}` placeholders are expanded by the PayOS loader (see [env-var-resolution.md](env-var-resolution.md)), so the password can come from the environment.

**Where to put the block.** Like any PayOS setting, the block must sit in `payos.json` or in a `*.json` file **directly** in the runtime's configuration directory (`configDir`). The runtime does not read subdirectories, so a `task-scheduler.json` placed in, say, `scheduler/` is silently ignored and TSS does not start, with no TSS line in the log.

| Field | Default | Purpose |
| --- | --- | --- |
| `task-scheduler.enabled` | `false` | Must be `true` for TSS to start inside the runtime; anything else skips it. |
| `task-scheduler.database.*` | — | `dbUrl` and `dbDriver` are required; `dbUser`, `dbPassword`, `dbVendor`, `dbSchema`, `dbMaxPoolSize` as in the `database` block above. |
| `task-scheduler.scheduler.*` | — | `schedulerInstanceName`, `threadCount`, `maxConcurrentExecutionsPerTenant`, `maxParallelWorkersPerJob`, `hotReloadEnabled`, with the same defaults as the `scheduler` block. `metricsHttpPort` is not read: metrics go through the runtime's own `/metrics`. |
| `task-scheduler.admin.*` | — | `httpPort`, `platformRole`, `tenantRole`, as in the `admin` block. The endpoint uses the host's `security` configuration. |

There is no `jobs` block in embedded mode: the jobs root is always `jobs/` under the runtime's base directory (`--bundle-path`), and tenant and platform databases come from the host's already-loaded configuration.

```json
{
  "task-scheduler": {
    "enabled": true,
    "database": {
      "dbUrl": "jdbc:postgresql://db.internal:5432/tss",
      "dbUser": "tss_svc",
      "dbPassword": "${TSS_DB_PASSWORD}",
      "dbDriver": "org.postgresql.Driver",
      "dbVendor": "postgresql"
    },
    "scheduler": {
      "schedulerInstanceName": "TSS-Cluster",
      "threadCount": 5,
      "hotReloadEnabled": true
    },
    "admin": {
      "httpPort": 8095
    }
  }
}
```

## `jobs/<tenantId>/jobs.json` — per-tenant jobs and policy

One file per tenant directory under the jobs root. Besides the top-level `maxConcurrentExecutions` policy field, it declares an array of job entries. Only `implementation.type: "script"` + `trigger.type: "cron"` entries are loaded today (other combinations are Growth-phase and are skipped, not rejected).

**Top-level fields:**

| Field | Default | Purpose |
| --- | --- | --- |
| `maxConcurrentExecutions` | *(no override)* | This tenant's own override of the global `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` default. Absent, or `0`/negative, means "use the global default." Declared once here — **never** in `custom-jobs.json`, which is for individual job overrides, not tenant-level policy. |
| `jobs` | *(required array)* | This tenant's job entries, below. |

**Fields of one entry in the `jobs` array:**

| Field | Required | Purpose |
| --- | --- | --- |
| `id` | Yes | The job's declared identifier — used in logs only. The job's actual unique key is `(tenantId, scriptPath)`, not `id`; several tenants (or a tenant and the platform) can declare the same `id`. |
| `implementation.type` | Yes | Must be `"script"` — the only value loaded today. |
| `implementation.script` | Yes | The script path, resolved at execution time against the **shared script pool** (one flat directory under the scheduler base dir, keyed only by this path — not nested per tenant). |
| `implementation.mappingFiles` | No | Array of Hibernate mapping files this job's script needs on its tenant's `$DB` session. A tenant's effective mapping-file set is the **union** across all of that tenant's jobs. |
| `trigger.type` | Yes | Must be `"cron"` — the only value loaded today. |
| `trigger.expression` | Yes | A standard Quartz cron expression. |
| `arguments` | No | Free-form JSON object handed to the script as `context.getArguments()`. Absent means an empty object; a non-object value is logged and ignored. Stored with the job in Quartz and kept up to date by reconciliation, so an edit reaches every cluster node through hot reload or `sync`, without a restart. `${...}` placeholders are **not** expanded — keep secrets out. See [developer guide §6.1](../developer/task-scheduler-guide-v16-2026-11-09.md#61-passing-arguments-to-a-script-arguments). |
| `parallelizable` | No (default `true`) | `false` schedules this job as `ExclusiveScriptJob` (`@DisallowConcurrentExecution`) — Quartz then guarantees this job (`(tenant, script)`) never runs twice at once, cluster-wide: a trigger firing during a running execution waits for it, and if it waits beyond Quartz's 60 s misfire threshold it is recorded as `MISFIRE` and fired once to catch up. `true` (or omitted) schedules it as the plain, concurrency-unrestricted `ScriptJob`. |

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
    },
    {
      "id": "cleanup-hourly",
      "implementation": {
        "type": "script",
        "script": "jobs/cleanup.js"
      },
      "trigger": {
        "type": "cron",
        "expression": "0 0 * * * ?"
      },
      "parallelizable": true
    }
  ]
}
```

## `jobs/<tenantId>/custom-jobs.json` — tenant-scoped overrides and additions

Same per-entry schema as `jobs.json` above, with one addition: every entry **must** declare a `tenant` field matching the enclosing directory exactly — a mismatch, or an absent field, causes that entry to be rejected (logged, skipped). No top-level `maxConcurrentExecutions` field here — that's declared once, in `jobs.json` only. A matching entry replaces the `jobs.json` entry as a whole, `arguments` included (no key-by-key merge).

Loaded after `jobs.json` and merged into the same per-tenant job map: a matching `scriptPath` is replaced; a new one is added.

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

## `jobs/platform-jobs.json` — platform-level jobs

A single file at the `jobs/` root — not nested in any tenant directory. Same per-entry schema as `jobs.json` (no `tenant` field, and no top-level `maxConcurrentExecutions` — platform jobs are never subject to the per-tenant quota at all).

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

A platform-level job's script gets **no `$DB`** (no tenant to open a session for) and a **read+write** `$PlatformDB`; a tenant job's `$PlatformDB` is **read-only**. See the operations doc's own section on this for the full explanation.

## `jobs/template.json` — model for new tenants

Optional, at the jobs root, same schema as a tenant's `jobs.json`. At startup, on hot reload and on `sync`, TSS creates `jobs/<tenantId>/` for every tenant declared under `multitenancy.tenants` in the PayOS configuration (JSON files and config loaders, e.g. a database tenant registry; in embedded mode the host's live configuration) that has no directory yet, and writes this file into it as `jobs.json`, personalized for that tenant: every `{{tenantId}}` is replaced by the tenant's id, wherever it appears in a string. Without `template.json`, the directory is created empty.

```json
{
  "jobs": [
    {
      "id": "{{tenantId}}-daily-export",
      "implementation": { "type": "script", "script": "jobs/daily-export.js" },
      "trigger": { "type": "cron", "expression": "0 0 2 * * ?" },
      "arguments": { "folder": "/exports/{{tenantId}}", "tenant": "{{tenantId}}" }
    }
  ]
}
```

`{{tenantId}}` is the only placeholder; any other `{{...}}` is left as written, with a warning in the log. It is deliberately not the `${...}` syntax of PayOS's configuration files, which `jobs.json` never expands.

- Only ever creates: an existing tenant directory or file is never modified, and a later edit of `template.json` does not reach tenants already provisioned.
- A `template.json` that is not a valid JSON object is never copied (an error is logged, the directory is created empty), so it cannot break the loading of the jobs files.
- Nothing is created when the jobs root itself is missing, a tenant id is not a plain directory name (letters, digits, `.`, `_`, `-`), or a tenant is named `platform`.
- Apart from `{{tenantId}}`, the template's text is kept exactly as written. A new tenant's jobs are scheduled at the next node restart (see [operations §Safety rules](../operations/task-scheduling-system-v19-2026-11-09.md#safety-rules)).

## Cluster recovery

Every job is registered with Quartz's `requestRecovery(true)` — if a cluster node dies mid-execution, a surviving node re-fires that job as a new, separate execution. This is not independently configurable; it's always on for every job. The recovery execution also closes out its crashed predecessor's row (outcome `ABANDONED`) the moment it starts — otherwise that row would sit `completed_at IS NULL` forever and keep counting against the tenant's concurrency quota indefinitely. See [developer/task-scheduler-guide-v16-2026-11-09.md#11-multiple-attempts-for-the-same-trigger-firing](../developer/task-scheduler-guide-v16-2026-11-09.md#11-multiple-attempts-for-the-same-trigger-firing) for what you'll observe when it happens.

## Next

- [Operations: Task Scheduling System](../operations/task-scheduling-system-v19-2026-11-09.md) — deployment modes, clustering, starting/stopping, tenant/platform database resolution, the shared script pool, execution-history table reference and example queries.
- [Developer guide: Task Scheduling System](../developer/task-scheduler-guide-v16-2026-11-09.md) — the full developer guide: configuring TSS and tenant jobs, writing script jobs, the `context`/`$DB`/`$PlatformDB`/`$Parallel` surface, and execution outcomes & history.
- [Architecture: Task Scheduling System class diagram](../architecture/job-scheduler-class-diagram-v20-2026-11-09.md) — internal design (French).
