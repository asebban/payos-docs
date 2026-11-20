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

## Task Scheduling System administration endpoint

Served by `payos-job-scheduler` (`AdminHttpServer`), not by `payos-server-http`: its own listener on `task-scheduler.admin.httpPort` / `TSS_ADMIN_HTTP_PORT`, off by default, in both standalone and embedded modes. Every call needs `Authorization: Bearer <OIDC access token>` with the `tss-platform-admin` or `tss-tenant-admin` role (configurable). Details: [operations §Administration](../operations/task-scheduling-system-v22-2026-11-20.md#administration-manual-trigger-sync-and-list).

| Endpoint | Method | Purpose | Allowed to |
| --- | --- | --- | --- |
| `/tss/admin/jobs[?tenantId=<id>]` | GET | List scheduled jobs with cron, `parallelizable`, next and previous fire times. | Platform admin (any tenant); tenant admin (own tenant only). |
| `/tss/admin/jobs/trigger` | POST | Run one job now. Body `{"tenantId": "<id>", "scriptPath": "<path>"}`; omit `tenantId` for a platform job. Returns 202, 400 without `scriptPath`, 404 for an unknown job. | Platform admin; tenant admin for its own tenant's jobs. |
| `/tss/admin/sync` | POST | Re-read `jobs/` and reconcile the scheduler now; returns the reconciliation report. | Platform admin only. |

401 means no, invalid or expired token; 403 means the token lacks the role or targets another tenant. `X-Correlation-Id` is honoured for log correlation.

## Next

- [configuration/servers.md](../configuration/servers.md)
- [operations/observability.md](../operations/observability.md)
