Created: 2026-10-06
Last updated: 2026-10-08
Version: 12

# Task Scheduling System: developer guide

A self-contained, start-to-finish guide to writing and configuring job scripts for the Task Scheduling System (`payos-job-scheduler`, "TSS") — a Quartz-backed, clustered scheduler that fires your script on a cron schedule, separate from the HTTP/TCP/queue request path the rest of this guide otherwise covers. This page covers everything you need as a developer: how to configure TSS itself, where your job's own configuration lives, every field your script's `execute(context)` function can use, a realistic batch-job example, and what happens to an execution that your script never even gets to run. The condensed config-key reference (for a quick lookup once you already know this material) is [configuration/task-scheduling-system-v4-2026-10-07.md](../configuration/task-scheduling-system-v4-2026-10-07.md); the full day-2 operational picture (clustering, deployment modes, starting/stopping) is [operations/task-scheduling-system-v15-2026-10-07.md](../operations/task-scheduling-system-v15-2026-10-07.md).

## 1. Configuring TSS itself (`tss-config.json`)

TSS is its own standalone deployable — it does **not** read `bootstrap.json` or go through `PayOSConfig`. Its own config file (default `tss-config.json`, overridable via the `TSS_CONFIG_FILE` environment variable or system property) is resolved relative to `--scheduler-base-dir` (the directory you pass when launching `payos-job-scheduler`'s own jar; defaults to the process's working directory).

Every key below resolves across three tiers, in priority order: **environment variable** → **system property** (same name) → **a field in `tss-config.json`**, nested under the block named below. The first tier with a non-blank value wins — so you can override any single field with an env var without touching the file at all, which is exactly how you'd keep `TSS_DB_PASSWORD` out of the file entirely (see below).

### `database` block

TSS's own isolated schema (Quartz's own `QRTZ_*` tables, plus `tss_job_execution` — the history table this guide's §8 covers). **Never** a tenant or platform database — those are resolved separately (§2).

| Key | JSON field | Default | Effect on scheduler behavior |
| --- | --- | --- | --- |
| `TSS_DB_URL` | `dbUrl` | *(required)* | Where TSS's own schema lives. Every clustered node must point at the **same** database — this is literally how Quartz clustering works (nodes coordinate through shared rows in this database, not through talking to each other directly). |
| `TSS_DB_USER` | `dbUser` | *(none)* | Username for the connection above. |
| `TSS_DB_PASSWORD` | `dbPassword` | *(none)* | Password. **Always set this as the environment variable, never in the file** — TSS's config loader does not expand `${...}` placeholders the way the kernel's `bootstrap.json` does, so a literal `dbPassword` in the file is a literal plaintext secret on disk. |
| `TSS_DB_DRIVER` | `dbDriver` | *(required)* | JDBC driver class, e.g. `org.postgresql.Driver`. |
| `TSS_DB_VENDOR` | `dbVendor` | *(generic ANSI SQL)* | `postgresql` or `oracle` — selects which Quartz `DriverDelegate` is used for SQL dialect differences. Any other value (including leaving it unset) falls back to Quartz's generic `StdJDBCDelegate`, which works for H2/most ANSI-SQL databases but may not handle vendor-specific locking semantics optimally. |
| `TSS_DB_SCHEMA` | `dbSchema` | *(none)* | Schema name, if your database uses a non-default one. Affects the table-name prefix Quartz uses internally for every `QRTZ_*` table lookup. |
| `TSS_DB_MAX_POOL_SIZE` | `dbMaxPoolSize` | `5` | Size of TSS's own connection pool. This is the **same** pool `JobExecutionRepository` (the history-table DAO) and the per-tenant quota guard both reuse — it is not only for Quartz's own internal locking traffic. Too small a pool under heavy concurrent job load can itself become a bottleneck independent of `threadCount` below. |

### `scheduler` block

