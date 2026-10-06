Created: 2026-10-06
Last updated: 2026-10-06
Version: 2

# Task Scheduling System: execution outcomes & history

For developers writing job scripts for the Task Scheduling System (`payos-job-scheduler`, "TSS") — a Quartz-backed, clustered scheduler that fires your script on a cron schedule, separate from the HTTP/TCP/queue request path the rest of this guide otherwise covers. This page is about what happens **around** your script's execution — specifically, the cases where your script's `execute` function never runs at all, and how to look up what actually happened to any of your job's past runs. Registering the job itself (`jobs.json`, the cron expression, `parallelizable`) is covered in [configuration/task-scheduling-system.md](../configuration/task-scheduling-system.md); the full operational picture (deployment modes, clustering, starting/stopping) is in [operations/task-scheduling-system-v9-2026-10-06.md](../operations/task-scheduling-system-v9-2026-10-06.md).

## The script contract, concretely

A job script is a plain JS resource with the same shape as an API script, minus `loadControlData` — just one top-level `execute(context)` function:

```javascript
// jobs/reconciliation.js
function execute(context) {
    var entity = $DB.newParams();
    entity.put('status', 'reconciled');
    $DB.save('Invoice', entity);
}
```

`context` is TSS's own `JobContext` (not the `request`/`controlData` pair an API script gets) — it exposes `jobId`/`tenantId`/`scriptPath`/`getDb()`/`getPlatformDb()`, mirroring what `$DB`/`$PlatformDB` give you globally. Both bindings behave exactly like their API-script counterparts once bound — see [data-access.md](data-access.md) and [platform-database-usage.md](platform-database-usage.md) for the full `$DB`/`$PlatformDB` surface (manual transaction control included — a job script may legitimately call `$DB.beginTransaction()`/`commitTransaction()` for intermediate commits across a long batch, unlike an API script).

- For a **tenant-scoped job**, `$DB` is bound to that tenant's session; `$PlatformDB` is bound **read-only** (no `save`/`update`/`delete`).
- For a **platform-level job** (no tenant owner — see `configuration/task-scheduling-system.md`'s `platform-jobs.json`), `$DB` is **not bound at all** (`typeof $DB === 'undefined'`); `$PlatformDB` is bound **read+write**.

## The four outcomes

## The four outcomes

Every execution TSS records lands on exactly one of four outcomes:

| Outcome | Your `execute` function ran | Meaning |
| --- | --- | --- |
| `SUCCESS` | Yes | Completed without throwing. |
| `SCRIPT_ERROR` | Yes | Threw, or the engine couldn't find/execute an `execute` function at all. |
| `MISFIRE` | **No** | Quartz detected your trigger's scheduled fire time passed without ever actually starting the job (every worker thread was busy, or the cluster was briefly unavailable). |
| `QUOTA_REJECTED` | **No** | Your tenant already had too many of this job's executions in progress at once — see [Concurrency quota](#concurrency-quota-maxconcurrentexecutions) below. |

The two "no" cases matter for how you debug a job that silently "didn't run": there's nothing to catch in your script, nothing in your script's own logs — the rejection happened entirely before your code was ever reached, the same way a tenant's HTTP request can be rejected by `TenantPolicyService` before `execute` runs (see [Tenant quota enforcement](tenant-quota-enforcement.md), a different mechanism for a different kind of request — see [Not the same quota as HTTP requests](#not-the-same-quota-as-http-requests) below).

## Concurrency quota (`maxConcurrentExecutions`)

An operator can cap how many of your tenant's job executions may be in progress **at once**, cluster-wide — a concurrency cap, not a rate limit (nothing resets per minute; it's purely "how many right now"). If your tenant is at that cap when a new trigger fires, that execution is rejected outright (`QUOTA_REJECTED`) rather than queued or delayed.

- The operator sets a global default (`TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT`) applied to every tenant, and/or your own tenant's specific override (`maxConcurrentExecutions` in your `jobs.json`) if your workload genuinely needs a different number than everyone else. See [configuration/task-scheduling-system.md](../configuration/task-scheduling-system.md) for exactly where each is set.
- A platform-level job (one with no tenant owner) is never subject to this — there's no tenant to meter.
- This counts **all** of your tenant's in-progress executions together, across every job — it's a per-tenant cap, not a per-job cap.

Your own override lives at the top of your tenant's `jobs.json`, as a sibling of the `jobs` array:

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

## Querying your job's history

TSS has no network API — the database is the query interface (same position as every other part of TSS; see [configuration/task-scheduling-system.md](../configuration/task-scheduling-system.md)). Every execution of your job — successful, failed, misfired, or quota-rejected — is one row in `tss_job_execution`, keyed by `(tenant_id, script_path)`, the same identity Quartz's own scheduling already uses for your job:

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

Reading this top to bottom: the newest run (`c3f1...e2`) is still in progress (`completed_at` is `NULL`) — this is the same row `findCurrentlyExecuting`-style "what's running right now" views would also show. The one before it succeeded in about 3.5 seconds. The one before that never ran at all — rejected for quota, `started_at`/`completed_at` identical, nothing to measure. The oldest shown threw a `TypeError` partway through, captured verbatim in `failure_message`.

For a `MISFIRE` or `QUOTA_REJECTED` row, `started_at` and `completed_at` are always identical — there's no real duration to measure for an execution that never started. For a currently-running execution, `completed_at` is `NULL`.

## Multiple attempts for the same trigger firing

If a cluster node dies mid-execution, a surviving node can pick the job back up (Quartz's own clustered recovery — the operator enables this per job; see [configuration/task-scheduling-system.md](../configuration/task-scheduling-system.md)). When that happens, you'll see **two** rows for the same job in quick succession: the original attempt (permanently `completed_at IS NULL` — an honest record that it never finished, not a bug you need to investigate) and the recovered attempt (a normal `STARTED`→`COMPLETED` pair). They're never merged into one row — each firing gets its own identity (`fire_instance_id`), so your history always shows exactly what happened, attempt by attempt.

## Next

- [Data access](data-access.md) / [Platform database usage](platform-database-usage.md) — the `$DB`/`$PlatformDB` bindings your script actually uses.
- [Tenant quota enforcement](tenant-quota-enforcement.md) — the separate, HTTP-request-side quota mechanism.
- [Configuration: Task Scheduling System](../configuration/task-scheduling-system.md) — setting the concurrency quota, enabling recovery, `jobs.json`/`tss-config.json` reference.
- [Operations: Task Scheduling System](../operations/task-scheduling-system-v9-2026-10-06.md) — the full operational picture.
