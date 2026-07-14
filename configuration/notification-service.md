# Notification service configuration

The `notification-service` block configures the **publisher-side** notification connector
loaded inside the main PayOS runtime — the `INotificationServiceFactory` that backs the
`$Notification` scripting binding. The shipped implementation is
`payos-notification-connector` (`type = "nats"`), which publishes notification requests onto
a queue for the separate [`payos-service-notification`](../operations/notification-service.md)
daemon to consume and deliver. Don't confuse the two: this page configures the connector that
*publishes* notification requests from inside `BootServer`; the daemon that *delivers* them
(email, retries, fallback channels) has its own independent configuration, documented at
[operations/notification-service.md](../operations/notification-service.md).

## Shape

```json
{
  "notification-service": {
    "configuration": {
      "type": "nats",
      "host": "nats.internal",
      "port": 4222,
      "destination": "notifications.inbound"
    }
  }
}
```

## Keys

From `IConfigSpec.NotificationService`:

| Key | Default | Purpose |
| --- | --- | --- |
| `type` | connector-derived (`nats` for the queue connector) | Connector type; selects the underlying `IQueueClientFactory` (e.g. `nats`). |
| `host` | `localhost` | Broker host. |
| `port` | `4222` | Broker port. |
| `destination` | `notifications.inbound` | Topic/destination used for the connector's own connection. |

The `configuration` map is passed through verbatim (as `Map<String, String>`) to the active
connector's `INotificationServiceFactory#initialize`, so keys beyond `type` are
connector-specific — a non-queue-backed notification connector could define entirely
different keys here.

## Independent from `queue-service`

This block is deliberately **separate** from [`queue-service`](queue-service.md). Before this
was introduced, the queue-backed notification connector reused the generic `$Queue` client
configured by `queue-service`, which meant:

- A `queue-service` that was disabled/misconfigured for unrelated reasons broke every
  `$Notification.send(...)` call too.
- The notification connector couldn't point at a different broker/topic than `$Queue`.

`QueueNotificationServiceFactory.initialize(...)` now connects its own `IQueueClient` via
`QueueClients.create(type)` / `ServiceLoader`, exactly like `QueueServiceInitializer` does for
`$Queue` — but the two connections, and their configuration, are entirely independent.

## Connector discovery

The notification connector JAR (`ma.s2m.payos:payos-notification-connector`) must be on the
[connectors path](extensions-connectors.md). `NotificationServiceInitializer` resolves the
active `INotificationServiceFactory` via `ServiceLoader`, calls `initialize(configuration)`
once at boot (and again on hot-reload) with this block's `configuration` map, then registers
the factory with `PayOSConfig.setNotificationServiceFactory(...)` so scripts see
`$Notification`. If no connector is found on the classpath, this block is ignored and
`$Notification` is unavailable in scripts.

## Next

- [operations/notification-service.md](../operations/notification-service.md) — the separate
  consumer daemon that delivers what this connector publishes.
- [queue-service.md](queue-service.md) — the generic `$Queue` binding, now independent of this
  block.