| Key | JSON field | Default | Effect on scheduler behavior |
| --- | --- | --- | --- |
| `TSS_SCHEDULER_INSTANCE_NAME` | `schedulerInstanceName` | `TSS-Cluster` | Every node that should cluster together (share job registration, fail over for each other) **must** use the identical value. A node with a different `instanceName` is a completely separate, independent scheduler — even if it points at the exact same database, it will not coordinate with the others at all. This is the single most common cause of "I have 3 nodes but they're not clustering" misconfigurations. |
| `TSS_THREAD_COUNT` | `threadCount` | `5` | How many job executions this **one node** can run truly concurrently. If more triggers fire than there are free threads, Quartz queues the extra ones; if a queued trigger waits long enough that its scheduled fire time falls outside Quartz's misfire threshold, **it becomes a genuine misfire** (see §9) even though nothing actually failed — the node was just busy. If your jobs routinely overlap or you see unexpected `MISFIRE` rows, raising this (or adding nodes) is usually the fix, not adding more quota headroom. |
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `maxConcurrentExecutionsPerTenant` | `0` (disabled) | Global default for the per-tenant concurrency quota — §7 covers this in full. |
| `TSS_METRICS_HTTP_PORT` | `metricsHttpPort` | `0` (disabled) | **Standalone mode only** — see §11 for why. A positive value starts a tiny Prometheus scrape HTTP server on that port. |

### Full `tss-config.json` example

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
    "maxConcurrentExecutionsPerTenant": 20
  },
  "jobs": {
    "rootDir": "jobs",
    "payosConfigFile": "payos.json"
  }
}
```

`dbPassword` is deliberately absent — set `TSS_DB_PASSWORD` as an environment variable at launch instead.

### `jobs` block — where your job files live

| Key | JSON field | Default | Effect |
| --- | --- | --- | --- |
| `TSS_JOBS_ROOT_DIR` | `rootDir` | `jobs` | The directory TSS walks for your job definitions — §2 covers its exact layout. Resolved relative to `--scheduler-base-dir`. |
| `TSS_PAYOS_CONFIG_FILE` | `payosConfigFile` | `payos.json` | The PayOS config file TSS reads your tenant's database credentials from — **standalone mode only**. In embedded mode (TSS running inside `payos-runtime`), this is never read; TSS reuses the host's already-loaded tenant configuration directly, so this key has no effect there at all. |

### Running TSS locally to try your script

```bash
cd payos-job-scheduler
mvn clean package
java -jar target/payos-job-scheduler-<version>.jar --scheduler-base-dir /path/to/your/test/bundle
```

`--scheduler-base-dir` is the one directory everything else (`tss-config.json`, `jobs/`, your script files, the PayOS config file) is resolved relative to. For local testing, H2 works fine as `TSS_DB_VENDOR` (or just omit it).

## 2. Configuring jobs for a tenant — file conventions

Every job TSS can fire is declared in one of three file types, all under `TSS_JOBS_ROOT_DIR` (default `jobs/`):

```
jobs/
├── platform-jobs.json              ← platform-level jobs (no tenant owner)
├── acme-corp/
│   ├── jobs.json                   ← acme-corp's own jobs, editor-provided baseline
│   └── custom-jobs.json            ← acme-corp's own overrides/additions (optional)
└── beta-inc/
    └── jobs.json
