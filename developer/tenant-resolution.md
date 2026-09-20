# Tenant resolution

How PayOS decides which tenant a request belongs to — the exact priority order, what that means for `X-Tenant-Id`, and how to test as a specific tenant during development. Configuring tenants themselves (schemas, isolation modes, quotas) is covered in [configuration/multi-tenancy.md](../configuration/multi-tenancy.md); the architectural picture (scope lifecycle, isolation at the data layer) is in [architecture/multi-tenancy.md](../architecture/multi-tenancy.md). This page is the practical, "what actually wins" reference for application developers.

## The one rule that matters most

**If the caller is authenticated, their own identity decides the tenant — nothing else is even consulted.** A logged-in user or a valid API key always resolves to the tenant embedded in their own credentials (their Keycloak realm, or an explicit `tenantId` claim), never to whatever `X-Tenant-Id` header happened to be sent alongside the request. This is deliberate: a client-supplied header must never be able to override what the caller's own token says, or authentication would not actually mean anything for tenant isolation.

`X-Tenant-Id` (and the other fallbacks below) only come into play for **unauthenticated** requests.

## The exact priority order

For every request, on every transport (HTTP, TCP, Queue) — this is enforced once, centrally, before your application's script runs, so it behaves identically regardless of how the request arrived:

1. **A principal was resolved for this request** (a valid session, or a valid `Authorization: Bearer <token>`) →
   the tenant is the principal's own tenant: the `tenantId` claim on the token if present, otherwise the realm extracted from a Keycloak-style `issuer` (`.../realms/{realm}/...`). **Nothing else is considered.**
   - If the principal has no derivable tenant at all (no `tenantId` claim, no extractable realm), the request is **rejected** (`403`) rather than silently falling back to a header — an authenticated caller whose credentials don't carry a tenant is a configuration problem to fix, not something to paper over.
2. **No principal was resolved** (the request is anonymous) →
   1. The [tenant simulator](../configuration/multi-tenancy.md) (`multitenancy.tenantSimulator`), if enabled, wins outright — this is a **development/test-only** escape hatch; it must never be enabled in production.
   2. Otherwise: the `X-Tenant-Id` header, then a tenant already present in the request's internal context (relevant mainly to non-HTTP transports carrying it forward), then — as a last resort — whatever tenant is already active on the current thread (relevant when your code runs as part of an already-tenant-scoped operation).

If no tenant resolves at all and the application/request type requires one (`multitenancy.requireTenantId`, on by default), the request is rejected with `400` before it ever reaches your script.

## A new, stricter rule for API keys

Any request that carries an `Authorization: Bearer <token>` header **must** resolve to a valid principal, or it is rejected with `401` immediately — even if the resource it's targeting doesn't require any roles. Previously, an invalid or expired bearer token on a public/roleless resource was silently ignored, as if the header hadn't been sent at all; it no longer is. If your client sends a bearer token, that token must be valid.

## What this looks like in practice

**A logged-in user hitting your API:**

```
GET /myapp/api/orders
Cookie: <session cookie for a user in Keycloak realm "acme">
X-Tenant-Id: other-tenant        <-- ignored entirely
```

`$Tenant` inside your script is `"acme"`, not `"other-tenant"` — the header is not read at all once a principal exists.

**An API-key client:**

```
GET /myapp/api/orders
Authorization: Bearer eyJhbGciOi...   <-- token whose issuer is .../realms/acme/...
```

`$Tenant` is `"acme"`. If the token is invalid or expired, the request never reaches your script — the caller gets `401` straight from the platform.

**An anonymous request, testing as a specific tenant without logging in:**

```
GET /myapp/api/public-catalog
X-Tenant-Id: acme
```

`$Tenant` is `"acme"` — the header is honored precisely because there is no principal to override it with.

## Accessing the resolved tenant in your script

The result of all of the above is what you get as [`$Tenant`](scripting-bindings.md#tenant):

```javascript
function execute(request, controlData) {
    $Logger.info("Processing request for tenant: {}", $Tenant);
    var rows = $DB.find("SELECT * FROM accounts WHERE tenant = :t", { t: $Tenant });
    // ...
}
```

You never need to (and should not try to) re-derive the tenant yourself from headers or the principal — `$Tenant` already reflects the fully-resolved value, and `$DB` is automatically scoped to it (see [data access](data-access.md)).

## Testing locally as different tenants

Two supported ways, depending on whether you're testing an authenticated or an anonymous flow:

- **Anonymous/public resources**: set `X-Tenant-Id` directly on the request. This only works because there's no principal — it's the normal, supported way to exercise multi-tenant behavior for public endpoints without standing up a full OIDC login for every tenant.
- **Authenticated resources**: either log in as a real user in the target tenant's realm, or — in development only — enable `multitenancy.tenantSimulator` with a fixed `tenantId`, which substitutes for the tenant on any anonymous request as if it had been supplied (still subject to rule 1 above: it has no effect once a principal exists). See [configuration/multi-tenancy.md](../configuration/multi-tenancy.md#the-tenant-simulator-development-only) for the exact keys.

## Why this is enforced centrally, not per-application

Tenant resolution, quota checks (see [tenant quota enforcement](tenant-quota-enforcement.md)), and now bearer-token validation all happen in the platform's request pipeline before any application script runs — identically for every transport. You don't opt into this and can't opt out of it; there's nothing to configure per-application to get correct tenant isolation. If you're curious about exactly where in the kernel this happens, see [architecture/multi-tenancy.md](../architecture/multi-tenancy.md).
