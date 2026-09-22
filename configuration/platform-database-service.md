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
| `mapping-files` | Hibernate mapping files (`.hbm.xml`) for the entities living in this database. **Required** — startup fails if `platform-database-service` is present but resolves to zero mapping files. Unlike `database-service`, there is no per-application `model/` directory convention here: list every file explicitly. |

## This block is entirely optional

If `platform-database-service` is absent from configuration, `$PlatformDB` is simply not injected into scripts — startup does not fail, but `DatabaseServiceInitializer` logs a `WARN` line to flag that the block is missing. Add the block, or ignore the warning, depending on whether you actually have tenant-independent reference data to serve.

## Nothing here is tenant-scoped

`database-service` derives its effective schema/isolation from the [`multitenancy`](multi-tenancy.md) block per tenant. `platform-database-service` does not — it is one connection pool, one schema, one Hibernate mapping set, for the whole runtime's lifetime, regardless of which tenant is making the request. See [architecture/multi-tenancy.md](../architecture/multi-tenancy.md#platform-scoped-database-platformdb) for why this is a separate database connection rather than a per-provider tenant-filtering flag.

## Hot reload

`platform-database-service` participates in [hot reload](../operations/hot-reload.md), but **unlike** `database-service`: the previous connection pool is closed **immediately**, synchronously, with no `retired-session-factory-close-delay-seconds`-style grace period to let in-flight `$PlatformDB` requests drain. Avoid reloading configuration that changes `platform-database-service` while requests actively using `$PlatformDB` are in flight.

## Next

- [Developer: platform database usage (`$PlatformDB`)](../developer/platform-database-usage.md)
- [Configuration: database service](database-service.md)
- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