```

- **One subdirectory per tenant**, named exactly as the tenant id. TSS walks every immediate subdirectory of the jobs root and treats each as a tenant — a directory literally named `platform` is rejected (that name is reserved).
- **`jobs.json`** — a tenant's own baseline job set. This is where you register a new job for a tenant in the normal case.
- **`custom-jobs.json`** (optional, same directory) — for overriding or adding to what `jobs.json` already declares, without editing that file directly. Loaded *after* `jobs.json`; a matching `scriptPath` is replaced, a new one is added. Every entry here must declare a `tenant` field matching the directory exactly, or it's rejected.
- **`platform-jobs.json`** (optional, at the jobs root, not nested in any tenant directory) — jobs with no tenant owner at all. Only `platform-jobs.json` jobs get the read+write `$PlatformDB`; every tenant job's `$PlatformDB` is read-only (§3).

Only `implementation.type: "script"` + `trigger.type: "cron"` entries are loaded today; other combinations are silently skipped (reserved for a future phase), not rejected.

### `jobs.json` — full schema

| Field | Required | Effect |
| --- | --- | --- |
| `maxConcurrentExecutions` | No | Top-level, sibling to `jobs` — this tenant's own override of the global concurrency quota. §7 covers this in full. |
| `jobs[].id` | Yes | Log-display identifier only. The job's real unique key is `(tenantId, scriptPath)`, not `id` — two different tenants (or a tenant and the platform) can reuse the same `id` with no conflict. |
| `jobs[].implementation.type` | Yes | Must be `"script"`. |
| `jobs[].implementation.script` | Yes | The script path. Resolved at execution time against a **shared, flat script pool** under the scheduler base directory — not nested per tenant. Two tenants can declare the exact same `script` path; they read the identical file, scheduled independently, each against their own `$DB`. |
| `jobs[].implementation.mappingFiles` | No | Array of Hibernate mapping files this job's script needs on `$DB`. A tenant's *effective* mapping-file set is the union across every one of that tenant's jobs — declare only what this specific job touches, you don't need to repeat mappings another of the tenant's jobs already declared. |
| `jobs[].trigger.type` | Yes | Must be `"cron"`. |
| `jobs[].trigger.expression` | Yes | A standard Quartz cron expression (`"0 0 2 * * ?"` = every day at 02:00). |
| `jobs[].parallelizable` | No (default `true`) | **A completely different concurrency control from the quota in §7 — don't conflate them.** `false` makes Quartz guarantee this *one specific job* (`(tenantId, scriptPath)`) never runs twice at once, cluster-wide: if a second trigger fires while the first execution is still running, Quartz makes it **wait** until the first finishes, then runs it — it is never rejected, never skipped, just delayed. (Internally: `false` schedules the job as `ExclusiveScriptJob`, annotated `@DisallowConcurrentExecution`; `true`/omitted uses the plain `ScriptJob`.) The quota in §7, by contrast, is a per-**tenant** cap across *all* that tenant's jobs together, and a rejection under quota is an outright `QUOTA_REJECTED` — never queued, never retried automatically. |

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
      "trigger": { "type": "cron", "expression": "0 0 2 * * ?" },
      "parallelizable": false
    },
    {
      "id": "cleanup-hourly",
      "implementation": { "type": "script", "script": "jobs/cleanup.js" },
      "trigger": { "type": "cron", "expression": "0 0 * * * ?" },
      "parallelizable": true
    }
  ]
}
```

### `custom-jobs.json` — same schema, plus a required `tenant` field, no `maxConcurrentExecutions`

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
      "trigger": { "type": "cron", "expression": "0 30 2 * * ?" },
      "parallelizable": false
    }
  ]
}
```

### `platform-jobs.json` — same schema, no `tenant` field, no `maxConcurrentExecutions`

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
      "trigger": { "type": "cron", "expression": "0 0 3 * * ?" },
      "parallelizable": false
    }
  ]
}
```

## 3. `$DB` and `$PlatformDB` — the full method surface

Both are bound as globals, exactly like an API script — but which one(s) are bound, and with which methods, depends on whether your job is tenant-scoped or platform-level.

### `$DB` (tenant-scoped jobs only — unbound entirely for a platform-level job)

| Method | Effect |
| --- | --- |
| `newParams()` | A fresh, empty param map to populate before `save`/entity operations. |
| `get(entityName, id)` / `getFresh(entityName, id)` | Load one entity by id; `getFresh` bypasses any session-level cache. |
| `findAll(entityName)` | Every row of an entity type (tenant-filtered automatically). |
| `find(query, params)` / `list(query, params[, maxResults][, firstResult, maxResults])` | Query execution, with or without pagination. |
| `unique(query, params)` / `first(query, params)` | Exactly-one-result / first-result query helpers. |
| `save(entityName, entity)` / `update(entityName, entity)` / `delete(entityName, entity)` / `deleteById(entityName, id)` | Writes. |
| `executeUpdate(query, params)` | Bulk update/delete by query. |
| `markRollbackOnly()` | Flags the current transaction to roll back, without throwing. |
| `beginTransaction()` / `commitTransaction()` / `rollbackTransaction()` / `setAutoCommit(boolean)` / `isAutoCommit()` | **Manual transaction control — available to job scripts, unlike API scripts.** An API script's transaction can be shared across several nested calls on the same thread, so none of them are allowed to end it early; a job execution owns its own session outright, with nothing else sharing it, so this restriction doesn't apply. This is exactly what you need for the batch pattern in §4. |

### `$PlatformDB`

