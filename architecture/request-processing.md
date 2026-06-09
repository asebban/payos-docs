# Request processing

The complete path of a request, from the moment a transport server receives bytes to the moment a `Response` is serialized back. This is the heart of the runtime and spans the **server**, **resource**, and **scripting** layers.

## The transport abstraction

All transports implement `ma.s2m.payos.servers.IServer`:

```java
void start(String host, int port) throws UnableToStartServerException;
void stop();
Response processRequest(String appId, Request request) throws Exception;
```

Constants on `IServer` define the context keys shared across transports:

| Constant | Value | Meaning |
| --- | --- | --- |
| `CONTEXT_TENANT_ID` | `"tenantId"` | Resolved tenant for the request. |
| `CONTEXT_CORRELATION_ID` | `"correlationId"` | Trace identifier. |
| `CONTEXT_APP_ID` | `"appId"` | Target application. |

The abstract base `ma.s2m.payos.servers.Server` implements `processRequest`:

```
Server.processRequest(appId, request)
  ├─ Application application = Application.getApplicationById(appId)
  ├─ application.setServer(this)
  ├─ ResourceHandler handler = ResourceHandler.getHandler(request.getType())
  └─ return handler.handle(application, request)
```

Protocol selection happens in `ma.s2m.payos.servers.Servers`, which discovers `ServerProvider` implementations through `ServiceLoader` and dispatches by protocol name:

```java
IServer start(String host, int port, String protocol, Map<String,Object> serverConfig)
```

`UnableToStartServerException` (its only constructor takes a `String`) is thrown for an unsupported protocol, invalid configuration, or a port binding failure.

> Adding a protocol means implementing `ServerProvider` + `Server` and registering it in `META-INF/services/ma.s2m.payos.servers.ServerProvider` — see [extensibility.md](extensibility.md).

## Stage 1 — transport ingress

Each transport converts its wire format into a `Request` and resolves the target `appId`, the tenant, and the correlation ID. Tenant resolution and scope opening are described in [multi-tenancy.md](multi-tenancy.md); the per-transport details are summarized here.

### HTTP / HTTPS (`payos-server-http`)

Built on **Undertow**. The handler chain first answers built-in endpoints (see [reference/http-endpoints.md](../reference/http-endpoints.md)) — `OPTIONS` preflight, `/health`, `/me`, `/callback`, `/logout`, `/stop`, `/openapi.yaml`, `/swagger/**` — then routes `/{appId}/[api|page|component|menu]/**` through:

```
Undertow I/O thread
  └─ handleAsyncProcessing → exchange.dispatch(worker)
       └─ processInVirtualThread (Java 21 virtual thread)
            ├─ build Request (method, type, headers, query, path, body)
            ├─ resolve Application, SecurityService
            ├─ authenticated? TenantPolicyService.enforceAndOpenScope(...)
            │  else            openPreAuthTenantScope(...)
            ├─ super.processRequest(appId, request)   ← into the kernel
            ├─ enrichResponse(...)   ← add X-Tenant-Id, X-Correlation-Id
            └─ exchange.getIoThread().execute(sendSuccessResponse)
```

Blocking business logic runs on a **virtual thread**; the response is sent back on the original Undertow I/O thread. Every response receives hardened security headers (CSP, `X-Frame-Options: DENY`, `nosniff`, no-store cache, HSTS on HTTPS) and, when the origin is allowed, CORS headers. `Set-Cookie` headers are hardened with `HttpOnly`, `Secure`, and `SameSite=Lax`. See [security-architecture.md](security-architecture.md).

### TCP (`payos-server-tcp`)

A `ServerSocket` accept loop spawns one **virtual thread per connection**:

```
handleClient(socket)
  ├─ Request request = decoder.decode(in)
  ├─ String appId = Application.getAppIdFromUri(request.getPath())
  ├─ TenantPolicyService.enforceAndOpenScope(request, appId)
  ├─ Response response = handler.handle(appId, request)   ← plugin or super.processRequest
  ├─ add X-Tenant-Id / X-Correlation-Id headers
  └─ encoder.encode(response, out); out.flush()
```

