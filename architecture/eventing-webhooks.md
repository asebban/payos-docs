# Eventing & webhooks

PayOS emits **system events** at well-defined points in the request and lifecycle flows.
Each event can trigger two things: in-process **hook scripts** (run by the `HookEngine`) and
outbound **native webhooks** (delivered by the webhook dispatcher). This document describes
the model; the configuration of webhook subscriptions is in
[configuration/webhook-service.md](../configuration/webhook-service.md) and the developer
view in [developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

## Two reactions to one event

```
        ┌─────────────── system event ───────────────┐
        │                                             │
   HookEngine.execute(HookPoint, ctx, engine)   WebhookDispatcher.dispatch(WebhookEvent)
   (runs in-process hook script, sandboxed)     (delivers HTTP webhook to subscribers)
```

- **Hooks** run JavaScript in the same [sandbox](scripting-engine.md) as application code,
  using `evalScript` (no `execute`/`loadControlData` contract). They can observe and
  influence processing.
- **Native webhooks** are asynchronous outbound notifications to external systems,
  delivered through the `IWebhookDispatcher` SPI.

## System events

The catalog (from `IConfigSpec.Webhooks.SystemEvents`):

| Event | Fired when |
| --- | --- |
| `api.pre-request` | Before an API script executes. |
| `api.post-request` | After an API script succeeds. |
| `api.on-error` | When an API script throws. |
| `security.login` | On successful authentication. |
| `security.logout` | On logout. |
| `capability.activated` | A capability is activated. |
| `capability.deactivated` | A capability is deactivated. |
| `capability.installed` | A capability is installed. |
| `capability.uninstalled` | A capability is uninstalled. |
| `page.pre-serve` | Before a page is served. |
| `page.post-serve` | After a page is served. |
| `page.on-error` | When page rendering fails. |

The exhaustive index is in [reference/system-events.md](../reference/system-events.md).

### Where the API events fire

In `ApiResourceHandler` (see [request-processing.md](request-processing.md)):

```
... binding injection ...
HookEngine.execute(API_PRE_SCRIPT, ctx, engine);   dispatch "api.pre-request"
response = engine.executeScript(...);
  success → HookEngine.execute(API_POST_SCRIPT, ...); dispatch "api.post-request"
  error   → HookEngine.execute(API_ON_ERROR, ...);    dispatch "api.on-error"
```

Hook points map to `HookPoint` enum values (`API_PRE_SCRIPT`, `API_POST_SCRIPT`,
`API_ON_ERROR`, …). The hook cache is invalidated on configuration reload.

## The webhook dispatcher SPI

The kernel defines the dispatcher contract in `ma.s2m.payos.hooks`:

```java
interface IWebhookDispatcher        { void dispatch(WebhookEvent event); }
interface IWebhookDispatcherFactory { String type(); IWebhookDispatcher create(Map<String,Object> config); }
```

The shipped implementation is `webhook-service-http` (`type = "http"`,
`HttpWebhookDispatcher`). It is registered at bootstrap by `WebhookServiceInitializer` via
`WebhookDispatchers.create(type, config)` and stored with
`PayOSConfig.setWebhookDispatcher(...)`.

> Unlike the queue/secret connectors, the webhook factory is discovered with a **standard**
> `ServiceLoader` (not the connector classloader). The dispatcher is selected by the
> bootstrap key `webhooks.dispatcher` (default `"http"`).

## Webhook subscriptions

Applications subscribe to events through a per-application `webhooks.json`
(`IConfigSpec.Webhooks`). Each entry:

| Field | Purpose |
| --- | --- |
| `id` | Subscription identifier. |
| `event` | The system event to subscribe to. |
| `native` | Whether this is a native (kernel-dispatched) webhook. |
| `url` | Destination URL. |
| `secret` | Signing secret. |
| `method` | HTTP method. |
| `headers` | Extra headers. |
| `disabled` | Disable without removing. |
| `filter` → `path`, `method`, `statusCodes` | Restrict which requests trigger delivery. |

Configuration detail: [configuration/webhook-service.md](../configuration/webhook-service.md).

## Dispatcher service configuration

The HTTP dispatcher itself is configured under `http-webhook-service`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `true` | Enable the dispatcher. |
| `connectTimeoutMs` | `5000` | Connection timeout. |
| `requestTimeoutMs` | `10000` | Request timeout. |

## The `$WebHooks` binding

When a dispatcher is configured, scripts receive `$WebHooks` (a `WebhookHooksProxy`) for
triggering or interacting with webhooks programmatically. See
[developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

## Correlation & tenant propagation

Webhook deliveries and hook executions carry the request's `X-Correlation-Id` and
`X-Tenant-Id` so that asynchronous side effects remain traceable to their originating
request. See [operations/observability.md](../operations/observability.md).

## Next

- [Developer: webhooks & hooks](../developer/webhooks-and-hooks.md) — defining subscriptions.
- [Reference: system events](../reference/system-events.md) — the exhaustive event list.