- **Tenant-scoped job:** **read-only** — `newParams`/`get`/`getFresh`/`findAll`/`find`/`list`/`unique`/`first` only. No `save`/`update`/`delete`/`deleteById`/`executeUpdate` — not just blocked at runtime, the methods don't exist on the binding at all. You're reading cross-tenant reference data, outside your own tenant's scope.
- **Platform-level job:** **read + write** — everything above, plus `save`/`update`/`delete`/`deleteById`/`executeUpdate`. No transaction-control methods, though (no `beginTransaction`/etc.) — each write auto-commits the moment it returns. **No cross-write atomicity**: several writes in sequence get no rollback across them if the script later throws — a partial sequence of commits stays committed. If you need all-or-nothing across multiple platform writes, use idempotent upserts or explicit cleanup in a `catch`, not an assumed rollback.

`$DB`/`$PlatformDB` are simply absent (`typeof $DB === 'undefined'`) rather than bound-but-empty when not applicable — check with `typeof` if a script might run in either context, though in practice a given script is written for one or the other, not both.

### `$Logger`

Always bound, for every job, tenant-scoped or platform-level alike. Same `info(message)`/`warn(message)`/`error(message)`/`debug(message)` surface $Logger already follows everywhere else in PayOS (API scripts via `ApiResourceHandler`, hook scripts via `HookEngine`), each with the `{}`-placeholder overload for one or more arguments (e.g. `$Logger.warn("skipped {} of {} rows", skipped, total)`) — but unlike those, TSS's `$Logger` is **not** a bare SLF4J `Logger`: it's wrapped (`JobLoggerBinding`) specifically so every line a tenant-scoped job logs is automatically prefixed with `[tenant:<id>] `, with no need to interpolate `context.getTenantId()` yourself just to make a line traceable back to one tenant. A platform-level job (no owning tenant) gets lines unprefixed, exactly like the bare-`Logger` behavior this replaced. Lines still land in TSS's own process log under the executing class's logger name — there's no per-job or per-tenant log file, and `jobId`/`fireInstanceId` are still not auto-tagged; include those yourself if a line needs to be traced back to one specific execution, not just one tenant.

```javascript
function execute(context) {
    $Logger.info("starting reconciliation"); // -> "[tenant:acme-corp] starting reconciliation"
}
```

## 4. A realistic batch job

The pattern that actually matters for TSS: a job processing an entire table, with intermediate commits so a crash partway through doesn't lose all progress (and doesn't hold one open transaction for the job's whole, possibly long, runtime).

```javascript
// jobs/reconciliation.js
function execute(context) {
    var BATCH_SIZE = 100;
    var processed = 0;

    while (true) {
        var batch = $DB.list(
            "FROM Invoice WHERE status = :status",
            { status: 'pending' },
            BATCH_SIZE
        );
        if (batch.length === 0) {
            break;
        }

        $DB.beginTransaction();
        try {
            for (var i = 0; i < batch.length; i++) {
                var invoice = batch[i];
                invoice.put('status', 'reconciled');
                invoice.put('reconciledAt', new Date().toISOString());
                $DB.update('Invoice', invoice);
            }
            $DB.commitTransaction();
        } catch (e) {
            $DB.rollbackTransaction();
            throw e; // one bad batch fails the whole run — SCRIPT_ERROR, see §6
        }

        processed += batch.length;
    }

    context.getPlatformDb(); // e.g. record a shared "last reconciliation run" timestamp, if needed
}
```

Each batch is its own transaction — a crash after batch 3 of 10 leaves the first 30 rows genuinely committed as `reconciled`, not rolled back, and the next scheduled run picks up where this one left off (since the query only ever selects `status = 'pending'` rows). This is why `beginTransaction`/`commitTransaction` are available to job scripts specifically (§3) — a single implicit transaction for the whole run, as an ordinary API script gets, would hold a lock for the entire batch's duration and lose all progress on any mid-run failure.

## 5. The `context` argument — every field

`context` (TSS's own `JobContext` — not the `request`/`controlData` pair an API script's `execute` gets) exposes:

| Field (Java accessor / JS property) | Type | Value |
| --- | --- | --- |
| `getJobId()` | string | The job's declared `id` from `jobs.json` (log-display only — see §2's note on why it's not a unique key). |
| `getTenantId()` | string or `null` | The owning tenant, or `null` for a platform-level job. |
| `getScriptPath()` | string | This job's `implementation.script` value. |
| `getFiredAt()` | `Instant`/Date | When this execution actually started (not the originally-scheduled time — see §9 for why those can differ after a misfire catch-up). |
| `getDb()` | `$DB` binding, or `null` | `null` for a platform-level job. See §3. |
| `getPlatformDb()` | `$PlatformDB` binding, or `null` | `null` only if no platform database is configured at all for this deployment. See §3. |

