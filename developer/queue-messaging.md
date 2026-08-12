# Queue messaging (`$Queue`)

When a [queue service](../configuration/queue-service.md) connector is registered, scripts receive the `$Queue` binding — a `QueueBinding` wrapping the connector's `IQueueClient`. Use it to publish messages to a message-oriented middleware (e.g. NATS) for asynchronous integration. This page covers usage from JavaScript; the connector configuration is in [configuration/queue-service.md](../configuration/queue-service.md), and the queue transport (consuming requests *from* a queue) is in [architecture/extensibility.md](../architecture/extensibility.md).

`QueueBinding` is deliberately publish-only, mirroring how `CacheBinding`/`SlidingWindowBinding` narrow their underlying services rather than exposing the raw interface: it exposes only the `publish(...)` family and `isConnected()`. `subscribe(...)`, `connect(...)`, and `disconnect()` are not exposed at all — see the warning below for why.

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
| `isConnected()` | Whether the client is connected. |

`QueueBinding` (`ma.s2m.payos.scripting.QueueBinding`, `payos` repo) exposes only these — no
`subscribe(...)`, no `connect(...)`/`disconnect()`. See `IQueueClient.java` and `QueueMessage.java`
for the full underlying interface (relevant if you're implementing a new `IQueueClient` connector,
not for script authors).

> Message delivery through a MoM client is **asynchronous**. Do not assume a synchronous
> callback after `publish`.

> ⚠️ **There is no way to subscribe from a script, by design.** A subscription's lifetime must
> match the *process*, not a single script execution — calling `subscribe(...)` on the underlying
> `IQueueClient` from inside a request-scoped script would capture the JS callback in the
> underlying NATS JetStream `Dispatcher`, which invokes it later, asynchronously, on a broker
> client thread — indefinitely pinning the whole per-request GraalVM `Context` in memory (it is
> never explicitly closed), and the underlying JetStream subscription is durable, so a second
> invocation of the same script would likely fail outright (duplicate durable-consumer name).
> This is exactly why `$Queue` doesn't expose `subscribe(...)` at all. Consuming a queue —
> including receiving a reply on a `replyTo` destination — always means a dedicated
> `payos-server-queue` instance (a `protocol: "queue"` entry in `servers[]`, subscribed exactly
> once at boot, in Java, for the life of the process). See
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
service-adapter JAR on the [service-adapters path](../configuration/extensions-connectors.md).

## Next

- [Queue setup guide (de A à Z)](queue-setup-guide.md) — configuring and using both the publisher and consumer sides end to end, with a full worked example.
- [Configuration: queue service](../configuration/queue-service.md)
- [Architecture: extensibility (queue transport & connectors)](../architecture/extensibility.md)
