Created: 2026-10-06
Last updated: 2026-10-07
Version: 4

# Task Scheduling System configuration

**Not a `bootstrap.json`/`PayOSConfig` block, unlike every other page in this directory.** `payos-job-scheduler` ("TSS") is its own standalone deployable with its own, independent configuration pipeline (`tss-config.json`, resolved by its own `StandaloneConfigLoader`) — it never reads `bootstrap.json` and its keys are not in `IConfigSpec`, they're in TSS's own `IJobConfigSpec`. This page exists so TSS's config surface is still discoverable from here; the full operational narrative (deployment modes, clustering, tenant/platform database resolution) lives in [operations/task-scheduling-system-v15-2026-10-07.md](../operations/task-scheduling-system-v15-2026-10-07.md).

Every key in the `database`/`scheduler`/`jobs` blocks below resolves across three tiers, in priority order: **environment variable** → **system property** (same name) → a **field in `tss-config.json`**, nested under the block named in that table. `jobs.json`/`custom-jobs.json`/`platform-jobs.json` are a separate, declarative document format (TSS spec §7.2) — not part of this three-tier resolution at all; they're read directly from disk by `JobConfigLoader`.

| Key | Default | Purpose |
| --- | --- | --- |
| `TSS_CONFIG_FILE` | `tss-config.json` | Locates the JSON config file itself, resolved relative to `--scheduler-base-dir`. Only the env var / system property tiers apply here — there's no third, file-based tier for the file's own location. |

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
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `maxConcurrentExecutionsPerTenant` | `0` (disabled) | **Global default** for the per-tenant concurrency quota (how many of one tenant's executions may be in progress at once, cluster-wide, across every node). `0` or negative means no tenant is ever rejected on this basis. A tenant can override this for itself via `jobs.json`'s own `maxConcurrentExecutions` field (below). Platform-level jobs are never subject to it — there's no tenant to meter. Exceeding it produces the `QUOTA_REJECTED` outcome — see [developer/task-scheduler-guide-v10-2026-10-07.md](../developer/task-scheduler-guide-v10-2026-10-07.md). |
| `TSS_METRICS_HTTP_PORT` | `metricsHttpPort` | `0` (disabled) | **Standalone mode only.** `0`/negative never starts anything — embedded mode never reads this key at all. A positive value starts a tiny Prometheus scrape HTTP server on that port, serving `GET /metrics` — see [developer/task-scheduler-guide-v10-2026-10-07.md §11](../developer/task-scheduler-guide-v10-2026-10-07.md#11-metrics) for why standalone mode needs this and embedded mode doesn't. |

## `jobs` block — file locations

Both values are resolved **relative to `--scheduler-base-dir`** (standalone mode) or the embedded host's bundle path, not the process's working directory, when given as relative paths.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_JOBS_ROOT_DIR` | `rootDir` | `jobs` | Directory `JobConfigLoader` walks — one subdirectory per tenant (each with its own `jobs.json`/`custom-jobs.json`), plus this directory's own `platform-jobs.json` for platform-level jobs. |
| `TSS_PAYOS_CONFIG_FILE` | `payosConfigFile` | `payos.json` | The PayOS config file TSS reads tenant/platform database settings from — standalone mode only (embedded mode reuses the host's already-loaded `PayOSConfig.settings` directly, no file re-read). |

## Full `tss-config.json` example

All three blocks above live in this one file — the JSON-file tier of the three-tier resolution. Any field shown here can instead come from an environment variable or system property, which takes priority over the file.

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
    "metricsHttpPort": 9404
  },
  "jobs": {
    "rootDir": "jobs",
    "payosConfigFile": "payos.json"
  }
}
```

`dbPassword` is deliberately omitted — set `TSS_DB_PASSWORD` as an environment variable instead. `rootDir`/`payosConfigFile` are shown here at their defaults; omit the whole `jobs` block if you don't need to override either.

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
| `parallelizable` | No (default `true`) | `false` schedules this job as `ExclusiveScriptJob` (`@DisallowConcurrentExecution`) — Quartz then guarantees this job never runs twice at once, cluster-wide. `true` (or omitted) schedules it as the plain, concurrency-unrestricted `ScriptJob`. |

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

Same per-entry schema as `jobs.json` above, with one addition: every entry **must** declare a `tenant` field matching the enclosing directory exactly — a mismatch, or an absent field, causes that entry to be rejected (logged, skipped). No top-level `maxConcurrentExecutions` field here — that's declared once, in `jobs.json` only.

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

## Cluster recovery

Every job is registered with Quartz's `requestRecovery(true)` — if a cluster node dies mid-execution, a surviving node re-fires that job as a new, separate execution. This is not independently configurable; it's always on for every job. The recovery execution also closes out its crashed predecessor's row (outcome `ABANDONED`) the moment it starts — otherwise that row would sit `completed_at IS NULL` forever and keep counting against the tenant's concurrency quota indefinitely. See [developer/task-scheduler-guide-v10-2026-10-07.md#10-multiple-attempts-for-the-same-trigger-firing](../developer/task-scheduler-guide-v10-2026-10-07.md#10-multiple-attempts-for-the-same-trigger-firing) for what you'll observe when it happens.

## Next

- [Operations: Task Scheduling System](../operations/task-scheduling-system-v15-2026-10-07.md) — deployment modes, clustering, starting/stopping, tenant/platform database resolution, the shared script pool, execution-history table reference and example queries.
- [Developer guide: Task Scheduling System](../developer/task-scheduler-guide-v10-2026-10-07.md) — the full developer guide: configuring TSS and tenant jobs, writing script jobs, the `context`/`$DB`/`$PlatformDB` surface, and execution outcomes & history.
- [Architecture: Task Scheduling System class diagram](../architecture/job-scheduler-class-diagram-v18-2026-10-07.md) — internal design (French).
