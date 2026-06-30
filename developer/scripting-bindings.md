# Scripting bindings reference

Before a script runs, the kernel injects a set of `$`-prefixed **bindings** into the
scripting context via `scriptingEngine.putMember(name, value)`. They are the only sanctioned
way for JavaScript to reach the platform (the sandbox blocks ambient I/O — see
[architecture/scripting-engine.md](../architecture/scripting-engine.md)).

This page is the authoritative per-binding reference. For a one-line lookup table, see
[reference/script-bindings-index.md](../reference/script-bindings-index.md).

## Always available

These are injected for every API request.

### `$Request`

The transport-neutral [`Request`](../architecture/request-processing.md). Body, headers,
parameters, method, path, and context data (tenant, correlation, appId). Also passed as the
`request` argument to `loadControlData` / `execute`.

```javascript
var body = JSON.parse($Request.getBodyAsString() || "{}");
var corr = $Request.getContextData().get("correlationId");
```

### `$Response`

The mutable [`Response`](../architecture/request-processing.md) for the current call. Set
status, headers, and (optionally) body explicitly. Also exposes `contextData`, a
`Map<String, Object>` for passing backend-internal data from the script to later Java
processing — it is never serialized into the HTTP response.

```javascript
$Response.setStatusCode(200);
$Response.setHeader("X-Custom", "value");
$Response.addContextData("skipAudit", true);
```

### `$Api`

API invocation utility (`ApiProxy`) that calls APIs within the current application or falls
back to remote HTTP/HTTPS endpoints when the local API is not found. Provides `get()`,
`post()`, `put()`, and `delete()` methods.

```javascript
// Call a local API within the same bundle
var response = $Api.get("/orders/123");

// If local API not found and path is a valid HTTP URL, falls back to remote call
var remoteResponse = $Api.post("http://payment-service:8080/process", JSON.stringify({amount: 100}));
```

### `$App`

The current [`Application`](application-model.md) — its id, name, version, and configuration.

### `$Principal` / `$User`

The authenticated principal as a map: `id` (from the OIDC `sub`), `email`, `name`,
`preferred_username`, and `roles`. Present when the request is authenticated (see
[security-architecture](../architecture/security-architecture.md)).

```javascript
var userId = $Principal.get("id");
var roles  = $Principal.get("roles");
```

### `$Tenant`

The resolved tenant identifier for the current request (see
[multi-tenancy](../architecture/multi-tenancy.md)).

## Available when configured

These bindings are injected only when the corresponding service is configured at bootstrap.
If you use one without configuring its service, it will be absent.

### `$DB`

The database access service (`IDatabaseService`). Backed by the
[database connector](../configuration/database-service.md). See [data access](data-access.md).

```javascript
var rows = $DB.find("SELECT * FROM accounts WHERE tenant = ?", [$Tenant]);
```

### `$Queue`

The message-queue client (`IQueueClient`), registered via `PayOSConfig.setQueueClient`.
Backed by a [queue connector](../configuration/queue-service.md) such as NATS. See
[queue messaging](queue-messaging.md).

```javascript
$Queue.publish("payments.events", JSON.stringify({ id: 42 }));
```

### `$Secrets`

The secret provider (`ISecretProvider`), backed by a [secret service](../configuration/secret-service.md) (`filesystem` or `vault`). See [secrets usage](secrets-usage.md).

```javascript
var apiKey = $Secrets.getSecret("psp-api-key");
```

### `$I18n`

The internationalization accessor, resolving messages for the request's locale (see
[internationalization](internationalization.md) and [i18n config](../configuration/i18n.md)).

```javascript
var msg = $I18n.t("payment.declined");
```

### `$WebHooks`

The webhook hooks proxy (`WebhookHooksProxy`), available when a
[webhook dispatcher](../configuration/webhook-service.md) is configured. See
[webhooks & hooks](webhooks-and-hooks.md).

### `$Library`

Loads shared JavaScript from the application's `lib/` directory (see
[writing APIs](writing-apis.md#using-shared-libraries)).

## Java interop

In addition to bindings, scripts may reach whitelisted Java classes via `Java.type()` (the
sandbox blocks a denylist such as `java.lang.System`). See
[Java extensions](java-extensions.md).

## Injection order

Bindings are injected by `ApiResourceHandler` immediately before script execution, after the
tenant scope is opened and security has run. Optional bindings (`$DB`, `$Queue`, `$Secrets`,
`$I18n`, `$WebHooks`) are injected only when their service is present. The full pipeline is
in [architecture/request-processing.md](../architecture/request-processing.md).
