# Webhooks & hooks

PayOS reacts to [system events](../architecture/eventing-webhooks.md) in two ways: in-process
**hook scripts** and outbound **native webhooks**. This page is the developer's how-to; the
model is in [architecture/eventing-webhooks.md](../architecture/eventing-webhooks.md) and the
dispatcher configuration is in
[configuration/webhook-service.md](../configuration/webhook-service.md).

## Hook scripts vs. native webhooks

| | Hook script | Native webhook |
| --- | --- | --- |
| Runs | In-process JavaScript (sandboxed) | Outbound HTTP to an external URL |
| Driven by | `HookEngine.execute(HookPoint, …)` | `IWebhookDispatcher.dispatch(WebhookEvent)` |
| Use for | Observing/augmenting processing | Notifying external systems |

Both fire from the same event points (e.g. `api.pre-request`, `api.post-request`,
`api.on-error`).

## Defining native webhook subscriptions

Subscriptions are declared in an application's `config/webhooks.json`. Each entry:

```json
{
  "webhooks": [
    {
      "id": "notify-ledger",
      "event": "api.post-request",
      "native": true,
      "url": "https://ledger.internal/events",
      "method": "POST",
      "secret": "${LEDGER_WEBHOOK_SECRET}",
      "headers": { "X-Source": "payos" },
      "filter": {
        "path": "/payments/api/payment",
        "method": "POST",
        "statusCodes": [200, 201]
      }
    }
  ]
}
```

| Field | Purpose |
| --- | --- |
| `id` | Subscription identifier. |
| `event` | The [system event](../reference/system-events.md) to subscribe to. |
| `native` | `true` for kernel-dispatched native webhooks. |
| `url` | Destination URL. |
| `method` | HTTP method (default `POST`). |
| `secret` | Signing secret. |
| `headers` | Extra headers to send. |
| `disabled` | Disable without removing. |
| `filter.path` / `filter.method` / `filter.statusCodes` | Restrict which requests trigger delivery. |

## Writing hook scripts

Hook scripts run JavaScript at a hook point using `evalScript` (no
`loadControlData`/`execute` contract). They observe the request/response context. Place them
where the application's hook resolution expects them; the engine invalidates its hook cache
on [configuration reload](../operations/hot-reload.md).

```javascript
// a post-request hook: log request completion
$Logger.info("api.post-request path=" + $Request.getPath() +
    " user=" + ($Principal && $Principal.get("id")));
```

There is no `$Audit` script binding — the PCI-DSS audit trail (`AuditLogger`/`AuditEvent`) is a
Java-side facade only, not reachable from scripts. `$Logger` (always available, a plain SLF4J
logger) is the binding to use from a hook script.

## The `$WebHooks` binding

When a dispatcher is configured, scripts can interact with webhooks programmatically via
`$WebHooks` (a `WebhookHooksProxy`) — for example, to trigger a delivery explicitly from
business logic.

## Delivery characteristics

- Delivery is asynchronous; do not block business logic on webhook completion.
- The HTTP dispatcher honors `connectTimeoutMs` (default 5000) and `requestTimeoutMs`
  (default 10000) — see [configuration/webhook-service.md](../configuration/webhook-service.md).
- Each delivery carries the request's `X-Correlation-Id` and `X-Tenant-Id` for traceability.

## Next

- [Architecture: eventing & webhooks](../architecture/eventing-webhooks.md)
- [Configuration: webhook service](../configuration/webhook-service.md)
- [Reference: system events](../reference/system-events.md)
