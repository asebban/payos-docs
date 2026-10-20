# Queue service configuration

The `queue-service` block configures a message-oriented-middleware connector. The shipped
implementation is `queue-service-nats` (`type = "nats"`). The connector registers an
`IQueueClient` exposed to scripts as `$Queue`. Developer usage is in
[developer/queue-messaging.md](../developer/queue-messaging.md); the queue **transport**
(serving requests delivered over a queue) is configured in
[servers.md](servers.md) and described in
[architecture/extensibility.md](../architecture/extensibility.md).

## Shape

```json
{
  "queue-service": {
    "configuration": {
      "enabled": true,
      "name": "primary",
      "type": "nats",
      "host": "localhost",
      "port": 4222,
      "publisher-topic": "payos.events",
      "consumer-topic": "payos.requests"
    }
  }
}
```

## Keys

From `IConfigSpec.QueueService`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `false` | Enable initialization of the queue client at bootstrap. |
| `name` | — | Logical name of the queue service. |
| `type` | — | Connector type; selects the `IQueueClientFactory` (e.g. `nats`). |
| `host` | `localhost` | Broker host. |
| `port` | `4222` (NATS) | Broker port. |
| `publisher-topic` | — | Default topic for outbound `$Queue.publish`. |
| `consumer-topic` | — | Topic consumed by the queue transport (`default-topic` if unset). |

## Service-adapter discovery

The NATS service-adapter JAR (`ma.s2m.payos:queue-service-nats`) must be on the
[service-adapters path](extensions-connectors.md). The factory is selected by `type` via
`QueueClients.create(type)` / `ServiceLoader`, and the resulting client is registered with
`PayOSConfig.setQueueClient(...)` so scripts see `$Queue`.

## Publish vs. serve

| You want to… | Configure | Doc |
| --- | --- | --- |
| Publish messages from a script (`$Queue`) | `queue-service` (this page) | [developer/queue-messaging.md](../developer/queue-messaging.md) |
| Serve requests that arrive over a queue | a `queue` entry in `servers` | [servers.md](servers.md) |

`$Queue.publish(destination, message[, replyTo])` — the only `publish(...)` family `$Queue` exposes (`message` a plain `Map`/object or a `QueueMessage`) — no longer calls this connector's client directly: it writes a Transactional Outbox row instead, drained afterward by a dispatcher paired with the `queue` entry in `servers[]` (`maxAttempts`/`pollIntervalSeconds`, see [servers.md](servers.md#queue-keys-queue)). The two legacy, implicit-default-topic overloads (`publish(message)`/`publish(message, replyTopic)`) were removed from `$Queue` entirely rather than left unprotected — see [architecture/atomicity-db-audit-queue-v9-2026-10-19.md](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md).

## Correlation

The queue transport derives the correlation id from the message payload, then a header, then
generates a UUID. Carry correlation/tenant in your published payloads for traceability — see
[operations/observability.md](../operations/observability.md).

## Next

- [developer/queue-setup-guide.md](../developer/queue-setup-guide.md) — configuring and using both publisher and consumer sides end to end, with a full worked example.
- [developer/queue-messaging.md](../developer/queue-messaging.md)
- [servers.md](servers.md) — the queue transport.
