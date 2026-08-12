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

## Next

- [developer/data-access.md](../developer/data-access.md)
- [architecture/data-architecture.md](../architecture/data-architecture.md)
