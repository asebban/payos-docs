# Multi-tenancy configuration

The `multitenancy` block governs how tenants are required, resolved, isolated, quota-limited,
and (in development) simulated. The runtime model is in
[architecture/multi-tenancy.md](../architecture/multi-tenancy.md).

## Shape

```json
{
  "multitenancy": {
    "requireTenantId": true,
    "default-database-schema": "public",
    "default-isolation-mode": "schema",
    "default-tenant-quotas": {
      "requestsPerMinute": 600,
      "enabled": true
    },
    "tenantSimulator": {
      "enabled": false,
      "tenantId": "dev-tenant"
    },
    "tenants": [
      { "id": "acme", "database-schema": "acme", "isolation-mode": "schema",
        "quotas": { "requestsPerMinute": 1200, "enabled": true } }
    ]
  }
}
```

## Keys

From `IConfigSpec.MultiTenancy`:

| Key | Default | Purpose |
| --- | --- | --- |
| `requireTenantId` | `true` | Reject requests with no resolvable tenant. |
| `default-database-schema` | — | Default schema applied to tenants. |
| `default-isolation-mode` | — | Default isolation mode (e.g. schema-per-tenant). |
| `default-tenant-quotas.requestsPerMinute` | — | Default per-tenant rate limit. |
| `default-tenant-quotas.enabled` | — | Whether default quotas are enforced. |
| `tenantSimulator.enabled` | `false` | **Development only** — inject a fixed tenant. |
| `tenantSimulator.tenantId` | — | The simulated tenant id. |
| `tenants[]` | — | Per-tenant overrides (schema, isolation, quotas). |

## Tenant resolution

The tenant is resolved (in order):

1. `Request.contextData[CONTEXT_TENANT_ID]` (`"tenantId"`),
2. the `X-Tenant-Id` header (case-insensitive),
3. the MDC tenant (`TenantScope.MDC_TENANT_ID`).

If none resolves and `requireTenantId` is `true`, the request is rejected **before** business
logic runs. See [architecture/multi-tenancy.md](../architecture/multi-tenancy.md).

## The tenant simulator (development only)

`tenantSimulator` injects a fixed tenant so you can call APIs locally without sending
`X-Tenant-Id`:

```json
"tenantSimulator": { "enabled": true, "tenantId": "dev-tenant" }
```

> **Never enable the simulator in production.** It bypasses real tenant resolution and would
> collapse isolation.

## Per-tenant overrides

Each `tenants[]` entry can override the schema, isolation mode, and quotas for a specific
tenant, layering on top of the `default-*` values.

## Relationship to other blocks

- **Database:** `default-database-schema` and `default-isolation-mode` drive how `$DB`
  resolves the schema — see [database-service.md](database-service.md).
- **Security:** the authenticated tenant can be cross-checked against
  `authorized-tenants` on the application — see [security-oidc.md](security-oidc.md).

## Next

- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
- [database-service.md](database-service.md)