In JavaScript, GraalVM exposes Java getters as plain properties too — `context.jobId`, `context.tenantId` etc. work the same as `context.getJobId()`.

## 6. The five outcomes

Every execution TSS records lands on exactly one of five outcomes:

| Outcome | Your `execute` function ran | Meaning |
| --- | --- | --- |
| `SUCCESS` | Yes | Completed without throwing. |
| `SCRIPT_ERROR` | Yes | Threw, or the engine couldn't find/execute an `execute` function at all. |
| `MISFIRE` | **No** | Quartz detected your trigger's scheduled fire time passed without ever actually starting the job — see §1's `threadCount` note and §9 below. |
| `QUOTA_REJECTED` | **No** | Your tenant already had too many of this job's executions in progress at once — see §7. |
| `ABANDONED` | Yes (but never finished) | Your job's owning node crashed mid-execution; a surviving node's recovery closed out the row on its behalf — see §10. |

The `MISFIRE`/`QUOTA_REJECTED` "no" cases matter for how you debug a job that silently "didn't run": there's nothing to catch in your script, nothing in your script's own logs — the rejection happened entirely before your code was ever reached, the same way a tenant's HTTP request can be rejected by `TenantPolicyService` before `execute` runs (see [Tenant quota enforcement](tenant-quota-enforcement.md), a genuinely different mechanism for a different kind of request — §7 explains the distinction fully).

## 7. Concurrency quota (`maxConcurrentExecutions`)

An operator can cap how many of your tenant's job executions may be in progress **at once**, cluster-wide — a concurrency cap, not a rate limit (nothing resets per minute; it's purely "how many right now"). If your tenant is at that cap when a new trigger fires, that execution is rejected outright (`QUOTA_REJECTED`) rather than queued or delayed.

- The operator sets a global default (`TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT`, §1) applied to every tenant, and/or your own tenant's specific override (`maxConcurrentExecutions` in your `jobs.json`, §2) if your workload genuinely needs a different number than everyone else.
- A platform-level job is never subject to this — there's no tenant to meter.
- This counts **all** of your tenant's in-progress executions together, across every job — it's a per-tenant cap, not a per-job cap. (`parallelizable: false`, §2, is the per-*job* control — the two are independent and often used together, not alternatives to each other.)

Your own override lives at the top of your tenant's `jobs.json`:

```json
{
  "maxConcurrentExecutions": 2,
  "jobs": [
    {
      "id": "reconciliation-daily",
      "implementation": { "type": "script", "script": "jobs/reconciliation.js" },
      "trigger": { "type": "cron", "expression": "0 0 2 * * ?" },
      "parallelizable": false
    }
  ]
}
```

With the override above, if this tenant already has 2 executions in progress anywhere (this job or any other of theirs), a 3rd trigger firing — for this job or any other — is rejected with `QUOTA_REJECTED`, regardless of what the global default is set to.

### Not the same quota as HTTP requests

This is a completely separate mechanism from [tenant quota enforcement](tenant-quota-enforcement.md) (the `requestsPerMinute` limit on your tenant's HTTP/TCP requests, enforced by `TenantPolicyService`/`ISlidingWindowCounter`). Different counter, different config surface (`jobs.json`'s `maxConcurrentExecutions` vs. `multitenancy.tenants[].quota.requestsPerMinute`), different semantics (concurrency cap vs. rate limit). A tenant can be well within its HTTP request quota and still have a job rejected for `QUOTA_REJECTED`, or vice versa — they don't interact at all.

## 8. Querying your job's history

TSS has no network API — the database is the query interface. Every execution of your job — successful, failed, misfired, quota-rejected, or abandoned by a crashed node — is one row in `tss_job_execution`, keyed by `(tenant_id, script_path)`, the same identity Quartz's own scheduling already uses for your job:

```sql
SELECT fire_instance_id, node, started_at, completed_at, outcome, failure_message
FROM tss_job_execution
WHERE tenant_id = 'acme-corp' AND script_path = 'jobs/reconciliation.js'
ORDER BY started_at DESC
LIMIT 5;
```

A typical result, newest first:

