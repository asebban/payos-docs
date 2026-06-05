# Webhook service configuration

PayOS delivers outbound **native webhooks** through an `IWebhookDispatcher`. The shipped
implementation is `webhook-service-http` (`type = "http"`). Two configuration blocks are
involved: `webhooks` (selects the dispatcher) and `http-webhook-service` (configures the HTTP
dispatcher). The model is in
[architecture/eventing-webhooks.md](../architecture/eventing-webhooks.md); developer usage is
in [developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

## Selecting the dispatcher

```json
{
  "webhooks": {
    "dispatcher": "http"
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `webhooks.dispatcher` | `http` | Dispatcher type; selects the `IWebhookDispatcherFactory`. |

> The webhook factory is discovered with a **standard** `ServiceLoader` (not the connector
> classloader). At bootstrap, `WebhookServiceInitializer` calls
> `WebhookDispatchers.create(type, config)` and stores the result via
> `PayOSConfig.setWebhookDispatcher(...)`.

## HTTP dispatcher settings

```json
{
  "http-webhook-service": {
    "enabled": true,
    "connectTimeoutMs": 5000,
    "requestTimeoutMs": 10000
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `true` | Enable the HTTP dispatcher. |
| `connectTimeoutMs` | `5000` | Connection timeout (ms). |
| `requestTimeoutMs` | `10000` | Request timeout (ms). |

## Per-application subscriptions

Which events trigger a delivery — and where — is declared per application in
`config/webhooks.json`, not in the bootstrap. Each subscription specifies the `event`,
target `url`, `method`, signing `secret`, and an optional `filter` (`path`, `method`,
`statusCodes`). The full subscription schema is in
[developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

## Events

Subscriptions reference [system events](../reference/system-events.md) such as
`api.post-request`, `api.on-error`, `security.login`, and the capability/page events.

## Observability

Deliveries carry the originating request's `X-Correlation-Id` and `X-Tenant-Id`. See
[operations/observability.md](../operations/observability.md).

## Next

- [Architecture: eventing & webhooks](../architecture/eventing-webhooks.md)
- [Developer: webhooks & hooks](../developer/webhooks-and-hooks.md)
- [Reference: system events](../reference/system-events.md)
