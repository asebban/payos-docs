# Data architecture

How PayOS provides **multi-tenant data access** to applications. The data layer is a
pluggable service behind the kernel SPI `IDatabaseService`, implemented by the
`database-service` module (`DynamicDataAccessService`).

## The SPI

The kernel defines two interfaces (`ma.s2m.payos.database`):

```java
interface IDatabaseService { /* tenant-scoped data access + request scope lifecycle */ }

interface IDatabaseServiceFactory {
    IDatabaseService create(Map<String, DatabaseTenantDefinition> tenantDefinitions);
}
```

Unlike the other services, the database service has **no `type` discriminator** — it is
kernel-managed: `DatabaseServiceInitializer` reads the configuration, builds the per-tenant
definitions, and calls the factory.

## The implementation

| Aspect | Detail |
| --- | --- |
| Module | `database-service` |
| Artifact | `ma.s2m:dynamic-database-service` |
| Factory | `ma.s2m.DynamicDatabaseServiceFactory` |
| Service | `ma.s2m.DynamicDataAccessService` |
| ORM | Hibernate `5.6.15.Final` |

`DynamicDataAccessService` provides dynamic, mapping-driven data access. The query APIs,
transaction handling, and multi-tenant modes are documented in the module's own reference,
`database-service/docs/DynamicDataAccessService.md`, and from the developer's perspective in
[developer/data-access.md](../developer/data-access.md).

## Initialization and registration

```
BootServer
  └─ DatabaseServiceInitializer.initialize(PayOSConfig.settings)
        ├─ read "database-service" config (global) and per-tenant overrides
        ├─ build Map<String, DatabaseTenantDefinition>
        ├─ DynamicDatabaseServiceFactory.create(tenantDefs)
        └─ PayOSConfig.setDatabaseService(service)
```

The service is then exposed to scripts as `$DB` (injected by `ApiResourceHandler` only when
a database service is configured). Data sources per application are cached in
`PayOSConfig.dataSources`.

A detailed description of the database initialization process is described [here](./database-initialization-process.md)

## Configuration

Database configuration lives under `database-service` → `configuration`, both at the global
bootstrap level and per tenant under `multitenancy.tenants[].database-service`:

| Key | Purpose |
| --- | --- |
| `dialect` | Hibernate SQL dialect. |
| `driver_class` | JDBC driver class. |
| `url` | JDBC URL. |
| `username`, `password` | Credentials (prefer environment/secret substitution). |
| `schema` | Default schema. |
| `max-pool-size` | Connection pool maximum. |
| `minimum-idle` | Connection pool minimum idle. |
| `ddl-auto` | Schema management strategy. |
| `retired-session-factory-close-delay-seconds` | Grace period before closing a replaced session factory after reconfiguration. |

Full key documentation: [configuration/database-service.md](../configuration/database-service.md).

> The JDBC driver itself is **not** bundled in the kernel. Provide it as a connector JAR in
> `connectors-dir` (see [extensibility.md](extensibility.md)).

## Multi-tenant data access

The pipeline binds the resolved tenant to the service for the request's lifetime:

```
databaseService.setCurrentTenant(tenantId);
databaseService.beginRequestScope();
   // every $DB operation in the script is scoped to this tenant
databaseService.endRequestScope();
databaseService.clearCurrentTenant();
```

Per-tenant schemas and isolation modes (configured under `multitenancy`) let a single runtime serve many tenants with the appropriate data separation. The architectural rules for tenant isolation are in [multi-tenancy.md](multi-tenancy.md).

### Session-factory lifecycle on reconfiguration

Because configuration can be hot-reloaded (see [runtime-architecture.md](runtime-architecture.md)), a tenant's session factory may be replaced at runtime. The `retired-session-factory-close-delay-seconds` setting keeps the old factory open briefly so in-flight requests can complete before it is closed.

## Mapping files

Applications declare their data model through `mapping-files` (an array of paths) at the application level or simply put mapping files in the `model` directory. These drive the dynamic data access. Application-level data configuration is part of the application model — see [developer/application-model.md](../developer/application-model.md).

## Next

- [Developer: data access](../developer/data-access.md) — using `$DB` from a script.
- [Configuration: database service](../configuration/database-service.md) — every key.
