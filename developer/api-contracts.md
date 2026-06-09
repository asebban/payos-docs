# API Contracts — PayOS HTTP Server

This document defines the complete HTTP API surface of the PayOS HTTP transport layer. It covers all endpoints, URI patterns, request/response headers, status codes, CORS behaviour, security flow, and error contracts.

---

## Table of Contents

1. [URI Routing Conventions](#1-uri-routing-conventions)
2. [Built-in System Endpoints](#2-built-in-system-endpoints)
3. [Application Endpoints](#3-application-endpoints)
4. [Request Headers](#4-request-headers)
5. [Response Headers](#5-response-headers)
6. [HTTP Status Codes](#6-http-status-codes)
7. [CORS Contract](#7-cors-contract)
8. [Authentication & Security Flow](#8-authentication--security-flow)
9. [Error Response Contract](#9-error-response-contract)
10. [Cookie Contract](#10-cookie-contract)
11. [Redirect Handling](#11-redirect-handling)

---

## 1. URI Routing Conventions

### Application Resource Pattern

```
/{appId}/{resourceType}/{resourcePath}
```

| Segment | Description | Example |
|---|---|---|
| `appId` | Identifier of the registered application | `payments-app` |
| `resourceType` | One of `api`, `page`, `component`, `menu` | `api` |
| `resourcePath` | Resource-specific path (for `api`, `page`, `component`) | `/users/123` |

**Routing matrix:**

| URI pattern | Handled by | Allowed methods |
|---|---|---|
| `/{appId}/api/**` | `ApiResourceHandler` | GET, POST, PUT, DELETE |
| `/{appId}/page/**` | `VueResourceHandler` | GET |
| `/{appId}/component/**` | `ComponentHandler` | GET |
| `/{appId}/menu` | `MenuHandler` | GET |

**Note — `page` suffix shortcut.** When the URI ends with exactly `/page` (no trailing slash), the server appends a trailing slash automatically before processing.

**AppId extraction.** The first path segment after the root slash is always treated as `appId`. The second segment determines the resource type.

---

## 2. Built-in System Endpoints

These endpoints are handled directly by the HTTP server and do not involve application routing.

---

### `/health`

Liveness probe for load balancers and container orchestrators.

**HTTP method:** Any (routing is URI-based; `OPTIONS` is intercepted first as CORS preflight).

**Authentication:** Not required.

**Response:**

| Status | Content-Type | Body |
|---|---|---|
| `200 OK` | `text/plain; charset=UTF-8` | `OK` |

---

### `/me`

Returns the currently authenticated principal.

**HTTP method:** Any.

**Authentication:** Required. Delegates to the configured `ISecurityService`.

**Response:**

| Status | Content-Type | Body |
|---|---|---|
| `200 OK` | `application/json` | Principal object (structure depends on security provider) |
| `401 Unauthorized` | `text/plain` | Authentication error message |

---

### `/stop`

Initiates a graceful server shutdown.

**HTTP method:** Any.

**Authentication:** Not required — protected by origin check only.  
**Access restriction:** Only requests where `InetAddress.isLoopbackAddress() || isAnyLocalAddress()` is true are accepted.

**Response:**

| Status | Content-Type | Body | Condition |
|---|---|---|---|
| `200 OK` | `text/plain; charset=UTF-8` | `stopping` | Request from localhost |
| `403 Forbidden` | `text/plain; charset=UTF-8` | `Forbidden` | Request from non-loopback address |

---

### `/callback`

OAuth 2.0 / OIDC authorization code callback. Completes the authentication flow after the identity provider redirects back.

**HTTP method:** Any (the request is always forwarded to the security service as a GET internally).

**Authentication:** Not required (this endpoint performs authentication).  
**Delegates to:** `ISecurityService.handleCallback(request)`.

**Response:** Determined entirely by the security service implementation (typically a session cookie and a redirect to the original URI).

---

### `/logout`

Invalidates the current session.

**HTTP method:** Any (the actual HTTP method is forwarded to the security service).

**Authentication:** Typically required (depends on security service implementation).  
**Delegates to:** `ISecurityService.handleLogout(request)`.

**Response:** Determined by the security service implementation.

---

### `OPTIONS *`

CORS preflight handler. Matches all URIs for `OPTIONS` requests.

**Authentication:** Not required.

**Response:**

| Status | Body |
|---|---|
| `204 No Content` | _(empty)_ |

Full CORS headers are added (see [Section 7](#7-cors-contract)).

---

## 3. Application Endpoints

### `GET /{appId}/api/{path}`

Executes a GET API script registered under `{path}` in the application `{appId}`.

**Authentication:** Required for non-public resources (enforced by security service).

**Request:**

| Element | Details |
|---|---|
| Query parameters | Forwarded to the script as `$Request.parameters` |
| Request body | Not applicable for GET |
| `X-Tenant-Id` | Used for tenant scope resolution |
| `X-Correlation-Id` | Generated if absent |

**Response:**

| Status | Body | Notes |
|---|---|---|
| `200 OK` | Script-defined JSON or text | Default success |
| `4xx` / `5xx` | Error message (text/plain) | Script or platform error |

---

### `POST / PUT / DELETE /{appId}/api/{path}`

Executes a POST, PUT, or DELETE API script.

**Authentication:** Required for non-public resources.

**Request body:** Raw string, read in full before script execution. Accessible in the script as `$Request.body` or `$Request.getJsonBody()`.

**Headers:**

| Header | Notes |
|---|---|
| `Content-Type` | Should be `application/json; charset=UTF-8` for JSON payloads |
| `X-Tenant-Id` | Used for tenant scope resolution |
| `X-Correlation-Id` | Generated if absent |

**Response:** Same contract as GET.

---

### `GET /{appId}/page/{path}`

Serves a Vue/HTML page resource registered under `{path}`.

**Authentication:** Unauthenticated requests receive a minimal pre-auth tenant scope (login pages remain accessible).

**Response:**

| Status | Content-Type | Body |
|---|---|---|
| `200 OK` | `text/html` | Rendered page content |
| `404 Not Found` | `text/plain` | Page not found |

---

### `GET /{appId}/component/{path}`

Serves a UI component resource.

**Response:** Same structure as page resources.

---

### `GET /{appId}/menu`

Returns the composed, tenant-filtered menu entries for the application.

**Authentication:** Required (authenticated tenant context is used to filter active capabilities).

**Request headers:**

| Header | Required | Notes |
|---|---|---|
| `X-Tenant-Id` | Recommended | Determines which capability menu entries are included |
| `X-Correlation-Id` | Optional | Auto-generated if absent |

**Composition logic:**

1. Load `menu/entries.json` from the application's own directory. Entry IDs are kept as-is.
2. For each ID in the application's `extends` list:
   - If the extended entity is a capability, check the activation store: `isActive(capabilityId, appId, tenantId)`.
   - If active, load its `menu/entries.json` and prefix all `id` and `parentId` values with `{capabilityId}/`.
   - If the extended entity is not a capability, include its entries without prefixing.
3. Merge into a single flat list (own entries first, then extends in declaration order).

**Response:**

| Status | Content-Type | Body |
|---|---|---|
| `200 OK` | `application/json; charset=UTF-8` | Flat JSON array of menu entry objects |

**Response body example:**

```json
[
  { "id": "dashboard", "name": "Dashboard", "route": "/dashboard" },
  { "id": "reporting/reports", "name": "Reports" },
  { "id": "reporting/monthly", "name": "Monthly", "route": "/reports/monthly", "parentId": "reporting/reports" }
]
```

**Menu entry fields:**

| Field | Type | Required | Processed by | Description |
|---|---|---|---|---|
| `id` | string | yes | Backend + Frontend | Stable unique identifier. Capability entries carry a `{capabilityId}/` prefix added by the backend. Used by the frontend to build the parent/child tree. |
| `parentId` | string \| null | no | Backend + Frontend | ID of the parent entry. Absent/null means top-level. The backend prefixes this value for capability entries. The frontend uses it to build the menu tree. |
| `label` | string | yes | Frontend | Display text rendered for the entry. |
| `roles` | string[] | no | Frontend | User must hold at least one role for the entry to be visible. Empty/absent means visible to all authenticated users. Frontend display filter only — server-side authorization is always enforced independently. |
| `component` | string | no | Frontend | Name of the page component to activate on click (via `setPage()`). Takes priority over `page` and `href`. |
| `page` | string | no | Frontend | Page identifier to load on click (via `loadPage()`). Used when `component` is absent. |
| `href` | string | no | Frontend | Absolute or relative URL opened on click (via `window.open()`). Used when both `component` and `page` are absent. |
| `target` | string | no | Frontend | `_self` (default) or `_blank`. Applies when `href` is set. |

---

## 4. Request Headers

| Header | Required | Description |
|---|---|---|
| `X-Tenant-Id` | Recommended | Tenant identifier used for scope isolation, CORS resolution, and menu filtering. If absent, falls back to `multitenancy.tenantSimulator.tenantId` (development only). |
| `X-Correlation-Id` | Optional | Request trace identifier propagated unchanged through all downstream calls and logs. Auto-generated (UUID v4) if absent. |
| `Content-Type` | Required for POST/PUT | Must be `application/json; charset=UTF-8` for JSON payloads. |
| `Authorization` | Conditional | Bearer token or Basic credentials, depending on security provider configuration. |
| `Origin` | Conditional | Required by browsers for cross-origin requests. Used for CORS origin validation. |
| `X-Requested-With` | Optional | Set to `XMLHttpRequest` by jQuery/Axios. Used to detect XHR clients and return a JSON redirect hint instead of a `302` response. |
| `Cookie` | Conditional | Session cookie forwarded automatically by the browser. Used by the OIDC/session security provider. |

---

## 5. Response Headers

### Always Present (all responses)

| Header | Value | Description |
|---|---|---|
| `Content-Type` | `application/json` or `text/plain` | Determined by the handler. |
| `Content-Security-Policy` | `default-src 'none'; frame-ancestors 'none'; base-uri 'none'` | XSS and clickjacking mitigation. |
| `X-Frame-Options` | `DENY` | Prevents embedding in iframes. |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME-type sniffing. |
| `Referrer-Policy` | `no-referrer` | Suppresses the `Referer` header on navigation. |
| `Cache-Control` | `no-store, no-cache, must-revalidate, max-age=0` | Disables all caching. |
| `Pragma` | `no-cache` | HTTP/1.0 cache suppression. |

### Successful Application Responses Only

These headers are added by `enrichResponse()`, which is called only on the success path for application requests (`/{appId}/**`).

| Header | Value | Description |
|---|---|---|
| `X-Correlation-Id` | UUID | Echo of incoming value, or auto-generated. |
| `X-Tenant-Id` | Resolved tenant ID | Echo of resolved tenant scope. |


### HTTPS Only

| Header | Value |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |

### CORS Responses

| Header | Value | Condition |
|---|---|---|
| `Access-Control-Allow-Origin` | Validated origin or `null` | Present on all cross-origin responses. |
| `Access-Control-Allow-Methods` | `GET,POST,PUT,DELETE,OPTIONS` | Present on all cross-origin responses. |
| `Access-Control-Allow-Headers` | `Content-Type, Authorization, X-Requested-With, X-Tenant-Id, X-Correlation-Id` | Present on all cross-origin responses. |
| `Access-Control-Allow-Credentials` | `true` \| `false` | `true` only when origin is validated. |
| `Vary` | `Origin` | Added when origin validation is performed. |

### Conditional

| Header | Condition |
|---|---|
| `Location` | Redirect (3xx) responses. |
| `Set-Cookie` | Authentication responses. All values are automatically hardened (see [Section 10](#10-cookie-contract)). |

---

## 6. HTTP Status Codes

| Code | Constant | Trigger |
|---|---|---|
| `200 OK` | `STATUS_OK` | Successful resource response. Default when status not set by handler. |
| `204 No Content` | — | CORS preflight (`OPTIONS`). |
| `302 Found` | `STATUS_REDIRECT` | Server-side redirect (auth flow). XHR clients receive `200` with a JSON hint instead (see [Section 11](#11-redirect-handling)). |
| `400 Bad Request` | `STATUS_BAD_REQUEST` | Missing or malformed `appId`, unrecognised resource type. |
| `401 Unauthorized` | `STATUS_UNAUTHORIZED` | Authentication required but not present or invalid. Returned by the security service. |
| `403 Forbidden` | `STATUS_FORBIDDEN` | CORS origin not allowed; `/stop` called from non-loopback; tenant policy violation. |
| `404 Not Found` | `STATUS_NOT_FOUND` | URI pattern not recognised; application not registered; resource not found within application. |
| `405 Method Not Allowed` | — | HTTP method other than GET/POST/PUT/DELETE used on an application endpoint. |
| `429 Too Many Requests` | `STATUS_TOO_MANY_REQUESTS` | Rate limit enforced by tenant policy or security service. |
| `500 Internal Server Error` | `STATUS_INTERNAL_ERROR` | Unhandled exception during request processing. |

---

## 7. CORS Contract

### Origin Resolution (First Match Wins)

```
1. Application-level  →  app.config.security.cors.allowedOrigins
2. Tenant-level       →  tenant.config.security.cors.allowedOrigins
3. Global-level       →  bootstrap.json → security.cors.allowedOrigins
```

Each level is an array of origin strings. The wildcard `"*"` matches any origin.

### Validation Outcome

| Outcome | `Access-Control-Allow-Origin` | `Access-Control-Allow-Credentials` |
|---|---|---|
| Origin matched | `<origin>` (echoed) | `true` |
| Origin not matched | `null` | _(absent)_ |

### Preflight (`OPTIONS`)

All `OPTIONS` requests return `204 No Content` with the full CORS header set regardless of origin validation, so the browser can read the outcome.

---

## 8. Authentication & Security Flow

### Request Processing Decision

```
Incoming request to /{appId}/**
        │
        ├─ isAuthenticated(request) == true
        │       └─ TenantPolicyService.enforceAndOpenScope(request, appId)
        │               └─ processRequest(appId, request)
        │
        └─ isAuthenticated(request) == false
                └─ openPreAuthTenantScope(request, appId)
                        └─ processRequest(appId, request)
```

Unauthenticated requests still reach the application so that login pages and public resources remain accessible. The application or script layer is responsible for returning `401` / `403` as needed.

### Pre-Auth Tenant Scope

When the request is unauthenticated:

1. `X-Tenant-Id` is read from the request header.
2. If absent, `multitenancy.tenantSimulator.tenantId` is used (development/testing only).
3. `X-Correlation-Id` is generated as UUID v4 if not present.
4. Both values are stored in the request context and in MDC for log correlation.

### Authenticated Tenant Scope

The `TenantPolicyService` additionally:

- Validates that the tenant is permitted to access the application.
- Populates MDC with `tenantId`, `correlationId`, and `appId`.

### Authenticated Tenant ID Resolution

For authenticated requests, if `X-Tenant-Id` is absent from the incoming headers, the server calls `securityService.resolveAuthenticatedTenantId(request)` to extract the tenant from the authenticated session (e.g. from the OIDC token claims). The resolved value is injected into the request before `TenantPolicyService.enforceAndOpenScope` is called.

### Security Service Interface (`ISecurityService`)

| Method | Called by | Purpose |
|---|---|---|
| `isAuthenticated(request)` | Per-request | Gate for tenant policy enforcement |
| `resolveAuthenticatedTenantId(request)` | Authenticated requests without `X-Tenant-Id` | Extracts tenant identifier from the authenticated session |
| `getCurrentUser(request)` | `/me` | Returns principal as response |
| `handleCallback(request)` | `/callback` | Completes OIDC auth code exchange |
| `handleLogout(request)` | `/logout` | Invalidates session |

---

## 9. Error Response Contract

### Format

All error responses use:

```
Status:        <HTTP status code>
Content-Type:  text/plain; charset=UTF-8
Headers:       CORS headers + security headers
Body:          Plain text message
```

**Note:** `X-Correlation-Id` and `X-Tenant-Id` are **not** added to error responses produced by `sendErrorResponse()` (e.g. 400 missing appId, 404 not found, 405 method not allowed, 500 unhandled exception). They are only present on successful application responses enriched by `enrichResponse()`. Errors returned by the security service or application logic that go through `sendSuccessResponse()` (e.g. `401`, `403`) will carry these headers.

### Examples

**400 Bad Request — missing appId**

```
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=UTF-8
X-Content-Type-Options: nosniff
Cache-Control: no-store, no-cache, must-revalidate, max-age=0

Bad Request: Missing appId in URI
```

**403 Forbidden — CORS origin rejected**

```
HTTP/1.1 403 Forbidden
Content-Type: text/plain; charset=UTF-8
Access-Control-Allow-Origin: null
X-Content-Type-Options: nosniff

Forbidden
```

**404 Not Found — URI pattern not matched**

```
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=UTF-8
X-Content-Type-Options: nosniff

Not Found
```

**500 Internal Server Error**

```
HTTP/1.1 500 Internal Server Error
Content-Type: text/plain; charset=UTF-8
X-Content-Type-Options: nosniff

<exception message>
```

---

## 10. Cookie Contract

All `Set-Cookie` header values produced by any handler (including the security service) are automatically hardened by the server before being sent to the client:

| Attribute | Applied when | Value |
|---|---|---|
| `HttpOnly` | Not already present | Prevents JavaScript access |
| `Secure` | Not already present | HTTPS transmission only |
| `SameSite` | Not already present | `Lax` — CSRF mitigation |

This hardening is unconditional and cannot be bypassed by application scripts.

---

## 11. Redirect Handling

When a handler returns a response with a `Location` header (3xx redirect):

| Client type | Detection | Server behaviour |
|---|---|---|
| **Browser** | No `X-Requested-With: XMLHttpRequest` header | Forwards the redirect: status code from response (e.g. `302`), `Location` header set. |
| **XHR / Fetch** | `X-Requested-With: XMLHttpRequest` present | Returns `200 OK` with a JSON body containing the redirect target, so the SPA can handle navigation itself. |

**XHR redirect hint body:**

```json
{ "redirect": "https://idp.example.com/auth?..." }
```

**Exception:** Redirect interception is skipped when the `Response` object carries an `X-Api-Call` header (set internally by the API resource handler). In that case, the raw status code and `Location` header are forwarded to the client unchanged.
