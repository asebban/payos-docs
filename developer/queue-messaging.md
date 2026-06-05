# Queue messaging (`$Queue`)

When a [queue service](../configuration/queue-service.md) connector is registered, scripts
receive the `$Queue` binding — an `IQueueClient`. Use it to publish messages to a
message-oriented middleware (e.g. NATS) for asynchronous integration. This page covers usage
from JavaScript; the connector configuration is in
[configuration/queue-service.md](../configuration/queue-service.md), and the queue transport
(consuming requests *from* a queue) is in
[architecture/extensibility.md](../architecture/extensibility.md).

## Two distinct concepts

| Concept | Mechanism | Doc |
| --- | --- | --- |
| **Publishing from a script** | `$Queue.publish(...)` | this page |
| **Serving requests delivered over a queue** | the `queue` transport (`payos-server-queue`) | [architecture/extensibility.md](../architecture/extensibility.md) |

This page is about the first: emitting messages from your API logic.

## Publishing

```javascript
function execute(request, controlData) {
    var event = { type: "payment.created", id: controlData.id, tenant: $Tenant };
    $Queue.publish("payments.events", JSON.stringify(event));
    return { id: controlData.id };
}
```

The `IQueueClient` interface provides:

| Method | Purpose |
| --- | --- |
| `publish(topic, message)` | Publish a message to a topic. |
| `publish(topic, message, replyTopic)` | Publish expecting a reply on `replyTopic`. |
| `isConnected()` | Whether the client is connected. |
| `connect()` / `disconnect()` | Lifecycle (managed by the kernel; rarely called from scripts). |

> Message delivery through a MoM client is **asynchronous**. Do not assume a synchronous
> callback after `publish`.

## Registration

The queue client is created from configuration at bootstrap and registered with
`PayOSConfig.setQueueClient(client)`, which makes it available as `$Queue`. The shipped
connector is `queue-service-nats` (`type = "nats"`). See
[configuration/queue-service.md](../configuration/queue-service.md).

## Correlation propagation

Carry the request's correlation and tenant identifiers in your message payloads so async
consumers remain traceable to the originating request:

```javascript
var ctx = $Request.getContextData();
var event = {
    type: "payment.created",
    correlationId: ctx.get("correlationId"),
    tenant: $Tenant,
    payload: controlData
};
$Queue.publish("payments.events", JSON.stringify(event));
```

See [operations/observability.md](../operations/observability.md).

## If `$Queue` is missing

`$Queue` is injected only when a queue client is registered. Configure the
[queue service](../configuration/queue-service.md) and place the `queue-service-nats`
connector JAR on the [connectors path](../configuration/extensions-connectors.md).

## Next

- [Configuration: queue service](../configuration/queue-service.md)
- [Architecture: extensibility (queue transport & connectors)](../architecture/extensibility.md)
