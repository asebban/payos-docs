Created: 2026-10-18
Last updated: 2026-10-19
Version: 2 — supersedes v1: links to the new end-to-end [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md).

# Database service configuration

The `database-service` block configures the Hibernate-based database connector
(`dynamic-database-service`), which exposes `$DB` to scripts. Developer usage is in
[developer/data-access.md](../developer/data-access.md); the architectural model is in
[architecture/data-architecture.md](../architecture/data-architecture.md).

## Shape

```json
{
  "database-service": {
    "dialect": "org.hibernate.dialect.PostgreSQLDialect",
    "driver_class": "org.postgresql.Driver",
    "url": "jdbc:postgresql://db:5432/payos",
    "username": "payos",
    "password": "${DB_PASSWORD}",
    "schema": "public",
    "max-pool-size": 20,
    "minimum-idle": 5,
    "ddl-auto": "validate",
    "retired-session-factory-close-delay-seconds": 60
  }
}
```

## Keys

From `IConfigSpec.DatabaseService`:

| Key | Purpose |
| --- | --- |
| `dialect` | Hibernate SQL dialect. |
| `driver_class` | JDBC driver class. |
| `url` | JDBC connection URL. |
| `username` | DB user. |
| `password` | DB password (use `${ENV}` / encryption). |
| `schema` | Default schema. |
| `max-pool-size` | Maximum connection pool size. |
| `minimum-idle` | Minimum idle connections. |
| `ddl-auto` | Schema management mode (e.g. `validate`, `update`, `none`). |
| `retired-session-factory-close-delay-seconds` | Delay before closing a retired session factory after a [hot reload](../operations/hot-reload.md). |

## The JDBC driver is a service-adapter dependency

The service-adapter JAR (`dynamic-database-service`) and the **JDBC driver** must be available on
the [service-adapters path](extensions-connectors.md). The kernel does not bundle drivers — choose
the driver matching your `url`/`dialect`.

## Multi-tenancy interaction

Per-request tenant binding is automatic. The effective schema and isolation come from the
[`multitenancy`](multi-tenancy.md) block (`default-database-schema`,
`default-isolation-mode`, and per-tenant overrides), not from script code. The kernel opens a
request scope before injecting `$DB` and closes it in the pipeline's `finally` stage.

## Hot reload

Database configuration participates in [hot reload](../operations/hot-reload.md): on change,
services are reinitialized and the previous session factory is retired after
`retired-session-factory-close-delay-seconds` to drain in-flight work.

## Per-application override

An application may carry its own `database-service` object to use a different database than
the global one. See [developer/application-model.md](../developer/application-model.md).

## Reference/master data shared by every tenant

`database-service` is always tenant-scoped — see [Multi-tenancy interaction](#multi-tenancy-interaction) above. For data that must **not** be duplicated per tenant (a country or currency table), see [platform-database-service.md](platform-database-service.md) — a separate, optional database backing `$PlatformDB`.

## Mapping files actually come from `applications[]`

The `database-service` block above configures the **connection** (dialect, URL, pool, `ddl-auto`) — it has no `mapping-files` key of its own for a tenant's main `$DB`. A tenant's actual Hibernate mapping-file set is resolved by unioning the `mapping-files` array (root-level key, with `extends` inheritance) of every `applications[]` entry that tenant is authorized for — see [configuration/json-configuration-reference.md §3.2](json-configuration-reference.md). This matters whenever a module needs its own entity resolvable on a tenant's `$DB` without a kernel mapping of its own — for example, `payos-database-audit-trail`/`payos-server-queue` registering `PayosAuditEvent`/`PayosQueueOutbox` for `$Audit`/`$Queue`: see [configuration/audit-trail.md §Required: register the mapping file](audit-trail.md#required-register-the-mapping-file) and [configuration/servers.md §Required: register the outbox mapping file](servers.md#required-register-the-outbox-mapping-file). The per-application `database-service.mapping-files` key shown in that reference page's example is a **different** thing — it only applies when that one application opts into its own dedicated, non-shared database.

## `$DB` can be entirely absent on an `atomicityScope: "platform"` endpoint

An endpoint's `atomicityScope` field (`mappings.json`, default `"tenant"`) decides which of `$DB`/`$PlatformDB` carries `$Audit`/`$Queue`'s atomicity guarantee. An endpoint declared `atomicityScope: "platform"` gets no `$DB` at all — not a read-only version, `typeof $DB === 'undefined'` — since it has no declared reason left to touch the tenant database. This is independent of whether `database-service` itself is configured. See [architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v10-2026-10-20.md) for the full mechanism.

## Next

- [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md) — ordered end-to-end setup for `$DB`/`$PlatformDB` + `$Audit` + `$Queue`, including the `applications[].mapping-files` step above.
- [developer/data-access.md](../developer/data-access.md)
- [architecture/data-architecture.md](../architecture/data-architecture.md)
- [architecture/atomicity-db-audit-queue-v10-2026-10-20.md](../architecture/atomicity-db-audit-queue-v10-2026-10-20.md) — `atomicityScope` and the `$DB`/`$PlatformDB` lockdown.
- [configuration/platform-database-service.md](platform-database-service.md) — the tenant-independent counterpart, for reference/master data.
