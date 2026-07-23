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
| `subscribe(listener)` / `subscribe(destination, AckMessageListener)` / `subscribe(List<String>, AckMessageListener)` | **Never call these from a script.** `$Queue` is the raw `IQueueClient`, so these methods are technically reachable, but a subscription's lifetime must match the *process*, not a single script execution — see the warning below. |
| `isConnected()` | Whether the client is connected. |
| `connect(...)` / `disconnect()` | Lifecycle (managed by the kernel; rarely called from scripts). |

See `payos/src/main/java/ma/s2m/payos/queue/IQueueClient.java` and `QueueMessage.java` for the
authoritative signatures.

> Message delivery through a MoM client is **asynchronous**. Do not assume a synchronous
> callback after `publish`.

> ⚠️ **Do not call `$Queue.subscribe(...)` from a script.** The JS callback you'd pass in gets
> captured by the underlying NATS JetStream `Dispatcher` and invoked later, asynchronously, on a
> broker client thread — indefinitely pinning the whole per-request GraalVM `Context` in memory
> (it is never explicitly closed), and the underlying JetStream subscription is durable, so a
> second invocation of the same script will likely fail outright (duplicate durable-consumer
> name). Consuming a queue — including receiving a reply on a `replyTo` destination — always
> means a dedicated `payos-server-queue` instance (a `protocol: "queue"` entry in `servers[]`,
> subscribed exactly once at boot, in Java, for the life of the process), never an inline
> `subscribe(...)` call from request-scoped script code. See
> [Queue setup guide §4](queue-setup-guide.md#4-configurer-et-utiliser-le-côté-consumer) for the
> consumer-side setup, and
> [§4.4](queue-setup-guide.md#44-réponse-et-pattern-requêteréponse-optionnel) for the correct
> reply pattern.

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

- [Queue setup guide (de A à Z)](queue-setup-guide.md) — configuring and using both the publisher and consumer sides end to end, with a full worked example.
- [Configuration: queue service](../configuration/queue-service.md)
- [Architecture: extensibility (queue transport & connectors)](../architecture/extensibility.md)