| `fire_instance_id` | `node` | `started_at` | `completed_at` | `outcome` | `failure_message` |
| --- | --- | --- | --- | --- | --- |
| `c3f1...e2` | `node-b-1` | 09:02:00.010 | `NULL` | `NULL` | `NULL` |
| `9a4d...7f` | `node-a-3` | 08:02:00.004 | 08:02:03.512 | `SUCCESS` | `NULL` |
| `b7e2...11` | `node-a-1` | 07:02:00.000 | 07:02:00.000 | `QUOTA_REJECTED` | `NULL` |
| `5c90...a4` | `node-a-2` | 06:02:00.002 | 06:02:01.890 | `SCRIPT_ERROR` | `TypeError: Cannot read property 'id' of null` |

Reading this top to bottom: the newest run is still in progress (`completed_at` is `NULL`). The one before it succeeded in ~3.5 seconds. The one before that never ran at all — rejected for quota, `started_at`/`completed_at` identical, nothing to measure. The oldest shown threw a `TypeError` partway through, captured verbatim in `failure_message`. For a `MISFIRE`/`QUOTA_REJECTED`/`ABANDONED` row, `started_at`/`completed_at` semantics differ slightly from a normal `SUCCESS`/`SCRIPT_ERROR` row — see §6/§9/§10.

## 9. Misfires are often followed by a real run

