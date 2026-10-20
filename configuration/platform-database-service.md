Created: 2026-10-18
Last updated: 2026-10-19
Version: 2 — supersedes v1: links to the new end-to-end [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md).

# Platform database service configuration

The `platform-database-service` block configures an optional, second Hibernate-based database connection — separate from [`database-service`](database-service.md) — that exposes `$PlatformDB` to scripts. Unlike `database-service`, it is **not** tenant-scoped: there is exactly one connection, shared by every tenant, and the runtime never injects or filters by `tenantId` on it. It exists for reference/master data that must not be duplicated per tenant (a country list, a currency table, an ISO code lookup). Developer usage is in [developer/platform-database-usage.md](../developer/platform-database-usage.md); the architectural reasoning is in [architecture/multi-tenancy.md](../architecture/multi-tenancy.md#platform-scoped-database-platformdb).

## Shape

```json
{
  "platform-database-service": {
    "configuration": {
      "dialect": "org.hibernate.dialect.PostgreSQLDialect",
      "driver_class": "org.postgresql.Driver",
      "url": "jdbc:postgresql://db:5432/payos_platform",
      "username": "payos_platform",
      "password": "${PLATFORM_DB_PASSWORD}",
      "schema": "platform",
      "max-pool-size": 10,
      "minimum-idle": 2,
      "ddl-auto": "validate"
    },
    "mapping-files": ["model/Country.hbm.xml", "model/Currency.hbm.xml"]
  }
}
```

## Keys

The `configuration` object accepts the same keys as [`database-service.configuration`](database-service.md#keys) (`dialect`, `driver_class`, `url`, `username`, `password`, `schema`, `max-pool-size`, `minimum-idle`, `ddl-auto`) — see that page for what each one does. There is no `retired-session-factory-close-delay-seconds` equivalent yet and no per-tenant `tenants[]` override, since this block is not tenant-scoped at all.

| Key | Purpose |
| --- | --- |
| `configuration.*` | Same keys as `database-service.configuration` — see [database-service.md](database-service.md#keys). |
| `mapping-files` | Hibernate mapping files (`.hbm.xml`) for the entities living in this database. **Required** — startup fails if `platform-database-service` is present but resolves to zero mapping files. Unlike `database-service`, there is no per-application `model/` directory convention here: list every file explicitly. This is also where `payos-database-audit-trail`/`payos-server-queue` must be registered for `$Audit`/`$Queue` to work on `atomicityScope: "platform"` endpoints — see [configuration/audit-trail.md §Required: register the mapping file](audit-trail.md#required-register-the-mapping-file) and [configuration/servers.md §Required: register the outbox mapping file](servers.md#required-register-the-outbox-mapping-file). |

## This block is entirely optional

If `platform-database-service` is absent from configuration, `$PlatformDB` is simply not injected into scripts — startup does not fail, but `DatabaseServiceInitializer` logs a `WARN` line to flag that the block is missing. Add the block, or ignore the warning, depending on whether you actually have tenant-independent reference data to serve.

## Nothing here is tenant-scoped

`database-service` derives its effective schema/isolation from the [`multitenancy`](multi-tenancy.md) block per tenant. `platform-database-service` does not — it is one connection pool, one schema, one Hibernate mapping set, for the whole runtime's lifetime, regardless of which tenant is making the request. See [architecture/multi-tenancy.md](../architecture/multi-tenancy.md#platform-scoped-database-platformdb) for why this is a separate database connection rather than a per-provider tenant-filtering flag.

## `$PlatformDB` is read-only unless an endpoint opts into it

Configuring this block makes `$PlatformDB` available, but by default it's a **read-only** subset (`get`/`getFresh`/`findAll`/`find`/`list`/`unique`/`first`/`newParams` — no writes). Full read/write access is granted only to an endpoint that declares `atomicityScope: "platform"` in its `mappings.json` entry — the same field that also removes `$DB` from that endpoint entirely. This is enforced by the platform, not left to application convention. See [developer/platform-database-usage.md](../developer/platform-database-usage.md#atomicityscope-and-the-read-onlyfull-split) and [architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md).

## Hot reload

`platform-database-service` participates in [hot reload](../operations/hot-reload.md), but **unlike** `database-service`: the previous connection pool is closed **immediately**, synchronously, with no `retired-session-factory-close-delay-seconds`-style grace period to let in-flight `$PlatformDB` requests drain. Avoid reloading configuration that changes `platform-database-service` while requests actively using `$PlatformDB` are in flight.

## Next

- [Developer: atomicity setup guide (de A à Z)](../developer/atomicity-setup-guide.md) — ordered end-to-end setup for `$DB`/`$PlatformDB` + `$Audit` + `$Queue`.
- [Architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md) — the `atomicityScope` field and the read-only/full split.
- [Developer: platform database usage (`$PlatformDB`)](../developer/platform-database-usage.md)
- [Configuration: database service](database-service.md)
- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
