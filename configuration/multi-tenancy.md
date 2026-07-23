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

Each `tenants[]` entry can override the schema, isolation mode, quotas, security settings,
and database configuration for a specific tenant, layering on top of the `default-*` values.

### `tenants[]` entry structure

| Key | Type | Default | Purpose |
| --- | --- | --- | --- |
| `id` (required) | string | — | Unique tenant identifier. |
| `database-schema` or `schema` | string | inherits `default-database-schema` | Database schema for this tenant. |
| `isolation-mode` or `isolationMode` | string | inherits `default-isolation-mode` | Isolation mode (`schema`, etc.). |
| `tcp.handlers.dir` | string | — | Per-tenant TCP plugin directory. |
| `quotas` | object | inherits `default-tenant-quotas` | Rate limits (see below). |
| `security` | object | inherits global `security` | Per-tenant OIDC overrides (see below). |
| `database-service` | object | inherits global `database-service` | Per-tenant database configuration (see below). |

#### `quotas` object

| Key | Type | Default | Purpose |
| --- | --- | --- | --- |
| `requestsPerMinute` | int | inherits `default-tenant-quotas.requestsPerMinute` | Requests per minute limit. |
| `enabled` | boolean | inherits `default-tenant-quotas.enabled` | Enforce quota for this tenant. |

#### `security` object

Accepts all keys from the global `security` block (see [security-oidc.md](security-oidc.md)):
`provider`, `oidcProviderBaseUrl`, `realm`, `discoveryUri`, `clientId`, `clientSecret`,
`callBackUri`, `scope`, `preferredJwsAlgorithm`, `logoutUrl`, `postLogoutRedirectUri`,
`sessionTtlSeconds`, `sessionMaxEntries`, `sessionCookieSecure`, `allowedOrigins`.

#### `database-service` object

Accepts all keys from the global `database-service` block (see
[database-service.md](database-service.md)): `name`, `dialect`, `driver-class`, `url`,
`username`, `password`, `schema`, `max-pool-size`, `minimum-idle`, `ddl-auto`,
`retired-session-factory-close-delay-seconds`.

### Example with full overrides

```json
{
  "multitenancy": {
    "tenants": [
      {
        "id": "acme",
        "database-schema": "acme",
        "isolation-mode": "schema",
        "quotas": {
          "requestsPerMinute": 1200,
          "enabled": true
        },
        "security": {
          "clientId": "acme-client",
          "clientSecret": "${ACME_OIDC_SECRET}",
          "realm": "acme"
        },
        "database-service": {
          "url": "jdbc:postgresql://acme-db.example.com:5432/payos",
          "username": "acme_user",
          "password": "${ACME_DB_PASSWORD}"
        }
      }
    ]
  }
}
```

## How quotas are enforced (backend selection)

`quotas.requestsPerMinute`/`enabled` are read by `TenantPolicyService.validateQuota` (`payos-kernel`), which counts requests for the current tenant within a one-minute window and rejects with `429 Too Many Requests` once the limit is exceeded. The counting mechanism itself is selected automatically at runtime, in this order — you don't configure which one is used directly, it follows from whether [`sliding-window-service`](sliding-window-service.md) is set up:

1. **`sliding-window-service` configured** (`PayOSConfig.getSlidingWindowCounter()` non-null) — quota requests are counted via `ISlidingWindowCounter.recordAndCount(tenantId, 60000)`: an exact sliding window (no burst can straddle two windows the way a fixed/tumbling window can), and with `storeType: "redis"` a quota that's actually enforced globally across every instance in a multi-instance deployment, not per-instance.
2. **Not configured, but `sliding-window-counter-memory` is on the classpath** (it is, by default, via `payos-runtime`'s dependencies) — the same exact-sliding-window algorithm is still used, resolved once via SPI (`SlidingWindowCounters.resolve("memory", ...)`), just without the cross-instance guarantee: each instance counts its own traffic independently.
3. **Neither available** (e.g. a custom runtime build that strips the memory module) — falls back to the original per-minute bucket counter (`RateWindow`), which resets abruptly at each minute boundary rather than sliding continuously.

In practice, tier 1 or 2 apply to essentially every deployment today — tier 3 exists so quota enforcement never hard-fails if the sliding-window modules are absent. Enabling `sliding-window-service` with `storeType: "redis"` is the only way to get a quota that is actually correct across multiple instances; see [sliding-window-service.md](sliding-window-service.md) for how to configure it, and [developer/tenant-quota-enforcement.md](../developer/tenant-quota-enforcement.md) for what this means for an application developer.

## Relationship to other blocks

- **Database:** `default-database-schema` and `default-isolation-mode` drive how `$DB`
  resolves the schema — see [database-service.md](database-service.md).
- **Security:** the authenticated tenant can be cross-checked against
  `authorized-tenants` on the application — see [security-oidc.md](security-oidc.md).

## Next

- [Architecture: multi-tenancy](../architecture/multi-tenancy.md)
- [database-service.md](database-service.md)
- [sliding-window-service.md](sliding-window-service.md) — the backend that makes quota enforcement exact and, with `redis`, multi-instance-safe.
- [developer/tenant-quota-enforcement.md](../developer/tenant-quota-enforcement.md) — what quota enforcement looks like from an application developer's perspective.
