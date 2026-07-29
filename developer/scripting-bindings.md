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

### `$Analytics`, `$Metrics`, `$Integration`, `$EventStore`

Four bindings over the event-category facades built in
[event-category-payload-contracts-v7-2026-07-28.md](event-category-payload-contracts-v7-2026-07-28.md)
— business/usage tracking, numeric time series, external choreography, and durable replay,
respectively. Unlike every other binding in this section, they wrap a **static facade**
(`AnalyticsRecorder`/`MetricsRecorder`/`IntegrationEventPublisher`/`EventStore`, all in
`payos-kernel`), not a per-request service instance — the facade always resolves to at least an
SLF4J default via SPI, so all four are injected unconditionally, exactly like `$Logger`/`$Errors`.
Each pre-fills `tenantId`/`correlationId` from the current request so a script never passes
either explicitly, the same ergonomic pattern `$Cache`/`$SlidingWindow` use for tenant
namespacing.

```javascript
$Analytics.logEvent("checkout_started", { plan: "pro" });
$Analytics.logEvent("checkout_started", { plan: "pro" }, $Principal.get("id")); // explicit actorId

$Metrics.record("connector.execution.duration", "histogram", 42.0, "ms",
        { connectorType: "CardNetwork", outcome: "SUCCESS" });

$Integration.publish("ma.s2m.payos.payment.completed.v1", { paymentId: paymentId, amount: 1500 });

$EventStore.append("payment-" + paymentId, "Payment", "PaymentAuthorized",
        1 /* sequenceNumber */, 1 /* schemaVersion */, { amount: 1500, authCode: "00" });
var history = $EventStore.replay("payment-" + paymentId); // array of plain objects, oldest first
```

**Error semantics differ per binding, matching each category's own delivery guarantee:**
`$Analytics`, `$Metrics`, and `$Integration` are **best-effort** — any failure (a bad call from
the script included) is logged at `warn` and never thrown, so a broken analytics/metrics/
integration call can never break the surrounding script. `$EventStore` is the one exception: it
**propagates errors** (e.g. `sequenceNumber` out of order raises `IllegalStateException`), because
event-sourcing correctness depends on every fact actually landing — silently swallowing a bad
append would corrupt replay with no visible error. Wrap `$EventStore.append(...)` in `try/catch`
if the script needs to handle that itself.

`$EventStore.append(...)` has a second overload taking an explicit `occurredAtEpochMilli` as its
last argument, for backfill scenarios where the fact happened before "now" — the default overload
uses the current instant, which is correct for the common case of a script emitting an event live.

### `$Audit`, `$Diagnostics`

The remaining two of the six event categories, both pre-existing "reference shape" categories
(§1 and §6 of the contract doc) rather than newly built — but only now, alongside the four
above, exposed to scripts.

`$Audit` wraps `AuditLogger` but exposes **only** its free-form `logEvent(eventType, result,
extra)` entry point — deliberately none of `IAuditLogger`'s typed lifecycle methods
(`logAuthSuccess`, `logSessionCreated`, `logApiExecution`, `logStartup`, ...). Those are system
events the kernel already emits itself at the right moment; letting a script call them too would
be redundant at best and, at worst, let it fabricate a fake `AUTH_SUCCESS` record in a trail
that's supposed to be immutable/WORM. `tenantId`/`correlationId`/`appId`/`path`/`userId` are all
pre-filled from the current request/principal — the script only supplies what's actually
event-specific.

```javascript
$Audit.logEvent("CARD_TOKENISED", "SUCCESS", { maskedPan: "************1234", tokenId: "tok_xyz" });
```

`$Diagnostics` wraps `Diagnostics` with a single generic `logEvent(nature, stage, errorCode,
rootCauseCategory, attemptCount, reason, details)` — `DiagnosticEvent` is already nature-agnostic
(see [connector-framework-usage.md](connector-framework-usage.md) for the `nature="connector"`
precedent), so a script can emit its own `nature` (e.g. `"script"`) without needing a dedicated
method.

```javascript
$Diagnostics.logEvent("script", "VALIDATION_FAILED", "BAD_INPUT", null, 1,
        "missing required field 'amount'", { field: "amount" });
```

**Error semantics**: `$Audit` **propagates errors** (like `$EventStore`) — the audit trail "must
never lose an event," so a blank `eventType` must surface immediately, not be silently dropped.
`$Diagnostics` is **best-effort** (like `$Analytics`/`$Metrics`/`$Integration`) — the category
tolerates loss entirely by design (short retention, WARN-level logs).

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

A `QueueBinding` wrapping the message-queue client (`IQueueClient`), registered via
`PayOSConfig.setQueueClient`. Backed by a [queue connector](../configuration/queue-service.md)
such as NATS. Publish-only: exposes `publish(...)` and `isConnected()` only — `subscribe(...)` is
not reachable from scripts at all (a subscription's lifetime must match the process, not a single
script execution; see [queue messaging](queue-messaging.md) for why). Consuming a queue always
means a dedicated `payos-server-queue` instance instead — see
[queue setup guide §4](queue-setup-guide.md#4-configurer-et-utiliser-le-côté-consumer).

```javascript
// publish(destination, QueueMessage) — the overload for targeting an explicit destination.
// There is no publish(topic, message) two-string overload; see queue-messaging.md for the
// full set of publish overloads.
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

> **Reachable in a running deployment since 2026-07-27.** `BootServer` calls
> `ConnectorFrameworkInitializer.initialize(...)`, which sets the registry from
> `connectors.json` + `<runtimeBaseDir>/connectors/`. `$Connector` is absent only when no
> connectors are configured/found — a normal, fully-supported state, not a wiring gap.

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
tenant scope is opened and security has run. `HookEngine.injectCommonBindings(...)` duplicates
the same wiring for the standalone hook-execution path — the two must be kept in sync when adding
a binding. Optional bindings (`$DB`, `$Queue`, `$Secrets`, `$Cache`, `$SlidingWindow`, `$I18n`,
`$WebHooks`, `$Connector`, `$Notification`) are injected only when their service is present.
`$Analytics`, `$Metrics`, `$Integration`, `$EventStore`, `$Audit`, and `$Diagnostics` are always
injected, like `$Logger`/`$Errors` — see their entries above for why. The full pipeline is in
[architecture/request-processing.md](../architecture/request-processing.md).
