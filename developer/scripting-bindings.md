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

`getHeaders()` returns a Java `Map<String, String>`, not a plain JS object — iterate it with
`entrySet()` in a `for...of` loop rather than `Object.keys()`/`for...in`:

```javascript
for (var entry of $Request.getHeaders().entrySet()) {
    $Logger.info(entry.getKey() + ": " + entry.getValue());
}
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

### `$Logger`

A plain SLF4J logger. The simplest way to emit a log line from a script; unlike `$WebHooks` or
the audit trail, it has no structured event contract — just message + level.

```javascript
$Logger.info("processing order " + orderId);
$Logger.warn("unexpected discount code: " + code);
```

> **`console.log` also works** — GraalJS provides a built-in `console` global (`log`/`info`/
> `warn`/`error`) independently of the `$`-binding mechanism, writing to the host process's raw
> `stdout`/`stderr`. It is convenient for ad hoc local debugging, but unlike `$Logger` it does
> not go through SLF4J: it bypasses whatever log level/appender/aggregation the deployment has
> configured, and its output is not tagged with the structured fields (`tenantId`,
> `correlationId`, ...) that the platform's own log lines carry. Prefer `$Logger` for anything
> that needs to be visible in production logs.

### `$Errors`

`ErrorsProxy` — throws typed, script-facing business errors that the kernel translates into
the appropriate HTTP response.

```javascript
if (!order) {
    $Errors.notFound("Order not found");
}
if (amount <= 0) {
    $Errors.badRequest("amount must be positive");
}
```

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
// publish(destination, QueueMessage) — the overload for targeting an explicit destination.
// There is no publish(topic, message) two-string overload; see queue-messaging.md for the
// full set of publish/subscribe overloads.
var message = new (Java.type("ma.s2m.payos.queue.QueueMessage"))(
    orderId, JSON.stringify({ id: 42 }), {}, "payments.events");
$Queue.publish("payments.events", message);
```

### `$Secrets`

A `SecretsBinding` wrapping the tenant's configured `ISecretProvider`, backed by a
[secret service](../configuration/secret-service.md) (`filesystem` or `vault`). Exposes
`get`/`list`/`tokenize`/`detokenize` only — no writes. See [secrets usage](secrets-usage.md).

```javascript
var apiKey = $Secrets.get("psp-api-key");
```

### `$Cache`

A `CacheBinding` wrapping the resolved `ICacheStore` (`memory` or `redis`), backed by the
[cache service](../configuration/cache-service.md). Exposes `put`/`get`/`remove`/`exists`/
`increment` for data shared across every instance of a bundle, or across bundles on the same
cluster with the `redis` backend. Like `$Secrets`, it automatically namespaces every key by the
current tenant — a script never passes a tenant explicitly, and never collides with another
tenant reusing the same key name. See [cache usage](cache-usage.md).

```javascript
$Cache.put("quote:" + quoteId, JSON.stringify(quote), 60);
var raw = $Cache.get("quote:" + quoteId); // null if missing/expired
```

### `$SlidingWindow`

A `SlidingWindowBinding` wrapping the resolved `ISlidingWindowCounter` (`memory` or `redis`),
backed by the [sliding window counter service](../configuration/sliding-window-service.md). Unlike
`$Cache.increment`'s fixed/tumbling window, this counts events in an exact trailing window ending
"now". **Read-only**: only `count(key, windowMillis)` is exposed — a script can check the current
usage for a quota decision but cannot record a new event or reset the counter. Namespaces every
key by the current tenant, like `$Cache`. See [sliding window counter usage](sliding-window-usage.md).

```javascript
var callsThisHour = $SlidingWindow.count("api-calls", 3600000); // windowMillis, not seconds
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

### `$Connector`

A `ConnectorBinding`, injected when a `TenantConnectorRegistry` is set via
`PayOSConfig.setConnectorRegistry(...)`. This is the business/payment connector framework —
see [connector-framework-usage.md](connector-framework-usage.md) for the full script-caller
guide (resolution rules, idempotency behavior, error handling) and
[configuration/connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md)
for the full configuration and behavioral contract (idempotency, deduplication, retry, DLQ
routing, diagnostics).

> **Not yet reachable in a running deployment.** `PayOSConfig.setConnectorRegistry(...)` is
> defined but `BootServer` never calls it — so `$Connector` is always absent today unless
> something else in your bundle wires a registry in explicitly.

```javascript
var handle = $Connector('CardNetwork', 'visa'); // or $Connector('CardNetwork') for the first match
var response = handle.execute({ amount: 100, currency: "MAD" });

if (response.status() == "SUCCESS") {
    return response.data();
}
// response.errorCategory(): NONE | RETRYABLE_ERROR | PERMANENT_ERROR | TIMEOUT | UNKNOWN_ERROR
$Errors.badRequest(response.errorMessage());
```

`.execute(payload)` is the connector's **sole** interaction method — there is no separate
lookup/connect step. The idempotency key (if any) is not a parameter to `.execute()`; it comes
from the `X-Idempotency-Key` request header and is bound in automatically. Deduplication,
retry scheduling, execution-state persistence, and DLQ/terminal-state routing are all handled
by the platform, never by the connector or the calling script — see the configuration doc
linked above for the full decision sequence.

### `$Notification`

A `NotificationBinding`, injected when a notification service factory is registered via
`PayOSConfig.setNotificationServiceFactory(...)` (this **is** wired into `BootServer` today,
unlike `$Connector`). See [configuration/notification-service.md](../configuration/notification-service.md).

```javascript
var result = $Notification.send(
    "Your payment of 100 MAD was received", "+212600000000", "sms");
```

## Java interop

In addition to bindings, scripts may reach whitelisted Java classes via `Java.type()` (the
sandbox blocks a denylist such as `java.lang.System`). See
[Java extensions](java-extensions.md).

## Injection order

Bindings are injected by `ApiResourceHandler` immediately before script execution, after the
tenant scope is opened and security has run. Optional bindings (`$DB`, `$Queue`, `$Secrets`,
`$Cache`, `$SlidingWindow`, `$I18n`, `$WebHooks`, `$Connector`, `$Notification`) are injected only
when their service is present. The full pipeline is in [architecture/request-processing.md](../architecture/request-processing.md).