Seeing a `MISFIRE` row does **not** necessarily mean your job didn't run that day. Quartz's default behavior for a missed cron fire is to catch up by firing the job again almost immediately once the miss is detected (see §1's `threadCount` note for the most common real-world cause) — so a `MISFIRE` row for the originally-scheduled slot is commonly followed, moments later, by a completely normal `STARTED`→`COMPLETED` pair (a new, separate `fire_instance_id`) for the catch-up run. This is expected, not a bug: the `MISFIRE` row accurately records that *that one specific scheduled slot* was missed; it doesn't claim the job never ran at all. If you need to know whether your job *eventually* ran for a given day, check for a `SUCCESS`/`SCRIPT_ERROR` row near the `MISFIRE` one, not just the `MISFIRE` row in isolation.

## 10. Multiple attempts for the same trigger firing

If a cluster node dies mid-execution, a surviving node picks the job back up automatically (Quartz's own clustered recovery — always on for every job, not something you or an operator configure). When that happens, you'll see **two** rows for the same job in quick succession: the original, crashed attempt and the recovered attempt (a normal `STARTED`→`COMPLETED` pair). They're never merged into one row — each firing gets its own identity (`fire_instance_id`), so your history always shows exactly what happened, attempt by attempt.

The original, crashed attempt's row ends up with outcome `ABANDONED`: the moment the recovery execution starts, it closes out its predecessor's row (which would otherwise sit permanently `completed_at IS NULL` — nothing else would ever tell it the node it was running on is gone). You're not meant to do anything about an `ABANDONED` row yourself; it's an automatic byproduct of recovery, recorded so the predecessor's row doesn't sit open forever and doesn't keep counting against your tenant's concurrency quota indefinitely.

## 11. Metrics

Every execution that finishes — `SUCCESS`, `SCRIPT_ERROR`, `MISFIRE`, `QUOTA_REJECTED`, or `ABANDONED` — also emits into PayOS's existing metrics infrastructure (`PayOSMetrics`, the same `IMetricsRecorder`/`MetricSample` category every other PayOS component already uses — see [Event category payload contracts](event-category-payload-contracts-v8-2026-09-14.md) §4). TSS adds no metrics-specific tooling of its own: your job's metrics appear alongside every other PayOS metric in whatever backend your deployment already scrapes (Micrometer/Prometheus by default), queryable with the same tools you already use.

Two samples per finished execution:

| Sample | Type | Value/unit | Emitted for | Tags/fields |
| --- | --- | --- | --- | --- |
| `tss.job.execution.count` | `COUNTER` | `1.0` / `count` | every finished execution, including `MISFIRE`/`QUOTA_REJECTED`/`ABANDONED` | `tenantId` (own field), `correlationId` = `fireInstanceId` (own field), tags: `scriptPath`, `outcome`, `triggerType` |
| `tss.job.execution.duration` | `HISTOGRAM` | elapsed ms | only `SUCCESS`/`SCRIPT_ERROR` (a rejected/misfired/abandoned execution never ran, so there's nothing to time) | same tags as above |

`triggerType` is currently always `"cron"` — the only trigger type TSS supports today (§2); it becomes a real, variable tag only once a second trigger type exists. `outcome` is always one of the five values from §6. A platform-level job's samples carry `tenantId: null`, the same convention used everywhere else in this guide.

**`tenantId` reaches the scraped text as a `tenant_id` label; `correlationId` does not.** With the default Micrometer/Prometheus backend (confirmed by actually scraping it, not assumed):
- `tenantId` becomes a `tenant_id` tag whenever the JVM-wide system property `payos.metrics.tenant-tags-enabled` is `true`. This is a PayOS-wide flag, off by default for every component. Whether it's on depends on deployment mode:
  - **Standalone:** `StandaloneMain` sets it itself at startup (unless you've already set it explicitly yourself, in which case your own choice is respected) — standalone TSS owns 100% of its own JVM, so this can only ever affect TSS's own metrics. In practice, `tenant_id` is on for every normal standalone deployment without you doing anything.
  - **Embedded:** `RuntimeIntegrationHook` deliberately does **not** touch this flag at all — TSS is a guest in `payos-runtime`'s own JVM there, shared with the kernel's own metrics, connectors, and audit trail; deciding that host's metrics-cardinality policy isn't TSS's call to make. `tenant_id` appears on TSS's metrics in embedded mode only if the host (`payos-runtime`) has already enabled the flag for its own reasons — otherwise TSS's metrics simply have no `tenant_id` label, same as every other PayOS metric in that process.
- `correlationId` (`fireInstanceId`) is **never** exposed through `/metrics` today, regardless of that flag — `MicrometerMetricsRecorder` simply doesn't read that field at all. It still reaches any *other* `IMetricsRecorder` implementation your deployment might register via SPI (e.g. one that forwards to a tracing backend), but if you're reading Prometheus text, there is no `fireInstanceId` anywhere in it.

### What you'll actually see at `/metrics`

A real scrape, after one successful `jobs/reconciliation.js` execution for tenant `acme-corp` and one quota rejection for the same job, with tenant tagging on (standalone mode, or embedded with the host's flag already set):

```
# TYPE payos_tss_job_execution_count_total counter
payos_tss_job_execution_count_total{outcome="QUOTA_REJECTED",scriptPath="jobs/reconciliation.js",tenant_id="acme-corp",triggerType="cron"} 1.0
payos_tss_job_execution_count_total{outcome="SUCCESS",scriptPath="jobs/reconciliation.js",tenant_id="acme-corp",triggerType="cron"} 1.0
# TYPE payos_tss_job_execution_duration_seconds summary
payos_tss_job_execution_duration_seconds_count{outcome="SUCCESS",scriptPath="jobs/reconciliation.js",tenant_id="acme-corp",triggerType="cron"} 1
payos_tss_job_execution_duration_seconds_sum{outcome="SUCCESS",scriptPath="jobs/reconciliation.js",tenant_id="acme-corp",triggerType="cron"} 3.421
# TYPE payos_tss_job_execution_duration_seconds_max gauge
payos_tss_job_execution_duration_seconds_max{outcome="SUCCESS",scriptPath="jobs/reconciliation.js",tenant_id="acme-corp",triggerType="cron"} 3.421
```

Three things worth knowing, all visible above and none of them TSS-specific (every PayOS metric goes through this exact naming):

- Every metric name gets a `payos_` prefix and dots become underscores — the `tss.job.execution.count`/`tss.job.execution.duration` names from the table above are the *logical* `MetricSample` names, not literally what's in the scraped text.
- A `COUNTER` always gets a `_total` suffix.
- A `HISTOGRAM` recorded with a duration unit (`ms`, as both TSS samples are) becomes a Micrometer `Timer`, always exposed in **seconds**, as three separate series: `..._seconds_count`, `..._seconds_sum`, and a `..._seconds_max` gauge — never a single `..._duration_ms` series. To get average duration, divide `_sum` by `_count` over your query window, the same as any other Micrometer timer in PayOS.

### 11.1 How these metrics actually get exposed — embedded vs. standalone

Recording a sample and *exposing* it over HTTP are two different things. `MetricsRecorder.record(...)` writes into a **static, in-process** Prometheus registry (`MicrometerMetricsRecorder`'s own `PrometheusMeterRegistry`, one per JVM) — nothing about recording a sample starts a server or pushes anything anywhere. Something still has to scrape that registry and serve it as text over HTTP.

- **Embedded mode** (TSS running inside `payos-runtime`): no extra configuration needed. TSS shares the host's JVM, and `payos-server-http`'s own `GET /metrics` endpoint already scrapes that exact same static registry — TSS's job-execution metrics show up there automatically, next to every other PayOS metric, the moment `JobMetricsEmitter` records one. This is the normal, expected deployment for metrics.
- **Standalone mode** (TSS as its own process): there is **no HTTP server anywhere in that process** — `payos-job-scheduler` doesn't depend on `payos-server-http` at all. Without `TSS_METRICS_HTTP_PORT` set, recorded samples sit in that process's own in-memory registry and are never reachable from outside it — recorded, but not exposed. Set `TSS_METRICS_HTTP_PORT` (or `metricsHttpPort` in `tss-config.json`) to a positive port to start a tiny, purpose-built Prometheus HTTP server (`io.prometheus.metrics.exporter.httpserver.HTTPServer`, wired directly against the same registry `MicrometerMetricsRecorder.prometheusRegistry()` exposes) — point your scraper at `http://<standalone-host>:<port>/metrics` the same way you'd point it at any other Prometheus target. Everything about how a sample becomes a metric (naming, tags, counter/histogram mapping) stays identical either way — this setting only controls whether something is listening to serve it.

This is deliberately opt-in (`0` = disabled by default): a brand-new network listener on every standalone TSS node isn't something to turn on silently.

**A misconfigured port never crashes the scheduler.** If the configured port is already in use (or binding otherwise fails), TSS logs a warning and keeps running — job scheduling itself is completely unaffected, you simply won't have `/metrics` reachable for that run. Check the startup logs for `Failed to start the standalone metrics HTTP server` if the endpoint isn't responding.

## 12. Sensitive data masking

Before a script error's message ever reaches `tss_job_execution.failure_message` or a log line, TSS masks sensitive-looking fragments inside it — always on, no configuration, nothing to opt into.

**What gets masked:** any `key: value`/`key=value`-shaped fragment whose key name matches PayOS's one canonical denylist (`password`, `secret`, `token`, `apikey`, `privatekey`, `accesskey`, `credential`, `pan`, `cardnumber`/`creditcard`, `cvv`/`cvv2`/`cvc`, `pin`/`pinblock`, `ssn`, `iban`, `authorization`, `cookie` — the same list every other PayOS component uses, matched case-insensitively regardless of separator style). Only the *value* is replaced, with `***`; the key name and the rest of the message are left untouched.

```
throw new Error("password=hunter2 invalid")
```
becomes, in both `tss_job_execution.failure_message` and the exception message TSS's own logs carry:
```
password=*** invalid
```

**This is explicitly best-effort, not a guarantee — write defensively, don't rely on it to catch everything.** It's a pattern scanner, not a value-content scanner: a message with no recognizable `key:`/`key=` shape at all passes through completely unchanged. A bare, unlabeled secret with no key prefix — a raw card number pasted into a message with nothing like `cardNumber:` in front of it — is **not** caught; there's no way for a key-name-based mechanism to know an unlabeled value was ever sensitive. If your script constructs its own error messages, prefer including a recognizable field name when the value might be sensitive (`throw new Error("token=" + token)` rather than `throw new Error(token)`) — the former gets masked, the latter doesn't.

**Metrics are masked too, defensively** — `JobMetricsEmitter`'s tag map passes through the same canonical masking mechanism before every sample is recorded. Today's own tags (`scriptPath`, `outcome`, `triggerType`) never match the sensitive-key list, so this is normally invisible; it exists so a future tag addition doesn't accidentally bypass masking.

## Next

- [Configuration: Task Scheduling System](../configuration/task-scheduling-system-v4-2026-10-07.md) — condensed config-key reference, for a quick lookup once you know this material.
- [Operations: Task Scheduling System](../operations/task-scheduling-system-v15-2026-10-07.md) — deployment modes, clustering, starting/stopping, operational troubleshooting.
- [Architecture: Task Scheduling System class diagram](../architecture/job-scheduler-class-diagram-v18-2026-10-07.md) — internal design, every production class (French).
- [Data access](data-access.md) / [Platform database usage](platform-database-usage.md) — the same `$DB`/`$PlatformDB` concepts as used by API scripts, for comparison.
- [Tenant quota enforcement](tenant-quota-enforcement.md) — the separate, HTTP-request-side quota mechanism.
