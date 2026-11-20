Created: 2026-10-06
Last updated: 2026-11-20
Version: 12 — supersedes v11: documents standalone-mode license validation (`LicenseAuthenticityValidator`, `PAYOS_LICENSE_DEV_MODE`, the editor secret provider env vars) and `${VAR}`/`${file:...}` token resolution plus `CryptoService` decryption support in `tss-config.json`/`jobs.json`/`custom-jobs.json`/`platform-jobs.json`/job scripts/the top-level PayOS config file — corrects this page's own earlier "TSS never expands placeholders" and "encrypted files are not supported in standalone mode" statements, which were accurate before this revision but no longer are. See [Secrets, encryption and license validation (standalone mode)](#secrets-encryption-and-license-validation-standalone-mode) below.

# Task Scheduling System configuration

**Not a `bootstrap.json`/`PayOSConfig` block in standalone mode, unlike every other page in this directory** (embedded mode reads a `task-scheduler` block of `payos.json` instead — see [Embedded mode](#embedded-mode--the-task-scheduler-block-of-payosjson)). `payos-job-scheduler` ("TSS") is its own standalone deployable with its own, independent configuration pipeline (`tss-config.json`, resolved by its own `StandaloneConfigLoader`) — it never reads `bootstrap.json` and its keys are not in `IConfigSpec`, they're in TSS's own `IJobConfigSpec`. This page exists so TSS's config surface is still discoverable from here; the full operational narrative (deployment modes, clustering, tenant/platform database resolution) lives in [operations/task-scheduling-system-v22-2026-11-20.md](../operations/task-scheduling-system-v22-2026-11-20.md).

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
| `TSS_DB_PASSWORD` | `dbPassword` | *(none)* | Password. The environment variable wins priority either way, but a value that must live in the file can now be a token (`"dbPassword": "${TSS_DB_PASSWORD}"`) instead of a literal secret — see [Secrets, encryption and license validation (standalone mode)](#secrets-encryption-and-license-validation-standalone-mode) below. |
| `TSS_DB_DRIVER` | `dbDriver` | *(required)* | JDBC driver class, e.g. `org.postgresql.Driver`. |
| `TSS_DB_VENDOR` | `dbVendor` | *(generic ANSI SQL)* | `postgresql` or `oracle` — selects Quartz's `DriverDelegate`. Any other value (including unset) resolves to Quartz's generic `StdJDBCDelegate`. |
| `TSS_DB_SCHEMA` | `dbSchema` | *(none)* | Schema name, if your deployment uses a non-default one. |
| `TSS_DB_MAX_POOL_SIZE` | `dbMaxPoolSize` | `5` | Connection pool size for TSS's own pool. Also the pool `JobExecutionRepository`/`TenantConcurrencyQuotaGuard` reuse — see the class diagram, §13.4. |

## `scheduler` block — identity, sizing, and quota

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_SCHEDULER_INSTANCE_NAME` | `schedulerInstanceName` | `TSS-Cluster` | Every clustered node sharing one `jobs/` registry must use the **same** name. |
| `TSS_THREAD_COUNT` | `threadCount` | `5` | Quartz worker thread pool size — how many job executions can run concurrently on this one node. |
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `maxConcurrentExecutionsPerTenant` | `0` (disabled) | **Global default** for the per-tenant concurrency quota (how many of one tenant's executions may be in progress at once, cluster-wide, across every node). `0` or negative means no tenant is ever rejected on this basis. A tenant can override this for itself via `jobs.json`'s own `maxConcurrentExecutions` field (below). Platform-level jobs are never subject to it — there's no tenant to meter. Exceeding it produces the `QUOTA_REJECTED` outcome — see [developer/task-scheduler-guide-v22-2026-11-20.md](../developer/task-scheduler-guide-v22-2026-11-20.md). |
| `TSS_MAX_PARALLEL_WORKERS_PER_JOB` | `maxParallelWorkersPerJob` | `4` | Ceiling on how many worker threads one `$Parallel.forEach` call (tenant-scoped job scripts only) may use to fan a single execution's own batch out. A different axis from the quota above: that one bounds how many whole executions run at once; this one bounds how much *one* execution can fan out internally. See [developer/task-scheduler-guide-v22-2026-11-20.md §4](../developer/task-scheduler-guide-v22-2026-11-20.md#4-parallel--processing-a-batch-concurrently). Since workers can also read the (read-only) `$PlatformDB`, up to `threadCount × maxParallelWorkersPerJob` platform reads may be in flight at once — size `platform-database-service.configuration.max-pool-size` (HikariCP default `10`) accordingly. |
| `TSS_METRICS_HTTP_PORT` | `metricsHttpPort` | `0` (disabled) | **Standalone mode only.** `0`/negative never starts anything — embedded mode never reads this key at all. A positive value starts a tiny Prometheus scrape HTTP server on that port, serving `GET /metrics` — see [developer/task-scheduler-guide-v22-2026-11-20.md §12](../developer/task-scheduler-guide-v22-2026-11-20.md#12-metrics) for why standalone mode needs this and embedded mode doesn't. |
| `TSS_HOT_RELOAD_ENABLED` | `hotReloadEnabled` | `true` | Watches `jobs/` (with the same `ConfigWatcher` PayOS uses for its own configuration) and, about 2 s after the last change, reconciles the scheduler against the edited files — new jobs, changed triggers, `parallelizable` and `arguments`, removed jobs — without a restart. `false` means changes are applied only at startup or by the `sync` command. Embedded mode: `task-scheduler.scheduler.hotReloadEnabled`. See [operations §Changing jobs without a restart](../operations/task-scheduling-system-v22-2026-11-20.md#changing-jobs-without-a-restart). |
| `TSS_EVENT_TRIGGER_REFRESH_SECONDS` | `eventTriggerRefreshSeconds` | `30` | How often each node re-reads every job's declared event-driven triggers (`manual`/`queue`/`dependency`) and starts or stops them in memory to match — the delay before a change made elsewhere (another node's `sync`, a hot-reloaded edit) takes effect on *this* node. `0`/negative disables the periodic re-check; a refresh still happens at startup and after this node's own local reconciliation. Has no effect on temporal triggers (`cron`/`interval`/`once`), which Quartz itself tracks with no such delay. |

## `queue` block — TSS's own broker connection

TSS opens its own connection to a message broker for `queue` triggers (see the [developer guide](../developer/task-scheduler-guide-v22-2026-11-20.md#queue--fire-on-a-broker-message)) — entirely independent of any queue connection the host PayOS application already has for its own purposes (`payos-server-queue`, a notification connector...), the same way TSS's own database (above) is never a tenant's or the host's. Absent `TSS_QUEUE_TYPE` (the default): no queue client is connected at all, and a declared `queue` trigger then simply fails to start (logged, retried at the next event-trigger refresh) until it's configured. A connection failure at startup (`TSS_QUEUE_TYPE` set but the broker unreachable) is logged, never fatal, in both deployment modes — confirmed 2026-11-15 against a real broker, after fixing a standalone-mode regression where it used to kill the whole TSS process; queue triggers simply stay off (same retry) until connectivity is restored, exactly like the "no `TSS_QUEUE_TYPE` at all" case above.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_QUEUE_TYPE` | `queueType` | *(none — no queue client)* | The broker type to connect with, matching an `IQueueClientFactory#type()` found via SPI (`"nats"`...). |
| `TSS_QUEUE_HOST` | `queueHost` | *(required once `TSS_QUEUE_TYPE` is set)* | Broker host. |
| `TSS_QUEUE_PORT` | `queuePort` | `4222` (NATS's own default) | Broker port. |
| `TSS_QUEUE_ADAPTERS_DIR` | `queueAdaptersDir` | *(none — classpath only)* | Optional directory of extra `IQueueClientFactory` provider JARs (e.g. a Kafka or RabbitMQ connector you add without rebuilding TSS), loaded in addition to whatever connector is already bundled on the classpath (e.g. `queue-service-nats` as an ordinary Maven dependency of a custom TSS build). The same drop-in-a-JAR mechanism `payos` itself offers through `service-adapters-dir`, reimplemented independently here so TSS's queue code never depends on `PayOSConfig`. A relative path resolves against `--scheduler-base-dir` in standalone mode; in embedded mode (env var/system property only, no base-directory concept there), use an absolute path. |

In standalone mode these four keys resolve across the same three tiers as every other block, under `tss-config.json`'s `"queue"` block. In embedded mode (`RuntimeIntegrationHook`, which reads no TSS configuration file of its own) only the environment variable and system property tiers apply — there is no `task-scheduler.queue` block in `payos.json`.

A known limitation worth knowing before relying on `queue` triggers in a cluster: a destination is backed by one shared, durable consumer group cluster-wide — several of your own nodes subscribing to the same destination share its message load rather than each seeing every message (which is what keeps a message from firing the job more than once), but for the exact same reason, **two different jobs that both declare a `queue` trigger on the same destination also share that destination's messages between them** rather than each receiving every one. Give each job its own destination unless you specifically want several jobs to split one stream's load between them.

**`destination` naming is connector-specific — confirmed against a real broker, 2026-11-15.** `queue-service-nats`'s `NatsQueueClient` only accepts a destination made of letters, digits, `_` and `-`: it is used directly as the JetStream stream name, which cannot contain `.`, so a dotted, subject-hierarchy-style name otherwise idiomatic for NATS pub/sub (`payments.settled`) is rejected at subscribe time (`destination '...' is invalid: only letters, digits, '_' and '-' are allowed`), logged and retried exactly like any other failure to start a `queue` trigger — never fatal, but the trigger never actually starts until the name is fixed. Name every `queue` destination `payments-settled`-style (hyphens, not dots) to stay safe with this connector; a future connector for a different broker may have its own, different rule.

## `jobs` block — file locations

Both values are resolved **relative to `--scheduler-base-dir`** (standalone mode) or the embedded host's bundle path, not the process's working directory, when given as relative paths.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_JOBS_ROOT_DIR` | `rootDir` | `jobs` | Directory `JobConfigLoader` walks — one subdirectory per tenant (each with its own `jobs.json`/`custom-jobs.json`), plus this directory's own `platform-jobs.json` for platform-level jobs. |
| `TSS_PAYOS_CONFIG_FILE` | `payosConfigFile` | `payos.json` | The PayOS config file TSS reads tenant/platform database settings from — standalone mode only (embedded mode reuses the host's already-loaded `PayOSConfig.settings` directly, no file re-read). After merging that file's `configDir`, standalone TSS also runs every SPI-discovered PayOS config loader (`IConfigLoader`) the same way the runtime does — e.g. `tenant-config-loader-database`, bundled in the standalone jar and inactive unless its `tenant-config-loader` block is enabled — so tenants declared in a database registry table are seen without copying their JSON declarations. It also loads the JARs of that configuration's `service-adapters-dir` (relative to the PayOS config file's directory; `PAYOS_SERVICE_ADAPTERS_DIR` takes priority), as the runtime does: their config loaders and database factories are used, and Hibernate mapping files shipped in them as classpath resources resolve when a tenant or the platform database lists them. See [operations §Tenant and platform database resolution](../operations/task-scheduling-system-v22-2026-11-20.md#tenant-and-platform-database-resolution). |

## `admin` block — administration endpoint

The secured HTTP endpoint for manual triggering, on-demand reconciliation and job listing. Off unless a port is set. Callers authenticate with an OIDC Bearer token verified by PayOS's own security service (the `security` configuration of `payos.json`, or of the host runtime in embedded mode); the roles below decide what they may do. Embedded mode reads the same fields from `task-scheduler.admin`.

| Key | JSON field | Default | Purpose |
| --- | --- | --- | --- |
| `TSS_ADMIN_HTTP_PORT` | `httpPort` | `0` (disabled) | Port of the administration endpoint (`/tss/admin/...`), its own listener in both modes so it can be firewalled separately. `0` or negative never starts it. |
| `TSS_ADMIN_PLATFORM_ROLE` | `platformRole` | `tss-platform-admin` | Token role granting every operation, on every tenant and on platform jobs, including `sync`. |
| `TSS_ADMIN_TENANT_ROLE` | `tenantRole` | `tss-tenant-admin` | Token role granting `trigger` and `list` on the caller's own tenant only — the tenant carried by the token itself, derived from its issuer realm. |

See [operations §Administration: manual trigger, sync and list](../operations/task-scheduling-system-v22-2026-11-20.md#administration-manual-trigger-sync-and-list) for the endpoints, the command-line equivalents and the audit trail.

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
    "hotReloadEnabled": true,
    "eventTriggerRefreshSeconds": 30
  },
  "queue": {
    "queueType": "nats",
    "queueHost": "broker.internal",
    "queuePort": 4222
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

`dbPassword` is deliberately omitted — set `TSS_DB_PASSWORD` as an environment variable instead. `rootDir`/`payosConfigFile` are shown here at their defaults; omit the whole `jobs` block if you don't need to override either. The `queue` block is entirely optional — omit it (the default) if no job declares a `queue` trigger; `queuePort` is shown here at its own default too.

## Secrets, encryption and license validation (standalone mode)

Added 2026-11-20. Three related pieces, all gated on the same license check: a value in a standalone-mode JSON file can reference a token instead of a literal secret, the file itself can be shipped encrypted, and both require the same `PAYOS_LICENSE_DEV_MODE=false` real license check that unwraps the AES key making decryption possible — see [`LicenseValidatorProvider.createAuthenticityOnly`](../architecture/license-validation-architecture-v5-2026-11-20.md#17-payos-job-scheduler-tss-standalone--licenseauthenticityvalidator) for the mechanism.

**`${VAR}`/`${VAR:-default}`/`${VAR:?message}`/`${file:/path}`/`${config:a.b}` tokens** — the exact same syntax the kernel's own `ConfigLoader` resolves `payos.json` with (see [env-var-resolution.md](env-var-resolution.md)), via the same shared `ma.s2m.payos.config.EnvVarResolver` class — are now resolved in every JSON file standalone TSS reads: `tss-config.json`, `jobs.json`/`custom-jobs.json`/`platform-jobs.json`, and the top-level PayOS config file (`TSS_PAYOS_CONFIG_FILE`, default `payos.json`) this module reads for tenant/platform database settings. A string field in any of them can reference a token — `"dbPassword": "${TSS_DB_PASSWORD}"`, a job's `"arguments": {"apiKey": "${EXPORT_API_KEY}"}` — instead of a literal value. `template.json`'s own `{{tenantId}}` substitution happens first and separately — see the note on it above.

**`CryptoService.decryptIfEncrypted`** — the same mechanism, and once a real license is validated the same license-derived AES key, the kernel's own resource loaders use — now runs against every one of those same files before they are parsed: `tss-config.json`, `jobs.json`/`custom-jobs.json`/`platform-jobs.json`, `template.json`, the top-level PayOS config file, and every job script (`.js`) `FileSystemScriptLoader` reads. Any of them can therefore be shipped encrypted with `payosv2-packer` (the `P8G2` AES/GCM format), exactly like `payos.json` itself. **This supersedes this page's own earlier statement that encrypted configuration files are not supported in standalone mode** — that was accurate only because standalone TSS never initialized the editor encryption key at all; it now does, as part of license validation, below.

| File | Decrypted via `CryptoService`? | `${...}` tokens resolved? |
| --- | --- | --- |
| `tss-config.json` | Yes | Yes |
| `jobs.json` / `custom-jobs.json` / `platform-jobs.json` | Yes | Yes |
| `template.json` | Yes | No — only `{{tenantId}}`, at provisioning time (see above); the `jobs.json` it produces *is* token-resolved once actually loaded |
| Top-level PayOS config file (`payos.json`/`bootstrap.json`) | Yes | Yes — the rest of that configuration directory was already resolved this way via the kernel's own `ConfigLoader.loadConfigByDirectory` |
| Job scripts (`.js`) | Yes | No — not JSON; a script needing a secret reads it via `arguments`/`$Secrets`, not a token in its own source |

**License validation (standalone mode, `run` command only).** Both of the above depend on `CryptoService` actually having a key, which only happens once a real license is validated — `sync`/`trigger`/`list` are unaffected, gated instead on direct TSS database access. Checked before `tss-config.json` is even read, mirroring the kernel's own boot order (`BootServer`):

| Key | Default | Purpose |
| --- | --- | --- |
| `PAYOS_LICENSE_DEV_MODE` | unset = dev mode (no check) | Same variable the kernel reads. `false` enables the real check (`LicenseAuthenticityValidator`) and, on success, installs the license's AES key on `CryptoService`; unset or `true` runs a no-op (`DevLicenseValidator`) and leaves `CryptoService` keyless — an actually-encrypted file then fails to load with "Encryption key not configured." |
| `PAYOS_EDITOR_ID` / `PAYOS_EDITOR_KEY_NAME` | same defaults as the kernel | The signer identity the signature is checked against — never read from the license document itself. |
| `PAYOS_EDITOR_SECRET_TYPE` / `PAYOS_EDITOR_SECRET_CFG_*` / `PAYOS_EDITOR_SECRET_ENABLED` | — | Resolve the editor secret provider purely from environment variables, the same way the kernel does for its own pre-config license check — necessary here too, since `tss-config.json` cannot itself be read yet to provide this. |

The license file itself follows the same `LicenseFileLocator` convention as the kernel, resolved against `--scheduler-base-dir`. Unlike the kernel's own check, TSS never validates entitlements or quotas (it has none of its own) and accepts a license with no `machine_bindings` declared at all (machine fingerprint is checked only when the document declares one). Full mechanism: [architecture/license-validation-architecture-v5-2026-11-20.md §17](../architecture/license-validation-architecture-v5-2026-11-20.md#17-payos-job-scheduler-tss-standalone--licenseauthenticityvalidator); operator procedure: [operations guide](../operations/task-scheduling-system-v22-2026-11-20.md#license-validation-checked-before-any-configuration-file).

## Embedded mode — the `task-scheduler` block of `payos.json`

When TSS runs inside `payos-runtime` (`TaskSchedulerRuntimeIntegration`), it does not read `tss-config.json` or the `TSS_*` environment variables. It reads one block of the host's own PayOS configuration, `task-scheduler`, with the same JSON field names as the standalone blocks above. Because this block is part of the PayOS configuration, `${...}` placeholders are expanded by the PayOS loader (see [env-var-resolution.md](env-var-resolution.md)), so the password can come from the environment.

**Where to put the block.** Like any PayOS setting, the block must sit in `payos.json` or in a `*.json` file **directly** in the runtime's configuration directory (`configDir`). The runtime does not read subdirectories, so a `task-scheduler.json` placed in, say, `scheduler/` is silently ignored and TSS does not start, with no TSS line in the log.

| Field | Default | Purpose |
| --- | --- | --- |
| `task-scheduler.enabled` | `false` | Must be `true` for TSS to start inside the runtime; anything else skips it. |
| `task-scheduler.database.*` | — | `dbUrl` and `dbDriver` are required; `dbUser`, `dbPassword`, `dbVendor`, `dbSchema`, `dbMaxPoolSize` as in the `database` block above. |
| `task-scheduler.scheduler.*` | — | `schedulerInstanceName`, `threadCount`, `maxConcurrentExecutionsPerTenant`, `maxParallelWorkersPerJob`, `hotReloadEnabled`, `eventTriggerRefreshSeconds`, with the same defaults as the `scheduler` block. `metricsHttpPort` is not read: metrics go through the runtime's own `/metrics`. |
| `task-scheduler.admin.*` | — | `httpPort`, `platformRole`, `tenantRole`, as in the `admin` block. The endpoint uses the host's `security` configuration. |

There is no `jobs` block in embedded mode: the jobs root is always `jobs/` under the runtime's base directory (`--bundle-path`), and tenant and platform databases come from the host's already-loaded configuration. There is also no `task-scheduler.queue` block: TSS's own broker connection is configured in embedded mode through the `TSS_QUEUE_*` environment variables/system properties only, exactly as every other PayOS component's own configuration stays outside `payos.json` when it needs a value `${...}`-expansion and hot reload don't apply to — see [`queue` block](#queue-block--tsss-own-broker-connection) above.

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

## Trigger types — every option, every example

A job's `triggers` array (see the `jobs.json` schema below) mixes any number of these six types freely — a job can declare several `cron` entries, a `queue` entry and its own `manual` override all at once. Every entry needs `type`; `id` defaults to `<type>-<index>` (its position in the array, e.g. `cron-0`) when omitted, and two entries of the same job can never end up with the same id, declared or defaulted. Leaving `triggers` out entirely, or giving it an empty array, still lets the job run: TSS adds an enabled `manual` trigger with no arguments allowed for you, unless you declare your own `manual` entry.

### `cron` — Quartz cron expression

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `expression` | Yes | — | A Quartz cron expression (6 or 7 fields, e.g. `0 0 2 * * ?`); an invalid one is rejected when the job is loaded. |
| `timeZone` | No | the server's own time zone | IANA zone id (e.g. `Africa/Casablanca`) the expression is evaluated in; an unknown zone id is rejected. |

Misfire policy: Quartz's own default for cron — fire once as soon as possible after a missed firing, then resume the regular schedule (no burst of catch-up runs).

```json
{ "id": "nightly", "type": "cron", "expression": "0 0 2 * * ?", "timeZone": "Africa/Casablanca" }
```

### `interval` — fixed delay

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `every` | Yes | — | ISO-8601 duration (e.g. `PT5M`), at least one second; fires every `every`, starting when the trigger is first scheduled. |

Misfire policy: a missed firing is skipped — the schedule resumes at the next due time, never a burst of catch-up runs.

```json
{ "id": "every-5min", "type": "interval", "every": "PT5M" }
```

### `once` — a single future firing

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `at` | Yes | — | ISO-8601 instant/offset date-time; a date already in the past is simply never scheduled (not an error). |

Misfire policy: fire as soon as possible — a node that was down at `at` still runs it once it comes back up.

```json
{ "id": "migration", "type": "once", "at": "2026-12-01T03:00:00+01:00" }
```

### `manual` — administrator-triggered only

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `enabled` | No | `true` | `false` refuses every manual trigger request for this job (CLI or HTTP), whatever the quota or arguments say. |
| `allowArguments` | No | `false` | `true` lets a manual request pass arguments (CLI `--arg key=value` / the HTTP body's `arguments`), reaching the script as `context.getArguments()`. |

Present on every job whether declared or not (see above) — its `id` is always `manual` (declaring any other id for it is rejected); only declare it yourself to change `enabled`/`allowArguments` away from their defaults. Nothing to subscribe to: an administrator asks for it, through the CLI `trigger` command or `POST /tss/admin/jobs/trigger`.

```json
{ "type": "manual", "enabled": true, "allowArguments": true }
```

### `queue` — fires on a broker message

Needs TSS's own queue connection configured first — see the [`queue` block](#queue-block--tsss-own-broker-connection) above; a declared `queue` trigger simply fails to start (logged, retried at the next event-trigger refresh) until it is.

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `destination` | Yes | — | The queue/subject TSS subscribes to, on its own broker connection. |
| `attributeFilter` | No | *(none — every message matches)* | Object of key/value pairs; only a message whose metadata contains every one of them is accepted — any other message is acknowledged (never redelivered) without running the job. |
| `maxPayloadBytes` | No | `65536` (64 KiB) | A larger message is acknowledged and logged as a warning, without running the job. |

A tenant job only accepts a message whose metadata carries a `tenantId` entry equal to its own tenant — a message with none, or a different one, is not for it; a platform job (no tenant) accepts every message on the destination, its deliberate cross-tenant integration point. See the consumer-group limitation noted under the [`queue` block](#queue-block--tsss-own-broker-connection) above before putting two different jobs' `queue` triggers on the same destination.

```json
{ "id": "on-settled", "type": "queue", "destination": "payments.settled", "attributeFilter": { "currency": "MAD" }, "maxPayloadBytes": 65536 }
```

### `dependency` — fires after named jobs complete

| Field | Required | Default | Purpose |
| --- | --- | --- | --- |
| `upstream` | Yes | — | Array of upstream job ids, of the same tenant as this job; a platform job's id is written `platform:<id>`. |
| `on` | No | `SUCCESS` | `SUCCESS` counts only a successful run of the upstream job; `COMPLETED` also counts one that ended in a script error. |
| `maxAge` | No | *(no limit)* | ISO-8601 duration; an upstream completion older than this does not count towards firing. |

Fires once every upstream job in `upstream` has completed (per `on`) since this job last ran — tracked cluster-wide on a shared "attendance sheet" (`tss_dependency_join`/`tss_dependency_state`), so it fires exactly once regardless of which node each upstream completion lands on. Several completions of the same upstream job before the sheet fills count as one (the latest is kept); while a run it already fired is still waiting to start, a second full sheet does not queue another one.

```json
{ "id": "after-ab", "type": "dependency", "upstream": ["job-a", "job-b"], "on": "SUCCESS", "maxAge": "PT1H" }
```

## `jobs/<tenantId>/jobs.json` — per-tenant jobs and policy

One file per tenant directory under the jobs root. Besides the top-level `maxConcurrentExecutions` policy field, it declares an array of job entries. Only `implementation.type: "script"` entries are loaded. Each entry's `triggers` array accepts any mix of `cron`/`interval`/`once`/`manual`/`queue`/`dependency` — see [Trigger types — every option, every example](#trigger-types--every-option-every-example) above for every type's options and a worked example each. The former single `trigger: {...}` object is no longer accepted: an entry still using it is rejected as a whole, with a warning naming it; every other entry in the file still loads.

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
| `triggers` | No | Array of triggers (`cron`/`interval`/`once`/`manual`/`queue`/`dependency`) — absent or empty means the job only runs when triggered by hand. Every type's fields and one example each: [Trigger types — every option, every example](#trigger-types--every-option-every-example) above. |
| `arguments` | No | Free-form JSON object handed to the script as `context.getArguments()`. Absent means an empty object; a non-object value is logged and ignored. Stored with the job in Quartz and kept up to date by reconciliation, so an edit reaches every cluster node through hot reload or `sync`, without a restart. A string value can carry a `${VAR}`/`${VAR:-default}`/`${file:/path}` token — see [Secrets, encryption and license validation (standalone mode)](#secrets-encryption-and-license-validation-standalone-mode) below. See [developer guide §6.1](../developer/task-scheduler-guide-v22-2026-11-20.md#61-passing-arguments-to-a-script-arguments). |
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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 2 * * ?" }],
      "parallelizable": false
    },
    {
      "id": "cleanup-hourly",
      "implementation": {
        "type": "script",
        "script": "jobs/cleanup.js"
      },
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 * * * ?" }],
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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 30 2 * * ?" }],
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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 3 * * ?" }],
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
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 2 * * ?" }],
      "arguments": { "folder": "/exports/{{tenantId}}", "tenant": "{{tenantId}}" }
    }
  ]
}
```

`{{tenantId}}` is the only placeholder `template.json` itself understands; any other `{{...}}` is left as written, with a warning in the log. This is a different mechanism from, and runs before, the `${...}` token resolution covered in [Secrets, encryption and license validation (standalone mode)](#secrets-encryption-and-license-validation-standalone-mode) below: `{{tenantId}}` is substituted once, into the materialized per-tenant `jobs.json` text, at provisioning time; `${...}` tokens in that same text are then resolved every time `jobs.json` is actually loaded (startup, hot reload, `sync`) — the two never collide, and a `template.json` can freely combine both, e.g. `"folder": "/exports/{{tenantId}}", "apiKey": "${EXPORT_API_KEY}"`.

- Only ever creates: an existing tenant directory or file is never modified, and a later edit of `template.json` does not reach tenants already provisioned.
- A `template.json` that is not a valid JSON object is never copied (an error is logged, the directory is created empty), so it cannot break the loading of the jobs files.
- Nothing is created when the jobs root itself is missing, a tenant id is not a plain directory name (letters, digits, `.`, `_`, `-`), or a tenant is named `platform`.
- Apart from `{{tenantId}}`, the template's text is kept exactly as written. A new tenant's jobs are scheduled at the next node restart (see [operations §Safety rules](../operations/task-scheduling-system-v22-2026-11-20.md#safety-rules)).

## Cluster recovery

Every job is registered with Quartz's `requestRecovery(true)` — if a cluster node dies mid-execution, a surviving node re-fires that job as a new, separate execution. This is not independently configurable; it's always on for every job. The recovery execution also closes out its crashed predecessor's row (outcome `ABANDONED`) the moment it starts — otherwise that row would sit `completed_at IS NULL` forever and keep counting against the tenant's concurrency quota indefinitely. See [developer/task-scheduler-guide-v22-2026-11-20.md#11-multiple-attempts-for-the-same-trigger-firing](../developer/task-scheduler-guide-v22-2026-11-20.md#11-multiple-attempts-for-the-same-trigger-firing) for what you'll observe when it happens.

## Next

- [Operations: Task Scheduling System](../operations/task-scheduling-system-v22-2026-11-20.md) — deployment modes, clustering, starting/stopping, tenant/platform database resolution, the shared script pool, execution-history table reference and example queries.
- [Developer guide: Task Scheduling System](../developer/task-scheduler-guide-v22-2026-11-20.md) — the full developer guide: configuring TSS and tenant jobs, writing script jobs, the `context`/`$DB`/`$PlatformDB`/`$Parallel` surface, and execution outcomes & history.
- [Architecture: Task Scheduling System class diagram](../architecture/job-scheduler-class-diagram-v24-2026-11-15.md) — internal design (French).
