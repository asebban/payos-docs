# Multi-tenancy

PayOS is **multi-tenant by design**: isolation is enforced by the architecture and the request pipeline, not left to per-application configuration. This document explains how a tenant is identified, how the tenant scope is opened, and how isolation and quotas are enforced.

## Tenant identity

A tenant is identified by the `X-Tenant-Id` header (constant `Request.HEADER_TENANT_ID = "X-Tenant-Id"`). It is stored alongside the correlation ID in the request `contextData` under `IServer.CONTEXT_TENANT_ID = "tenantId"`, and mirrored into the SLF4J **MDC** so every log line is tenant-scoped.

### Resolution order

`ApiResourceHandler.resolveCurrentTenantId(request)` checks, in order:

1. `request.getContextData().get(IServer.CONTEXT_TENANT_ID)` — already resolved upstream by the transport;
2. the `X-Tenant-Id` request header (case-insensitive);
3. the SLF4J MDC key `TenantScope.MDC_TENANT_ID`.

For authenticated requests the tenant can also be derived from the security principal via `ISecurityService.resolveAuthenticatedTenantId(request)` (see [security-architecture.md](security-architecture.md)).

## Opening the tenant scope at ingress

Every transport opens a tenant scope **before** any business logic runs:

| Transport | Authenticated request | Unauthenticated request |
| --- | --- | --- |
| HTTP/HTTPS | `TenantPolicyService.enforceAndOpenScope(request, appId)` | `openPreAuthTenantScope(request, appId)` |
| TCP | `TenantPolicyService.enforceAndOpenScope(request, appId)` | — |
| Queue | `TenantPolicyService.enforceAndOpenScope(request, appId)` | — |

- `enforceAndOpenScope` applies the **strict** tenant policy: it validates the tenant against the application's authorization rules and opens the scope, throwing a `TenantPolicyException` on violation (the transport then returns an error response with the correlation/tenant headers preserved).
- `openPreAuthTenantScope` opens a **minimal** scope for login flows. If `X-Tenant-Id` is   absent it consults the tenant simulator configuration; if `X-Correlation-Id` is absent it   generates a UUID. Both values are written into `contextData`.

## Isolation at the data layer

When the scope is open, the API pipeline binds the tenant to the database service for the duration of the request:

```
databaseService.setCurrentTenant(tenantId);
databaseService.beginRequestScope();
   ... script runs, all $DB access is tenant-scoped ...
databaseService.endRequestScope();
databaseService.clearCurrentTenant();
```

The database service supports per-tenant schemas and isolation modes. See [data-architecture.md](data-architecture.md) and [configuration/multi-tenancy.md](../configuration/multi-tenancy.md).

## Configuration model

Multi-tenancy is configured under the `multitenancy` block (`IConfigSpec.Multitenancy`):

| Key | Purpose |
| --- | --- |
| `requireTenantId` | Whether a tenant id is mandatory on every request. |
| `default-database-schema` | Default schema when a tenant does not override it. |
| `default-isolation-mode` | Default isolation strategy. |
| `default-tenant-quotas` → `requestsPerMinute`, `enabled` | Default rate limiting. |
| `tenantSimulator` → `enabled`, `tenantId` | Dev/test fallback tenant when `X-Tenant-Id` is missing. |
| `tenants[]` | Per-tenant overrides: schema, isolation mode, `tcp.handlers.dir`, quotas, and a per-tenant `database-service` configuration. |

Full key documentation is in [configuration/multi-tenancy.md](../configuration/multi-tenancy.md).

### Tenant authorization on applications

Applications can restrict which tenants may use them via `authorized-tenants` / `authorized.tenants` (an allowlist) in the application configuration (`IConfigSpec.Applications.Application`). The tenant policy uses this list when enforcing the scope.

## Quotas

Per-tenant request quotas (`requestsPerMinute`, `enabled`) are configured at the default level and overridable per tenant. When enabled, the policy layer rejects requests that exceed the configured rate.

## The tenant simulator (non-production)

`multitenancy.tenantSimulator` provides a fallback tenant for **development and testing only**, so a developer can exercise tenant-scoped behavior without sending `X-Tenant-Id`. It must not be enabled in production, where a real tenant id is always required.

## Platform-scoped database (`$PlatformDB`)

Everything above describes `$DB`: every read gets a `tenantId` predicate added, every write gets a `tenantId` column set, automatically (`DynamicDataAccessService.withTenantInEntity`/`applyTenantFilterToReadQuery`/`enforceTenantOnLoadedEntity` in the `database-service` connector). Some data is deliberately **not** tenant-scoped — reference/master data like a country or currency table, identical for every tenant, that would be wasteful and risky to duplicate per tenant. For that case, PayOS supports an optional second, genuinely separate database: [`platform-database-service`](../configuration/platform-database-service.md), reachable from scripts as [`$PlatformDB`](../developer/platform-database-usage.md).

This is a **separate physical connection**, not a flag that suppresses tenant filtering on `$DB` itself, for two concrete reasons:

- **The ambient tenant is process-wide, not per-connection.** `DynamicDataAccessService`'s current-tenant tracking (`CURRENT_TENANT`) is a `static` `ThreadLocal` — shared by every instance of the class running on the same thread. A second `IDatabaseService` instance sharing `$DB`'s connection would still pick up whatever tenant `$DB` set for the request unless it explicitly ignored that ambient state (which is exactly what the platform-scoped constructor does — see `DynamicDataAccessService`'s Javadoc on its `TenantConfig`-only constructor).
- **A single dedicated database keeps the request's transaction semantics honest.** `$DB` and `$PlatformDB` are independent transactions, each owned by the kernel's own request-scope lifecycle (`beginRequestScope`/`commitRequestScope`/`rollbackRequestScope`/`endRequestScope`, called for both in `ApiResourceHandler`, mirroring the [single-transaction-per-request model](../developer/data-access.md#transactions-and-request-scope) `$DB` already uses). By convention (not platform-enforced), a given request writes through only one of the two — see [developer/platform-database-usage.md](../developer/platform-database-usage.md#db-and-platformdb-are-independent-transactions) for what that does and doesn't guarantee.

Unlike `database-service`, `platform-database-service` has no per-tenant routing at all: one JDBC/Hikari connection pool, one `SessionFactory`, one Hibernate mapping set, built once at startup and reused for the runtime's whole lifetime, regardless of which tenant is making the request.

## Traceability

The combination of `X-Tenant-Id` and `X-Correlation-Id` is mandatory cross-transport metadata. Both are propagated unchanged through context, logs, responses, and async processing, and both appear in error responses and audit logs to support regulated incident investigation. See [operations/observability.md](../operations/observability.md).

## Next

- [Security architecture](security-architecture.md) — how authenticated tenants are derived.
- [Data architecture](data-architecture.md) — per-tenant schemas and isolation.
- [Configuration: platform database service](../configuration/platform-database-service.md) — `$PlatformDB`'s configuration.
- [Developer: platform database usage](../developer/platform-database-usage.md) — `$PlatformDB`'s API surface.
