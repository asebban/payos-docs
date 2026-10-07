Created: 2026-10-07
Last updated: 2026-10-08
Version: 2

# TSS standalone smoke-test walkthrough (`client2` bundle)

A worked example of preparing and running a real Task Scheduling System (TSS) deployment against an existing PayOS bundle, plus three real findings live runs surfaced — one of them (a scheduler startup-ordering race) fixed along the way. This is a walkthrough of one concrete setup, not a reference — for the general configuration contract see [Running the Task Scheduling System (TSS)](task-scheduling-system-v15-2026-10-07.md) and its [class diagram](../architecture/job-scheduler-class-diagram-v18-2026-10-07.md).

## Goal and target

The request was to prepare a whole configuration able to run real jobs against `C:\Projets\DTS\MedTech@Work\Innovation\S2M\payos-bundles\client2`, an existing `payos-runtime` bundle (its own `bootstrap.json`, `payos.json`, `default.database.json`, `multitenancy.json`, `default.security.json`, `secrets.json`, plus a prebuilt `payos-job-scheduler-0.1.0-SNAPSHOT.jar` already sitting next to `payos-runtime-2.0.0-RELEASE.jar`).

## Deployment mode: standalone, not embedded

Embedded mode (`RuntimeIntegrationHook`) needs `payos-runtime` itself to call `RuntimeIntegrationHook.start(...)` during its own boot sequence. Grepping the `payos` repo found no such call site anywhere — embedding TSS into `payos-runtime` has never actually been wired up on the runtime side, so it would need a code change and a rebuild there first. Standalone mode needs none of that: the shaded jar in `client2` has `Main-Class: ma.s2m.payos.scheduler.bootstrap.StandaloneMain` and bundles every dependency (Quartz, H2 driver, `dynamic-database-service`'s `IDatabaseServiceFactory` via `META-INF/services`), so it runs as-is with `java -jar`. Standalone was the only mode that could actually be tested without touching another repo.

## TSS's own configuration file

Created `tss-config.json` at the bundle root:

```json
{
  "database": {
    "dbUrl": "jdbc:h2:file:./.h2/tss_db;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE",
    "dbUser": "sa",
    "dbPassword": "",
    "dbDriver": "org.h2.Driver",
    "dbVendor": "h2",
    "dbMaxPoolSize": 5
  },
  "scheduler": {
    "schedulerInstanceName": "TSS-Client2",
    "threadCount": 5,
    "metricsHttpPort": 9404
  }
}
```

The database block is deliberately a brand-new H2 file (`.h2/tss_db`), separate from the bundle's existing `.h2/payos_db` — TSS's own Quartz/`tss_job_execution` schema must stay isolated from tenant/platform data (see the class diagram's "why not reuse `DynamicDataAccessService` directly" note), so it gets its own file rather than a new schema bolted onto the existing one. `metricsHttpPort: 9404` turns on the Story 2.3 standalone-only Prometheus HTTP server, since this process has no `payos-server-http` of its own to expose `/metrics` through.

Everything else (`jobsRootDir`, `payosConfigFile`) was left at its default — `jobs/` and `payos.json`, both resolved relative to `--scheduler-base-dir`. Pointing `TSS_PAYOS_CONFIG_FILE` at the bundle's own real `payos.json` was the key decision: `TenantDatabaseRegistryLoader.loadPayOSSettings` only reads that file for its `configDir` pointer, then calls the exact same `ConfigLoader.loadConfigByDirectory` the kernel itself uses to merge every `*.json` in that directory — so `default.database.json`'s `database-service` block and `multitenancy.json`'s tenant declarations were picked up automatically, with no duplication of the bundle's real tenant/database configuration anywhere in `tss-config.json`.

## Applying TSS's database schema

Quartz does not create its own tables, and TSS does not auto-apply `tss_job_execution`'s DDL either — both are meant to be applied once, by an operator, before first start. Both SQL scripts are bundled inside the shaded jar (`org/quartz/impl/jdbcjobstore/tables_h2.sql`, `db/migration/tss_job_execution.sql`), so no separate download was needed — they were extracted and applied with the jar's own bundled H2 tools:

```bash
jar tf payos-job-scheduler-0.1.0-SNAPSHOT.jar | grep -E "tables_h2|tss_job_execution"   # confirm both are present
unzip -p payos-job-scheduler-0.1.0-SNAPSHOT.jar org/quartz/impl/jdbcjobstore/tables_h2.sql > tables_h2.sql
unzip -p payos-job-scheduler-0.1.0-SNAPSHOT.jar db/migration/tss_job_execution.sql > tss_job_execution.sql

java -cp payos-job-scheduler-0.1.0-SNAPSHOT.jar org.h2.tools.RunScript \
  -url "jdbc:h2:file:./.h2/tss_db;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE" -user sa -password "" -script tables_h2.sql
java -cp payos-job-scheduler-0.1.0-SNAPSHOT.jar org.h2.tools.RunScript \
  -url "jdbc:h2:file:./.h2/tss_db;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE" -user sa -password "" -script tss_job_execution.sql
```

`SHOW TABLES` afterward confirmed all 11 `QRTZ_*` tables plus `TSS_JOB_EXECUTION`, all in the `PUBLIC` schema of the new, dedicated `tss_db` file.

## Job registration: one of each shape, chosen to prove a specific thing

Script files live directly under the shared `jobs/` pool (`FileSystemScriptLoader`'s "shared script pool" design); registration files are nested per tenant (`jobs/<tenant>/jobs.json`) or, for platform-level jobs, a single `jobs/platform-jobs.json` at the root (`JobConfigLoader`'s reserved filename — not a `platform/` subdirectory, which is reserved for the Quartz group name instead). Three jobs were registered, each picked to exercise a different real path rather than three variations on the same thing:

| Job | Shape | What it proves |
| --- | --- | --- |
| `platform-heartbeat` | Platform-level, always succeeds, fires every minute | The full happy path: `STARTED`/`COMPLETED` events, a `SUCCESS` history row, `PayOSMetrics` counter/histogram samples — with no `$DB`/`$PlatformDB` access needed at all |
| `platform-error-demo` | Platform-level, always throws `Error("... password=hunter2, retry with apiKey=sk-demo-12345")`, fires every minute | Story 2.4's masking end to end, in a real deployed jar rather than a unit test — does the persisted `failure_message` and the thrown exception's own message actually come out masked? |
| `tenant1-db-touch` | Tenant-scoped (`tenant1`, already declared in the bundle's own `multitenancy.json`), calls `$DB.newParams()`, fires every minute | That `$DB` is a real, open session against `tenant1`'s own dedicated schema for a tenant this bundle already runs — not a mock |

`tenant1` was reused rather than invented specifically so the test exercised the bundle's own real, already-declared multitenancy configuration (dedicated-schema isolation mode) instead of a synthetic tenant that would never prove anything about this bundle.

## Running it

```powershell
# start-tss.ps1
java -jar payos-job-scheduler-0.1.0-SNAPSHOT.jar --scheduler-base-dir .
```

Run from the bundle root (`client2`). The schema-application step above only needs to run once, before the first start.

## What the real run actually showed

The jar was started and left running long enough for every job to fire at least twice. Startup logged `Registered 3 job(s) across 1 tenant(s)` and `TSS standalone metrics HTTP server listening on port 9404` — confirming the config was read correctly before any job ever fired.

Querying `tss_job_execution` directly afterward:

```
JOB_ID              | TENANT_ID | OUTCOME      | FAILURE_MESSAGE
platform-heartbeat  | null      | SUCCESS      | null
tenant1-db-touch    | tenant1   | SUCCESS      | null
platform-error-demo | null      | SCRIPT_ERROR | Error executing job script 'jobs/platform-error-demo.js': Error: Upstream call failed: password=*** retry with apiKey=***
```

All three jobs ran, `tenant1-db-touch` correctly carries `tenant_id = 'tenant1'` proving the dedicated-schema session opened against the real tenant, and the masked `failure_message` reads `password=***`/`apiKey=***` — never the real secrets — exactly as Story 2.4 intends.

`curl http://127.0.0.1:9404/metrics` showed `payos_tss_job_execution_count_total`/`_duration_seconds*` series for both platform jobs, correctly tagged by `scriptPath`/`outcome`/`triggerType`.

## Three real findings these runs surfaced

None of these was hypothetical — all three came directly out of real runs against this bundle, and none was caught by the existing test suite.

### 1. Quartz's own log line leaks the raw, unmasked secret

`JobRunShell`'s "threw a JobExecutionException" log line prints the full cause chain. The top-level message is masked (`password=*** retry with apiKey=***`), but the `Caused by:` line underneath it is the original, unmasked `ScriptingEngineException`/`PolyglotException` message — `password=hunter2, retry with apiKey=sk-demo-12345`, in plain text, in the log file:

```
org.quartz.JobExecutionException: Job 'platform-error-demo' (tenant 'null') failed: ... password=*** retry with apiKey=***
Caused by: ma.s2m.payos.scripting.exceptions.ScriptingEngineException: ... password=hunter2, retry with apiKey=sk-demo-12345
Caused by: org.graalvm.polyglot.PolyglotException: ... password=hunter2, retry with apiKey=sk-demo-12345
```

This is the same residual gap already called out in `SensitiveDataScrubber`'s own Javadoc and flagged during Story 2.4's code review (masking the `{}`-formatted log argument never masks a trailing raw `Throwable`'s own stack trace) — this run confirms it empirically, for the one path (`ScriptJob`'s own thrown exception, wrapping the original unmasked cause) that the code review flagged as the most consequential instance of it, since it is Quartz's own built-in failure log, not a call site TSS controls.

### 2. Tenant-scoped job metrics silently vanish from the scrape when mixed with platform jobs

`tenant1-db-touch` executed successfully twice (confirmed in the history table above), yet its metric is completely absent from the `/metrics` scrape — not just missing a tag, the whole series for that `scriptPath` never appears.

Root cause: `MicrometerMetricsRecorder`'s tenant tag is added conditionally, only when `sample.tenantId() != null`:

```java
if (TENANT_TAGS_ENABLED && sample.tenantId() != null) {
    tags.add(Tag.of("tenant_id", sample.tenantId()));
}
```

Both `platform-heartbeat` (`tenantId = null`) and `tenant1-db-touch` (`tenantId = "tenant1"`) record under the identical metric name (`payos.tss.job.execution.count`), but with two different tag-key sets — `{scriptPath, outcome, triggerType}` for the platform job, `{scriptPath, outcome, triggerType, tenant_id}` for the tenant job. `MicrometerMetricsRecorder.record(...)`'s own try/catch never logged a warning, and no exception appeared anywhere in the process log — the inconsistency is silently dropped somewhere between Micrometer's registry and the Prometheus exporter's scrape format, which is why nothing already in the test suite caught it: every existing metrics test exercises one tag shape at a time, never a platform job and a tenant job sharing the same metric name in the same registry instance within one run.

This is a real gap in Story 2.3's tenant-tagging design, invisible until a deployment actually mixes platform-level and tenant-scoped jobs under the same script-level metric — which this walkthrough's own job selection (deliberately one of each kind) is what exposed it.

### 3. Restarting against the same bundle hit a genuine startup race — found, fixed, and re-verified

The person running this walkthrough restarted the same jar against the same `client2` bundle (now with 3 jobs already persisted in Quartz's JDBC `JobStore` from the first run) and hit a real failure neither of the two findings above predicted:

```
org.quartz.JobExecutionException: ScriptJob was instantiated without ScriptExecutionDependencies —
SchedulerBootstrap must call scheduler.setJobFactory(new ScriptJobFactory(...)) before scheduling any ScriptJob
```

Root cause, confirmed by reading `SchedulerBootstrap.java`: `scheduler.start()` was called before `registerScriptJobs(...)` ever installed the real `ScriptJobFactory`. On a fresh, empty `JobStore` this ordering is harmless — there is nothing to fire in the gap. But on this restart, Quartz's own `ClusterManager`/`MisfireHandler` detected 3 triggers that had missed their fire time while the process was down, and fired one (`platform-heartbeat`) the instant `scheduler.start()` returned — through Quartz's default reflection-based factory, since the real one hadn't been installed yet. No existing test caught this because every test starts from a schema `QuartzH2TestSchema.applyTo(...)` just created, with nothing pending to misfire.

Fixed in `SchedulerBootstrap`: the static factory is now `build(...)` (constructs the scheduler, does **not** call `.start()`); a new instance method `start()` must be called explicitly, and only after `registerScriptJobs(...)` has installed the real factory. `StandaloneMain` and `RuntimeIntegrationHook` (and every test exercising the same pattern) were reordered to `build → registerScriptJobs → start`; `RuntimeIntegrationHook` calls `start()` from a `finally` block so a registration failure still leaves the scheduler running with zero jobs, matching its existing documented contract.

Re-verified against the exact same real `client2` database (still carrying the 3 pending misfired triggers from the original failure) with the fixed jar: `JobFactory set to: ScriptJobFactory@...` now logs minutes before the scheduler is ever started, the misfire handler still runs ("Handling 3 trigger(s) that missed their scheduled fire-time"), and every job fires through the real factory — `platform-error-demo` fails with its normal, expected, masked script error, never the dependency-injection error. Fixed in commit `5085fa6`, full suite still 200/200 green.

## Cleanup / re-running

The java process was stopped with a plain process kill for this walkthrough (Windows' `git-bash kill` does not run JVM shutdown hooks, so the graceful "Shutting down TSS standalone scheduler" log line was not expected and did not appear — this is a property of how the process was stopped for this test, not a product defect). `.h2/tss_db` already contains this run's history rows; delete it and re-apply both DDL scripts above for a clean slate, or leave it and keep appending.
