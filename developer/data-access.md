# Data access (`$DB`)

When a [database service](../configuration/database-service.md) is configured, scripts receive the `$DB` binding — an `IDatabaseService` backed by the `dynamic-database-service` connector (Hibernate-based). This page covers using `$DB` from JavaScript; the connector's configuration keys are in [configuration/database-service.md](../configuration/database-service.md), and the architectural model is in [architecture/data-architecture.md](../architecture/data-architecture.md).

## Tenant scoping is automatic

You never pass a tenant to `$DB`. The kernel binds the current tenant to the database service for the lifetime of the request (`setCurrentTenant` + `beginRequestScope`), so all access is scoped to the request's tenant and isolation mode automatically. See [multi-tenancy](../architecture/multi-tenancy.md).

## Querying

```javascript
function execute(request, controlData) {
    var accounts = $DB.find(
        "SELECT id, balance FROM account WHERE status = :status",
        {status: "ACTIVE"}
    );
    return { count: accounts.length, accounts: accounts };
}
```

## Writing

```javascript
function execute(request, controlData) {
    var id = $DB.save("account", {
        owner: $Principal.get("id"),
        balance: 0
    });
    $Response.setStatusCode(201);
    return { id: id };
}
```

> The exact method surface (`find`, `save`, `update`, `delete`, named operations) is provided by `DynamicDataAccessService`. See the connector's own reference, `database-service/docs/ DynamicDataAccessService.md`, for the complete API.

## Data-model mapping files

An application can declare entity/data-model mappings through `mapping-files` in its [application configuration](application-model.md). These describe how logical entities map to the underlying schema, letting `$DB` resolve named operations. It is also possible to just put by convention the mapping files in `model/` sub-directory of the application.

## Transactions and request scope

Each request opens a database **request scope** before bindings are injected and closes it in the `finally` stage of the pipeline (`endRequestScope`). Work performed via `$DB` during the request participates in that scope; you do not manage connections manually.

## Schema and isolation

The schema and isolation behavior come from configuration, not script code:

- `database-service.schema` / per-tenant `default-database-schema`,
- `multitenancy.default-isolation-mode`,
- connection pool sizing (`max-pool-size`, `minimum-idle`),
- `ddl-auto` for schema management.

These are documented in
[configuration/database-service.md](../configuration/database-service.md) and
[configuration/multi-tenancy.md](../configuration/multi-tenancy.md).

## If `$DB` is missing

`$DB` is injected only when a database service is configured. If your script references it
without a configured connector, the binding will be absent. Configure the
[database service](../configuration/database-service.md) and ensure the
`dynamic-database-service` connector JAR (and its JDBC driver) are on the
[connectors path](../configuration/extensions-connectors.md).

## Next

- [Configuration: database service](../configuration/database-service.md)
- [Architecture: data architecture](../architecture/data-architecture.md)
