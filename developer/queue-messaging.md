# Queue messaging (`$Queue`)

When a [queue service](../configuration/queue-service.md) connector is registered, scripts receive the `$Queue` binding — an `IQueueClient`. Use it to publish messages to a message-oriented middleware (e.g. NATS) for asynchronous integration. This page covers usage from JavaScript; the connector configuration is in [configuration/queue-service.md](../configuration/queue-service.md), and the queue transport (consuming requests *from* a queue) is in [architecture/extensibility.md](../architecture/extensibility.md).

## Two distinct concepts

| Concept | Mechanism | Doc |
| --- | --- | --- |
| **Publishing from a script** | `$Queue.publish(...)` | this page |
| **Serving requests delivered over a queue** | the `queue` transport (`payos-server-queue`) | [architecture/extensibility.md](../architecture/extensibility.md) |

This page is about the first: emitting messages from your API logic.

## Publishing

`IQueueClient.publish(...)` has three real overloads — there is **no** `publish(topic, message)`
two-string overload. Pick the one matching your use case:

```javascript
function execute(request, controlData) {
    var payload = JSON.stringify({ type: "payment.created", id: controlData.id, tenant: $Tenant });

    // Destination-based publish — the only overload that lets you target an arbitrary
    // destination; requires a QueueMessage envelope, not a raw string.
    var message = new (Java.type("ma.s2m.payos.queue.QueueMessage"))(
        controlData.id, payload, {}, "payments.events");
    $Queue.publish("payments.events", message);

    return { id: controlData.id };
}
```

| Method | Purpose |
| --- | --- |
| `publish(message)` | Publish a raw string message to the connector's default configured topic (set at `connect(host, port, topic)` time). |
| `publish(message, replyTopic)` | Same as above, but requests a reply on `replyTopic` (or the default topic if `replyTopic` is null/blank). **`replyTopic` is a reply-to topic, not a destination.** |
| `publish(destination, QueueMessage)` | Broker-agnostic publish to an explicit `destination` — no implicit default topic. Returns a `MessageHandle`. This is the overload to use when you need to target a specific destination from a script. |
| `subscribe(listener)` | Subscribe to the default topic (legacy `MessageListener`, no ack/nack). |
| `subscribe(destination, AckMessageListener)` | Subscribe to an explicit destination with per-message acknowledge/reject control. Returns a `SubscriptionHandle`. |
| `subscribe(List<String> destinations, AckMessageListener)` | Subscribe the same listener to multiple destinations at once; `QueueMessage.getDestination()` tells the listener which one a given delivery came from. |
| `isConnected()` | Whether the client is connected. |
| `connect(...)` / `disconnect()` | Lifecycle (managed by the kernel; rarely called from scripts). |

See `payos/src/main/java/ma/s2m/payos/queue/IQueueClient.java` and `QueueMessage.java` for the
authoritative signatures.

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
var message = new (Java.type("ma.s2m.payos.queue.QueueMessage"))(
    controlData.id, JSON.stringify(event), {}, "payments.events");
$Queue.publish("payments.events", message);
```

See [operations/observability.md](../operations/observability.md).

## If `$Queue` is missing

`$Queue` is injected only when a queue client is registered. Configure the
[queue service](../configuration/queue-service.md) and place the `queue-service-nats`
connector JAR on the [connectors path](../configuration/extensions-connectors.md).

## Next

- [Configuration: queue service](../configuration/queue-service.md)
- [Architecture: extensibility (queue transport & connectors)](../architecture/extensibility.md)
