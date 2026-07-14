# Security architecture

PayOS embeds security into the request path structurally — the *secure & compliant by design* principle. This document covers authentication (OIDC), sessions, the principal, authorization, idempotency, transport hardening, and the audit/traceability that PCI DSS requires.

## Authentication: the `ISecurityService` contract

`ma.s2m.payos.security.ISecurityService` defines the security operations used by the pipeline and the built-in HTTP endpoints:

```java
Response             check(Request, List<String> requiredRoles);   // non-null = forbidden
Response             authenticate(Request);                        // non-null = stop (e.g. redirect)
Response             handleCallback(Request);                      // OIDC callback
Response             handleLogout(Request);                        // logout
Response             getCurrentUser(Request);                      // user info JSON  (/me)
Map<String,Object>   getCurrentPrincipal(Request);                 // claims, may be null
boolean              isAuthenticated(Request);
String               resolveAuthenticatedTenantId(Request);        // tenant from claims
```

### Provider selection

`SecurityServiceFactory.create(application[, request])` chooses the implementation by resolving the `provider` value in this order:

1. application security config (`security.provider`),
2. the session store (`OidcSessionKeys.SESSION_PROVIDER`),
3. global security config (`PayOSConfig.settings.security.provider`),
4. default `"nimbus"`.

| `provider` | Implementation |
| --- | --- |
| `"nimbus"` (default) | `NimbusSecurityService` — Nimbus OIDC (primary). |
| `"pac4j"` (legacy) | `SecurityService` — pac4j OIDC. |

## OIDC integration (pac4j)

Packages `ma.s2m.payos.security.oidc.pac4j` and `…oidc.nimbus` implement OpenID Connect via
**pac4j 6.0.0**. The relevant configuration keys (under an application's or the global
`security` block — `IConfigSpec…Security`):

| Key | Purpose |
| --- | --- |
| `provider` | `pac4j` or `nimbus`. |
| `oidcProviderBaseUrl` / legacy `keycloakBaseUrl` | IdP base URL. |
| `discoveryUri` | OIDC discovery document URL. |
| `clientId`, `clientSecret` | OAuth client credentials. |
| `callBackUri` | Redirect URI registered with the IdP. |
| `scope` | Requested scopes. |
| `realm` | IdP realm. |
| `preferredJwsAlgorithm` | JWT signing algorithm. |
| `logoutUrl`, `postLogoutRedirectUri` | Logout flow. |
| `sessionTtlSeconds` (default ~30 min) | Session lifetime. |
| `sessionMaxEntries` (default 10,000) | Session store capacity. |
| `sessionCookieSecure` | Mark the session cookie `Secure`. |
| `allowedOrigins` | CORS allowlist (also a CORS source — see below). |

Full key documentation: [configuration/security-oidc.md](../configuration/security-oidc.md).

## Sessions

Sessions are kept in `PayOSSessionStore` (singleton), keyed by the `PAYOS_SESSION_ID` cookie. pac4j integrates through `DefaultPayOSWebContext`. Session TTL and capacity are bounded by `sessionTtlSeconds` and `sessionMaxEntries`. The session cookie is hardened (`HttpOnly`, `Secure`, `SameSite=Lax`) by the HTTP transport.

## The principal

When authenticated, `getCurrentPrincipal(request)` returns a `Map<String,Object>` of OIDC claims, exposed to scripts as `$Principal`:

| Key | Meaning |
| --- | --- |
| `id` | User identifier (from the `sub` claim). |
| `email`, `name`, `preferred_username`, `roles`, … | Standard / IdP claims. |

`$Principal` is `null` for unauthenticated requests. Scripts use it for authorization decisions beyond declarative roles — see [developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Authorization

API resources may declare required roles (`ApiResource.getRoles()`). When roles are present, the pipeline calls `authenticate` then `check(request, roles)`; a non-null `Response` from either stops processing (redirect to login, or 403). Resources without roles are public within their application.

## Idempotency

`IdempotencyService` provides safe retries for mutating requests: the pipeline calls
`checkIdempotency(request)` before execution. When idempotency is enabled, a missing or blank
idempotency key is rejected before script execution — unless `failOnAbsenceOfIdempotencyKey` is
set to `false`, in which case the request simply proceeds without an idempotency check. A repeat
with a valid cached record returns the cached `Response`. The pipeline calls
`storeResponse(request, response)` after successful execution. This protects financial
operations from duplicate side effects on client retries. Configuration is in
[configuration/idempotency.md](../configuration/idempotency.md).

## Transport hardening (HTTP)

The HTTP transport adds these headers to **every** response:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: no-referrer
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
Pragma: no-cache
Content-Security-Policy: default-src 'none'; frame-ancestors 'none'; base-uri 'none'
Strict-Transport-Security: max-age=31536000; includeSubDomains      (HTTPS only)
```

CORS headers are added only when the request origin matches an allowlist. CORS origins are resolved from three levels (most specific wins): application → tenant → global
`security.cors.allowedOrigins` / `security.allowedOrigins`. See [configuration/servers.md](../configuration/servers.md) and [configuration/security-oidc.md](../configuration/security-oidc.md).

## Secrets and cryptography

Sensitive material is never hard-coded. Secrets are retrieved through the secret-provider SPI (`$Secrets`), configuration files may be encrypted at rest (`CryptoService`), and the secret provider encrypts values with AES-256-GCM. You can find here a detailed description of the [Secret Provider Architecture](./secret-provider-architecture.md)

See [extensibility.md](extensibility.md) and [operations/secrets-management.md](../operations/secrets-management.md).

## Audit & traceability (PCI DSS)

`AuditLogger.logApiExecution(userId, tenantId, correlationId, apiPath, statusCode)` is
called in the pipeline's `finally` block (aligned with PCI DSS Req. 10.2.1). Combined with
mandatory `X-Correlation-Id` / `X-Tenant-Id` propagation and the secret-provider audit
logger (`payos.secret.audit`), every security-relevant action is attributable to a user,
tenant, and trace. Operational details: [operations/observability.md](../operations/observability.md).

## References

You can find below detailed descriptions of the different security components

- [Legacy PAC4J](./security/legacy-pac4j.md)
- [Nimbus Security Service (default)](./security/nimbus-security-service.md)
- [security features inventory](./security/security-inventory.md)

## Next

- [Multi-tenancy](multi-tenancy.md) — how the authenticated tenant becomes the request scope.
- [Eventing & webhooks](eventing-webhooks.md) — `security.login` / `security.logout` events.