The decoder, encoder, and handler are **pluggable** (`TcpMessageDecoder`, `TcpMessageEncoder`, `TcpMessageHandler`) and discovered by scanning JARs in `tcp-handlers-dir`. Together they separate **wire-format parsing** from **PayOS request processing**:

| Plugin | Contract | Responsibility |
| --- | --- | --- |
| `TcpMessageDecoder` | `decode(InputStream) -> Request` | Reads raw bytes from the socket and translates the external protocol into the neutral PayOS `Request` model. This is where ISO 8583, fixed-width records, length-prefixed frames, or any custom TCP framing are parsed. |
| `TcpMessageHandler` | `handle(appId, Request) -> Response` | Optional protocol-specific handling step. It can delegate to `super.processRequest(appId, request)` for normal PayOS API execution, or implement a specialized shortcut when the TCP protocol needs one. |
| `TcpMessageEncoder` | `encode(Response, OutputStream)` | Serializes the neutral PayOS `Response` back into the external TCP protocol expected by the client. |

If no plugin JAR provides one of these interfaces, the TCP server uses sensible UTF-8 defaults: it reads the input as a UTF-8 request payload, routes it through the normal kernel pipeline, and writes the response as UTF-8 bytes. This makes local/simple TCP usage work out of the box while allowing production protocols to plug in strict codecs. See [developer/java-extensions.md](../developer/java-extensions.md) and [configuration/servers.md](../configuration/servers.md).

For a fully detailed documentation on TCP Server, see the following [TCP Server documentation](./tcp%20server/README.md)

### Queue (`payos-server-queue`)

Subscribes to a topic via an `IQueueClient` and processes each message asynchronously:

```
subscribe((message, replyTopic, replyRequired) -> {
  Request request = parseRequest(message)      ← JSON envelope
  String appId    = extractAppId(request)
  TenantPolicyService.enforceAndOpenScope(request, appId)
  Response response = processRequest(appId, request)
  add X-Tenant-Id / X-Correlation-Id
  if (replyRequired) queueClient.publish(buildResponseMessage(request, response), replyTopic)
})
```

The request and response are JSON envelopes; the correlation ID falls back to the payload field, then the header, then a freshly generated UUID. Because `subscribe` is asynchronous, the server thread stays alive until `stop()`.

For a more detailed explanation about Queue servers, see [Queue server documentation](./queue%20server/queue-server.md)

## Stage 2 — resource routing

### Resource Processing

- `Server.processRequest` finds the target `Application` by appId.
- Dispatch to `ResourceHandler` by request type (`api`, `page`, `component`, `menu`, `lib`, `i18n`).
- Specialized loaders/handlers resolve and execute target behavior.
- **`api` resource type**: `ApiResourceHandler` executes JS scripts through a full **hooks + webhooks pipeline**: `pre-request` hook → API script → `post-request` hook (error path: `on-error` hook). `$WebHooks.dispatchNative()` is called automatically at each lifecycle point.
- **`page` resource type**: `VueResourceHandler` assembles Vue SFC pages through an equivalent pipeline: `page-pre-serve` hook → Vue assembly → `page-post-serve` hook (error path: `page-on-error` hook), with native webhook dispatch at each point.
- **`menu` resource type**: `MenuHandler` aggregates `entries.json` files from the application's `menu/` directory and all active capability extensions. Capability menu entries are only included when the capability is activated for the current app/tenant via `ActivationStore`.
- **`lib` resource type**: `LibraryHandler` loads and evaluates shared JavaScript files from the application's `lib/` directory. Libraries are injected into API scripts via the `$Library` binding and cached per file path with `lastModified`-based invalidation.
- **`i18n` resource type**: `I18nLoader` loads `config/i18n.json` and all JSON files under `i18n/{locale}/`. `I18nService` merges parent resources first, then local application resources, so local translations override inherited keys while missing keys are inherited from base apps or active capabilities.

`ResourceHandler.getHandler(type)` selects the handler for the resource type:

| Resource type (`IResource`) | Handler |
| --- | --- |
| `API_RESOURCE` | `ApiResourceHandler` |
| `PAGE_RESOURCE` | `VueResourceHandler` |
| `COMPONENT_RESOURCE` | `ComponentHandler` |
| `MENU_RESOURCE` | `MenuHandler` |
| `LIBRARY_RESOURCE` | `LibraryHandler` |
| `I18N_RESOURCE` | `I18nService` |

