Created: 2026-10-16
Last updated: 2026-10-17
Version: 4 — supersedes v3: `publish(...)` gained a plain-`Map` form (no `QueueMessage`/`Java.type(...)` construction needed) and an optional trailing `replyTo` — see [The `publish(...)` overloads](#the-publish-overloads).

# Queue messaging (`$Queue`)

When a [queue service](../configuration/queue-service.md) connector is registered, scripts receive the `$Queue` binding — a `QueueBinding`. Use it to publish messages to a message-oriented middleware (e.g. NATS) for asynchronous integration. This page covers usage from JavaScript; the connector configuration is in [configuration/queue-service.md](../configuration/queue-service.md), and the queue transport (consuming requests *from* a queue) is in [architecture/extensibility.md](../architecture/extensibility.md).

`QueueBinding` is deliberately narrow, mirroring how `CacheBinding`/`SlidingWindowBinding` narrow their underlying services rather than exposing the raw interface: it exposes only the `publish(...)` family below and `isConnected()`. `subscribe(...)`, `connect(...)`, and `disconnect()` are not exposed at all — see the warning below for why.

## Two distinct concepts

| Concept | Mechanism | Doc |
| --- | --- | --- |
| **Publishing from a script** | `$Queue.publish(destination, message[, replyTo])` | this page |
| **Serving requests delivered over a queue** | the `queue` transport (`payos-server-queue`) | [architecture/extensibility.md](../architecture/extensibility.md) |

This page is about the first: emitting messages from your API logic.

## The `publish(...)` overloads

`IQueueClient.publish(...)` itself has three overloads, but `$Queue` exposes a different set — four, all destination-based, none of them the legacy implicit-default-topic ones. The two legacy overloads (`publish(message)`, relying on a topic fixed at the connector's own `connect(host, port, topic)` time, and `publish(message, replyTopic)`) were removed from this binding entirely, not merely left available: with no durable `destination` string to route an outbox row by, a call through either could never carry the Transactional Outbox guarantee described below, and a binding where some calls are covered and others silently aren't would be worse than one with a single, consistent family. `IQueueClient` itself still has those two methods — only the script-facing `$Queue` binding dropped them.

| Method | `message` | Purpose |
| --- | --- | --- |
| `publish(destination, message)` | plain `Map`/object | Simplest form — no `QueueMessage` to construct. `message` is serialized to JSON as the payload; an id is generated for you. |
| `publish(destination, message, replyTo)` | plain `Map`/object | Same, plus sets the `"replyTo"` metadata entry `QueueServer` reads to route a reply (§4.4 of the setup guide). |
| `publish(destination, message)` | `QueueMessage` | Use this when you need to control the message id or pre-populate metadata yourself. Whatever metadata the `QueueMessage` already carries is preserved as-is. |
| `publish(destination, message, replyTo)` | `QueueMessage` | Same, plus sets (or overrides) the `"replyTo"` entry in that `QueueMessage`'s metadata. |
| `isConnected()` | — | Whether the underlying client is connected. |

All four return a `MessageHandle`. The plain-`Map` forms are the ones to reach for by default — reserve the `QueueMessage` forms for when you need an explicit id or already have metadata to carry over.

```javascript
function execute(request, controlData) {
    // Simplest form: a plain object, no QueueMessage construction.
    var handle = $Queue.publish("payments.events", {
        type: "payment.created",
        id: controlData.id,
        tenant: $Tenant
    });

    return { id: controlData.id, outboxId: handle.getId() };
}
```

Request/reply, using the plain-`Map` form's `replyTo` parameter instead of building metadata by hand:

```javascript
$Queue.publish("orders.created", { orderId: controlData.orderId }, "orders.created.replies");
```

The `QueueMessage` form is still available when you need an explicit message id (e.g. for idempotent consumers) or already have metadata beyond `replyTo`:

```javascript
var message = new (Java.type("ma.s2m.payos.queue.QueueMessage"))(
    controlData.id, JSON.stringify({ type: "payment.created", id: controlData.id }),
    { "source": "payments-api" }, "payments.events");
$Queue.publish("payments.events", message, "payments.events.replies");
```

## Transactional Outbox: how `publish(...)` actually works

None of the four overloads above call the broker during the request. Each writes a `payos_queue_outbox` row through whichever transaction this endpoint's `atomicityScope` names (`$DB` by default, or `$PlatformDB` for `atomicityScope: "platform"` — see [architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md)), so the publish commits or rolls back together with the rest of your script's database writes. A separate background component (`QueueOutboxDispatcher`, paired with the `queue` transport server) reads that table afterward and performs the real broker publish, retrying until success.

Concretely, this is what the call means:

- **Nothing is sent to the broker if your script's transaction doesn't commit** — a script that calls `publish(...)` and then throws has published nothing, full stop.
- **The call itself doesn't block on broker round-trip latency** — it's a local database write, not a network call to NATS.
- **Delivery happens after the request, not during it.** There is no overload left on `$Queue` for a synchronous, immediate broker call inside the request — if you genuinely need that, call `IQueueClient` directly from Java (outside the script sandbox), understanding you then own the atomicity trade-off yourself.
- **A message can be delivered more than once.** A crash landing between "published to the broker" and "marked dispatched" republishes on the next cycle — your consumer must tolerate an occasional duplicate.
- **The `MessageHandle` returned echoes the outbox row's id**, not a broker-assigned message id — the real broker publish, and whatever id the broker assigns, happens later, out of band.
- **This requires `atomicityScope` to resolve to a configured database.** If the endpoint's `atomicityScope` names a database that isn't configured, every `publish(...)` overload throws `IllegalStateException` immediately rather than publishing unprotected.

`QueueBinding` (`ma.s2m.payos.scripting.QueueBinding`, `payos` repo) exposes only the members
above — no `subscribe(...)`, no `connect(...)`/`disconnect()`. See `IQueueClient.java` and
`QueueMessage.java` for the full underlying interface (relevant if you're implementing a new
`IQueueClient` connector, not for script authors).

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
$Queue.publish("payments.events", {
    type: "payment.created",
    correlationId: ctx.get("correlationId"),
    tenant: $Tenant,
    payload: controlData
});
```

See [operations/observability.md](../operations/observability.md).

## If `$Queue` is missing

`$Queue` is injected only when a queue client is registered. Configure the
[queue service](../configuration/queue-service.md) and place the `queue-service-nats`
service-adapter JAR on the [service-adapters path](../configuration/extensions-connectors.md).

## Next

- [Architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md) — the full Transactional Outbox mechanism and the dispatcher that drains it.
- [Queue setup guide (de A à Z)](queue-setup-guide.md) — configuring and using both the publisher and consumer sides end to end, with a full worked example.
- [Configuration: queue service](../configuration/queue-service.md)
- [Architecture: extensibility (queue transport & connectors)](../architecture/extensibility.md)
