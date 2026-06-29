# Notification service (`payos-service-notification`)

Operator guidance for running `payos-service-notification`: a standalone daemon that consumes notification requests from a message queue and delivers them via channel adapters (email in the MVP), with retry, fallback-channel escalation, and a push status callback to the publisher.

Unlike the main PayOS runtime (`payos-runtime`/`BootServer`), this is its **own deployable process** with its own `main()` — it does not implement `IServer`/`ServerProvider`, is not booted from a bundle's `connectors-dir`/`extensions-dir`, and has no HTTP surface. The message queue is its only ingress. If you're looking for the main runtime's deployment model, see [deployment.md](deployment.md) instead — this page is specific to the notification daemon.

> **Not to be confused with** the `notification-service` *bootstrap.json* block — that one configures the `payos-notification-connector` running inside `BootServer`, which **publishes** notification requests onto the queue that this daemon consumes. See [configuration/notification-service.md](../configuration/notification-service.md) for the publisher side.

## Artifacts

| Artifact | Role |
| --- | --- |
| `payos-service-notification-<version>.jar` | Self-contained runnable JAR (built with  `maven-shade-plugin`) — includes the NATS broker adapter, the database connector, and the H2/ PostgreSQL JDBC drivers. Main class `ma.s2m.payos.notification.NotificationDaemon`. |

There is no separate connectors/extensions directory for this service — every backend it can use (NATS, Hibernate/H2/PostgreSQL) is already bundled into the JAR at build time as an ordinary Maven dependency, not resolved via `ServiceLoader`/`connectors-dir` at runtime the way the main runtime's connector JARs are.

## Building

```bash
cd payos-service-notification
mvn clean package
```

This produces `target/payos-service-notification-<version>.jar`, ready to run directly.

## Starting the service

```bash
java -jar payos-service-notification-<version>.jar
```

With no configuration at all, the daemon starts against sensible local defaults: NATS on `localhost:4222`, an in-memory H2 database, and `localhost:25` for SMTP — useful for a quick local smoke test, not for production.

### As a service (systemd example)

```ini
[Unit]
Description=PayOS Notification Service
After=network.target

[Service]
ExecStart=/usr/bin/java -jar /opt/payos-notification/payos-service-notification-<version>.jar
Restart=on-failure
Environment=NOTIFICATION_QUEUE_HOST=nats.internal
Environment=NOTIFICATION_QUEUE_PORT=4222
Environment=NOTIFICATION_DB_URL=jdbc:postgresql://db.internal:5432/notifications
Environment=NOTIFICATION_DB_USERNAME=notification_svc
Environment=NOTIFICATION_DB_PASSWORD=${NOTIFICATION_DB_PASSWORD}
Environment=NOTIFICATION_SMTP_HOST=smtp.internal

[Install]
WantedBy=multi-user.target
```

### Stopping

The daemon registers a JVM shutdown hook (`main()` calls `Runtime.getRuntime().addShutdownHook(...)`), so a normal `SIGTERM` (`systemctl stop`, plain `kill <pid>`, container stop) triggers a graceful shutdown: it cancels the queue subscription and disconnects the status-callback  publisher before exiting. Avoid `kill -9`/`SIGKILL` where possible — it skips the shutdown hook entirely.

## Configuration

Every setting is a value in `ConfigParam`, resolved from four sources, in priority order:

1. **Command-line arguments** — `--notification-queue-host value` or    `--notification-queue-host=value` (the flag is the env var name, lowercased and 
   dash-separated).
