# HTTP endpoints index

The endpoints served by the HTTP/HTTPS transport (`payos-server-http`). Application routes are
derived from the URI; the rest are built-in. Configuration is in
[configuration/servers.md](../configuration/servers.md).

## Application routes

| Pattern | Resource | Handler |
| --- | --- | --- |
| `/{appId}/api/**` | API scripts | `ApiResourceHandler` |
| `/{appId}/page/**` | Pages | `VueResourceHandler` |
| `/{appId}/component/**` | Components | `ComponentHandler` |
| `/{appId}/menu/**` | Menus | `MenuHandler` |

`{appId}` is the first path segment (`Application.getAppIdFromUri`).

## Built-in endpoints

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/health` | GET | Liveness/readiness probe. |
| `/me` | GET | Current authenticated principal. |
| `/callback` | GET | OIDC redirect callback. |
| `/logout` | GET | Log out the current session. |
| `/stop` | POST | Graceful shutdown — **localhost only**. |
| `/openapi.yaml` | GET | The OpenAPI document (from `swaggerUI.openapi-yaml`). |
| `/swagger/**` | GET | Swagger UI — **local-only by default**. |
| (any) | OPTIONS | CORS preflight handling. |

## Response headers

The transport adds hardened headers to responses:

| Header | Value |
| --- | --- |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `Referrer-Policy` | `no-referrer` |
| `Cache-Control` | `no-store, ...` |
| `Pragma` | `no-cache` |
| `Content-Security-Policy` | `default-src 'none'; frame-ancestors 'none'; base-uri 'none'` |
| `Strict-Transport-Security` | (HTTPS only) |
| `X-Correlation-Id` | The request's correlation id (generated if absent). |
| `X-Tenant-Id` | The resolved tenant id. |

CORS (`Access-Control-*`) is resolved app → tenant → global from `allowedOrigins`. See
[configuration/security-oidc.md](../configuration/security-oidc.md).

## Request headers of note

| Header | Constant | Purpose |
| --- | --- | --- |
| `X-Tenant-Id` | `Request.HEADER_TENANT_ID` | Tenant selection (case-insensitive). |
| `X-Correlation-Id` | `Request.HEADER_CORRELATION_ID` | Trace correlation. |

## Cookies

The session cookie is `PAYOS_SESSION_ID` (HttpOnly, SameSite=Lax, Secure on HTTPS / when
configured).

## Next

- [configuration/servers.md](../configuration/servers.md)
- [operations/observability.md](../operations/observability.md)