`ResourceLocator.locate(application, resource)` finds the concrete resource by walking the application's `extends` chain. For each extended app whose `category` is `"capability"`, it checks `IActivationStore.isActive(appId, requestingAppId, tenantId)` and **skips inactive capabilities**. This is how capability activation gates resource visibility (see [extensibility.md](extensibility.md)).

## Stage 3 — the API pipeline (`ApiResourceHandler`)

For `api` resources, `ApiResourceHandler.handle(application, request)` runs the full pipeline:

```
1. ResourceLocator.locate(...)                → ApiResource (script body, roles, sourceUri)
2. Security (if roles defined):
     SecurityServiceFactory.create(application, request)
     authenticate(request)  → non-null Response means stop (e.g. redirect to login)
     check(request, roles)  → non-null Response means forbidden
3. Idempotency: IdempotencyService.checkIdempotency(request) → cached Response if present
4. Request scope: databaseService.setCurrentTenant(tenant); beginRequestScope()
5. Create scripting engine + inject bindings (see below)
6. Hook API_PRE_SCRIPT  + dispatch native webhook "api.pre-request"
7. response = scriptingEngine.executeScript(apiScript, sourceUri, request)
8. On success:
     Hook API_POST_SCRIPT + dispatch native webhook "api.post-request"
     IdempotencyService.storeResponse(request, response)
   On exception:
     Hook API_ON_ERROR + dispatch native webhook "api.on-error"
     unwrap BusinessException if present
9. finally: AuditLogger.logApiExecution(userId, tenantId, correlationId, path, status)
            databaseService.endRequestScope(); clearCurrentTenant()
```

Hooks and native webhooks are detailed in [eventing-webhooks.md](eventing-webhooks.md);
idempotency and audit logging in [security-architecture.md](security-architecture.md).

### Binding injection

Before executing the script, the handler injects host objects via `scriptingEngine.putMember(name, value)`. Optional bindings appear only when the
corresponding service is configured:

| Binding | Always? | Source |
| --- | --- | --- |
| `$Request`, `$Response` | yes | the current exchange |
| `$Api`, `$App` | yes | `ApiProxy` / `AppProxy` over the application |
| `$Principal` | yes (may be `null`) | authenticated claims map |
| `$Tenant` | yes (may be `null`) | resolved tenant id |
| `$Logger` | yes | SLF4J logger |
| `$Library` | yes | `LibraryProxy` over `lib/` |
| `$I18n` | yes | `I18nProxy` |
| `$Errors` | yes | `ErrorsProxy` |
| `$DB` | only if a database service is configured | `IDatabaseService` |
| `$Queue` | only if a queue client is configured | `IQueueClient` |
| `$Secrets` | only if a secret provider is configured | `SecretsBinding(provider, tenant)` |
| `$WebHooks` | only if a dispatcher is configured | `WebhookHooksProxy` |

The complete binding contract is in [developer/scripting-bindings.md](../developer/scripting-bindings.md) and [reference/script-bindings-index.md](../reference/script-bindings-index.md).

## Stage 4 — script execution

`PolyglotScriptingEngine` evaluates the script and invokes its three-function contract:

```
loadControlData(request)            → control data map
execute(request, controlData)       → Response | Map | List
emitInsight(request, response, payload)
                                    → Map | List | scalar | null
```

Results from `execute` that are not already a `Response` are wrapped in one. The response body is then parsed as JSON when possible and passed to `emitInsight` as `payload`; if `emitInsight` returns a non-null value and a queue client is configured, the engine publishes the insight to the queue on a best-effort basis. The sandbox, caching, and security configuration are described in [scripting-engine.md](scripting-engine.md).

## Application id from URI

Every transport ultimately routes by application id. `Application.getAppIdFromUri(uri)` takes the first path segment:

```
/payments/api/charge  →  appId = "payments"
```

New transports must use this rule (or supply an explicit `appId` in `contextData`); they must not bypass URI-derived routing.

## Next

- [Scripting engine](scripting-engine.md) — how scripts run safely.
- [Multi-tenancy](multi-tenancy.md) — how the tenant scope is opened and enforced.
- [Eventing & webhooks](eventing-webhooks.md) — the hook points referenced above.