2. **Environment variables** — e.g. `NOTIFICATION_QUEUE_HOST`.
3. **A bundle config file** — `notification.json` inside the directory passed via    `--bundle-path`; a flat JSON object keyed by the same names as the environment variables (see [Bundle config file](#bundle-config-file) below). `--bundle-path` defaults to the current working directory when omitted — in that case a missing `notification.json` is not an error, it just means this source contributes nothing. If you pass `--bundle-path` explicitly, a missing file *is* an error (almost certainly a typo).
4. **Built-in default.**

The first source that has a value for a given key wins; there is no merging between sources for
an individual key.

### Queue (broker connection)

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_QUEUE_HOST` | `--notification-queue-host` | `localhost` | Broker host. |
| `NOTIFICATION_QUEUE_PORT` | `--notification-queue-port` | `4222` | Broker port. |
| `NOTIFICATION_QUEUE_DESTINATION` | `--notification-queue-destination` | `notifications` | Destination the daemon subscribes to for inbound notification requests. |
| `NOTIFICATION_QUEUE_BROKER_TYPE` | `--notification-queue-broker-type` | `nats` | Selects the `IQueueClientFactory` implementation by type, via `QueueClients.create(...)`. Only `nats` ships as a real broker today; other values require a registered connector on the classpath. |

### Database (primary store)

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_DB_URL` | `--notification-db-url` | `jdbc:h2:mem:notification;DB_CLOSE_DELAY=-1` | JDBC URL. Defaults to an in-memory H2 instance; point at PostgreSQL in production. |
| `NOTIFICATION_DB_USERNAME` | `--notification-db-username` | `sa` | Database username. |
| `NOTIFICATION_DB_PASSWORD` | `--notification-db-password` | *(empty)* | Database password. Prefer the environment variable over a CLI flag/bundle file so it doesn't appear in `ps`/shell history or get checked into a bundle. |
| `NOTIFICATION_DB_DRIVER` | `--notification-db-driver` | `org.h2.Driver` | JDBC driver class. Use `org.postgresql.Driver` for PostgreSQL. |
| `NOTIFICATION_DB_DIALECT` | `--notification-db-dialect` | `org.hibernate.dialect.H2Dialect` | Hibernate dialect. Use `org.hibernate.dialect.PostgreSQLDialect` for PostgreSQL. |
| `NOTIFICATION_DB_DDL_AUTO` | `--notification-db-ddl-auto` | `update` | Hibernate `hbm2ddl.auto` setting. |
| `NOTIFICATION_DB_SCHEMA` | `--notification-db-schema` | *(none)* | Database schema, if your deployment uses a non-default one. |

Both the H2 and PostgreSQL JDBC drivers are already bundled in the runnable JAR — no separate driver placement is needed when switching `NOTIFICATION_DB_DRIVER`/`NOTIFICATION_DB_DIALECT`.

### Filesystem storage fallback

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_STORE_FILESYSTEM_DIR` | `--notification-store-filesystem-dir` | `.` (current working directory) | Base directory for the filesystem-backed store, used automatically when the database is unavailable. |

At startup, the daemon probes the database (resolves the `IDatabaseServiceFactory` and opens a test JDBC connection using the settings above). If the database cannot be resolved or the connection fails, it logs a `WARN` and falls back to a filesystem-backed store instead of failing to start — one JSON file per notification, laid out as `{dir}/{tenantId}/{messageId}.json`, fully isolated per tenant. All the same operations (create, status update, attempt recording, status lookup) work the same way against either store; the only operator-visible difference is the log line announcing which one was selected. There is no automatic migration between the two stores — if the database comes back later, already-fallback-recorded notifications stay on disk until you migrate them manually.

The default writes directly into the daemon's working directory (e.g. `tenant-a/msg-123.json` next to wherever the JAR was launched from) — fine for a quick local run, but set `NOTIFICATION_STORE_FILESYSTEM_DIR` explicitly in production so fallback data lands in a dedicated, backed-up location rather than wherever the process happened to start. Ensure the configured directory (and its parent) is writable by the service's user, and is on a disk with enough headroom and durability guarantees for however long the database might stay down — this path has no retention/cleanup policy of its own.

Note that `--bundle-path` (see [Bundle config file](#bundle-config-file)) *also* defaults to the current working directory. Left at their defaults, both features read/write in the same place: the daemon looks for `notification.json` and writes `{tenantId}/{messageId}.json` records into the same directory. They don't conflict by name, but for clarity in production set both explicitly, ideally to different directories.

### Email channel (SMTP)

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_SMTP_HOST` | `--notification-smtp-host` | `localhost` | SMTP server host. |
| `NOTIFICATION_SMTP_PORT` | `--notification-smtp-port` | `25` | SMTP server port. |
| `NOTIFICATION_SMTP_USERNAME` | `--notification-smtp-username` | *(empty)* | SMTP auth username; SMTP auth is enabled automatically when this is non-blank. |
| `NOTIFICATION_SMTP_PASSWORD` | `--notification-smtp-password` | *(empty)* | SMTP auth password. Prefer the environment variable, same reasoning as the database password. |
| `NOTIFICATION_SMTP_FROM` | `--notification-smtp-from` | `notifications@payos.local` | `From` address on outgoing emails. |

`STARTTLS` is always requested (`mail.smtp.starttls.enable=true`) — there is no setting to disable it. Email is the only channel that ships in the MVP; the daemon rejects any other `channel` value at validation time (current allowlist: `email`).

### Retry policy

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_RETRY_MAX_ATTEMPTS` | `--notification-retry-max-attempts` | `3` | Max delivery attempts per channel before escalating to the next fallback channel (or terminal failure). |
| `NOTIFICATION_RETRY_INITIAL_BACKOFF_SECONDS` | `--notification-retry-initial-backoff-seconds` | `5` | Initial backoff before the first retry; backoff grows exponentially from this value. |

### Status callback

| Env var | CLI flag | Default | Purpose |
| --- | --- | --- | --- |
| `NOTIFICATION_STATUS_CALLBACK_DESTINATION` | `--notification-status-callback-destination` | `notifications.status` | Queue destination the daemon publishes terminal-outcome status callbacks to. |

Every notification that reaches a terminal outcome (`delivered` or `failed`) gets exactly one callback published here, containing `messageId`, `tenantId`, `status`, `channel`, and the full per-channel attempt history. Publishers subscribe to this single, fixed destination and filter to their own messages by `tenantId`/`messageId` — there is no per-publisher or per-tenant callback destination in the MVP.

### Bundle config file

If you prefer a file over individual environment variables, pass `--bundle-path <dir>` pointing
at a directory containing `notification.json` — a flat JSON object keyed by the environment
variable names above:

```json
{
  "NOTIFICATION_QUEUE_HOST": "nats.internal",
  "NOTIFICATION_QUEUE_PORT": "4222",
  "NOTIFICATION_DB_URL": "jdbc:postgresql://db.internal:5432/notifications",
  "NOTIFICATION_DB_DRIVER": "org.postgresql.Driver",
  "NOTIFICATION_DB_DIALECT": "org.hibernate.dialect.PostgreSQLDialect",
  "NOTIFICATION_SMTP_HOST": "smtp.internal"
}
```

```bash
java -jar payos-service-notification-<version>.jar --bundle-path /etc/payos-notification
```

If you pass `--bundle-path` explicitly and that directory has no `notification.json`, the daemon fails to start with a clear error rather than silently falling back to defaults — this is deliberate, since an explicit path with a missing file is almost certainly an operator typo. This strict check does **not** apply when `--bundle-path` is omitted: it then defaults to the current working directory, and a missing `notification.json` there is treated as "this source has nothing to contribute," not an error.

## Switching brokers

`NOTIFICATION_QUEUE_BROKER_TYPE` selects the broker by type via the same type-keyed `QueueClients.create(...)` lookup the database side uses for `IDatabaseServiceFactory`. This is purely configuration-driven — no code change is needed to point the daemon at a different registered broker adapter. In the current build, `nats` is the only real broker shipped; other values (e.g. a `stub-broker` type) only exist for test/extensibility purposes and are not production broker adapters.

## Logging

The daemon uses slf4j with logback. No `logback.xml` ships with the JAR — by default you get logback's built-in console configuration. To customize, put your own `logback.xml` on the classpath ahead of the JAR, or pass `-Dlogback.configurationFile=/path/to/logback.xml`. Every delivery-related log line includes `messageId`, `tenantId`, and `channel`, which is enough to reconstruct a notification's processing timeline without querying the database.

## Health and lifecycle

This service has no HTTP surface, so there is no `/health` endpoint to poll (unlike `payos-runtime` — see [deployment.md](deployment.md)). Monitor it the same way you'd monitor any plain JVM process: process liveness (`systemctl status`, container health checks against the  PID), and the startup log line confirming whether it selected the database-backed or  filesystem-backed store. Richer queue-consumer/delivery-rate metrics are planned (Epic 3) but not yet shipped.

## Multi-tenancy

`tenantId` is a required field on every published notification message — the daemon never infers it from the queue connection or any credential. There is nothing tenant-specific to configure on the operator side beyond the shared database/queue connection; tenant isolation happens at the data layer (a `tenant_id` row column for the Hibernate-backed store, a per-tenant subdirectory  for the filesystem-backed fallback).

## Next

- [deployment.md](deployment.md) — the main PayOS runtime's deployment model, for comparison.
- [secrets-management.md](secrets-management.md) — if/when this service's SMTP/DB credentials are moved to Vault-backed per-tenant secrets (not yet implemented).
- [observability.md](observability.md) — platform-wide logging/audit conventions this service   follows.
