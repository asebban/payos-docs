Created: 2026-10-16
Last updated: 2026-10-17
Version: 3 — supersedes v2: `$Queue.publish(...)` gained a plain-`Map` form and an optional trailing `replyTo`, for four overloads total — see [§4](#4-queue--the-transactional-outbox-pattern).

# Atomicity across `$DB`, `$Audit`, and `$Queue`

How PayOS guarantees that a script's database writes, its audit trail entries, and its queue publications succeed or fail together — without a distributed transaction (2PC/XA). This document explains the mechanism; the configuration keys it introduces are referenced from [configuration/database-service.md](../configuration/database-service.md), [configuration/platform-database-service.md](../configuration/platform-database-service.md), [configuration/audit-trail.md](../configuration/audit-trail.md), and [configuration/servers.md](../configuration/servers.md); the script-facing behavior is covered in [developer/data-access.md](../developer/data-access.md), [developer/platform-database-usage.md](../developer/platform-database-usage.md), [developer/queue-messaging.md](../developer/queue-messaging.md), and [developer/scripting-bindings.md](../developer/scripting-bindings.md).

## 1. The problem

A script that writes to `$DB`, calls `$Audit.logEvent(...)`, and calls `$Queue.publish(...)` makes three separate claims about the same business event. Before this change, only the first one was transactional — a failed audit write or a failed queue publish never rolled back the database write, and a database rollback never un-published a message already sent to the broker. Two requirements drove the fix:

- **`$Audit` must be strict.** If the audit row cannot be written, the whole request must fail — no exceptions, no silent loss.
- **`$Queue` must guarantee "nothing without commit, delivery eventually after."** No message is ever published if the database transaction didn't commit; every message whose transaction did commit is delivered, eventually, with retry until success.

A full two-phase-commit/XA transaction across the database and the broker was considered and rejected: it needs an external transaction coordinator, NATS and the filesystem don't implement XA natively, a coordinator failure between phases risks blocking, and it adds per-request latency for a guarantee this system doesn't actually need.

## 2. The existing load-bearing mechanism: the `$DB` request transaction

A script already runs inside a real database transaction, opened before the script (`databaseService.beginRequestScope()`) and committed or rolled back automatically afterward (`commitRequestScope()`/`rollbackRequestScope()`), depending on whether the script succeeded or threw. Writing `$Audit` and `$Queue` into that **same Hibernate session** makes their writes join that same transaction automatically — no new coordination machinery required.

`$PlatformDB` opens exactly the same way, in parallel, as a **second, independent transaction**. `$DB` and `$PlatformDB` are never one shared transaction.

### 2.1 Which transaction carries the guarantee — `atomicityScope`

Since `$DB` and `$PlatformDB` are independent, `$Audit`/`$Queue` must write into exactly one of them per request — whichever one the script actually uses. Both are resolved unconditionally for every request regardless of what the script does, so "is `$DB` configured" is not the same question as "does this endpoint use `$DB`." Deducing the right answer at runtime (e.g. by checking whether the `$DB` session has been touched yet) is equally unreliable — a script may call `$Audit.logEvent(...)` before its first write.

The chosen solution is a **declarative, per-endpoint field, resolved once** when `mappings.json` loads — exactly like `roles` already is:

```json
{
  "mappings": {
    "api": {
      "/reference-data/refresh": {
        "POST": { "handler": "scripts/refresh-reference-data.js", "atomicityScope": "platform" }
      },
      "/orders": {
        "POST": { "handler": "scripts/create-order.js", "atomicityScope": "tenant" }
      }
    }
  }
}
```

- **`"tenant"`** (default — the field may be omitted) → `$Audit`/`$Queue` write into `$DB`'s transaction.
- **`"platform"`** → `$Audit`/`$Queue` write into `$PlatformDB`'s transaction instead.

If an endpoint's `atomicityScope` names a database that isn't configured (`"tenant"` with no `$DB` for the resolved tenant, or `"platform"` with no `platform-database-service`), `$Audit.logEvent(...)`/`$Queue.publish(destination, message)` reject the call with a clear `IllegalStateException` the moment a script actually calls them — never a silent no-op.

### 2.2 Locking out the database `atomicityScope` doesn't cover

Declaring `atomicityScope` isn't enough by itself: nothing would otherwise stop a script from also writing to the *other* database, silently escaping the guarantee. The lockout is asymmetric, by design:

- **`atomicityScope` ≠ `"platform"` (the default) → `$PlatformDB` becomes read-only.** Scripts get a narrower binding exposing only `newParams`/`get`/`getFresh`/`findAll`/`find`/`list`/`unique`/`first` — no `save`/`update`/`delete`/`deleteById`/`executeUpdate`. Reading reference data this way stays useful and safe, so it isn't removed.
- **`atomicityScope` = `"platform"` → `$DB` isn't bound at all.** Not a read-only version — `typeof $DB === 'undefined'`, the same treatment `$DB` already gets when no database service is configured at all. An endpoint declared `"platform"` has, by construction, no declared reason to touch the tenant database, not even to read.

**This is a breaking change for two kinds of existing endpoints.** Any endpoint that wrote to `$PlatformDB` without declaring `atomicityScope: "platform"` loses that write capability (`$PlatformDB.save(...)` now fails with `Unknown identifier: save`). Any endpoint explicitly declared `atomicityScope: "platform"` that also used `$DB` loses `$DB` entirely (`$DB is not defined`). Before enabling this in a deployment with existing endpoints, audit which endpoints write to `$PlatformDB` today and give them the right `atomicityScope` value — see [developer/platform-database-usage.md](../developer/platform-database-usage.md) for the full before/after picture.

## 3. `$Audit` — a direct write, in the same transaction

`AuditBinding.logEvent(...)` builds a `payos_audit_event` row and writes it with `IDatabaseService.save("PayosAuditEvent", row)` through whichever session `atomicityScope` selected — the exact same generic entity-name mechanism any `$DB.save(...)` call already uses. There is no dispatcher, no buffer, and no "eventual" guarantee here: it's an ordinary ACID write. If anything later in the script fails, the rollback undoes the audit row along with everything else; if the script succeeds, both commit together.

`$Audit` still exposes only the free-form `logEvent(...)` family — never `IAuditLogger`'s typed system-lifecycle methods (`logAuthSuccess`, `logApiExecution`, ...). Those continue to flow through a **separate, pre-existing path**: the kernel's own system-event logging (`AuditLogger`/the configured `IAuditLogger`), asynchronous and tolerant of loss, never atomic with a script's transaction — see §5 below for how that path itself changed independently.

**The kernel owns no mapping for `payos_audit_event`.** `"PayosAuditEvent"` and this row's field names (`eventId`, `schemaVersion`, `eventType`, `tenantId`, `correlationId`, `appId`, `path`, `userId`, `result`, `operation`, `resourceType`, `resourceId`, `businessKeysJson`, `fieldsJson`, `createdAt`, `exportedAt`) are a documented, stable contract with whichever plugin module owns the actual table — today, `payos-database-audit-trail` (§5). The kernel never compiles against that module's Hibernate mapping or any class of it.

Unlike `$Analytics`/`$Metrics`/`$Integration`/`$Diagnostics`, `$Audit` does **not** swallow errors: a failed insert, or a missing `atomicityScope` session, propagates straight out of `logEvent(...)`, through the script, into the kernel's existing exception handling, which rolls back the same transaction this write just joined.

## 4. `$Queue` — the Transactional Outbox pattern

`QueueBinding.publish(...)` no longer calls `IQueueClient.publish(...)` directly. Every overload writes a row into `payos_queue_outbox` — `id`, `tenantId`, `destination`, `payload`, `metadata` (JSON), `status` (`PENDING`/`DISPATCHED`), `attempts`, `createdAt`, `lastAttemptedAt`, `lastError` — through the same `atomicityScope`-selected session, joining the same transaction `$Audit` writes into. A separate background component reads that table afterward and performs the real broker publish, with retry until success.

`$Queue` exposes four `publish` overloads, all destination-based: `publish(destination, message)` and `publish(destination, message, replyTo)`, each available with `message` as either a plain `Map<String, Object>` (serialized to JSON as the payload, id generated) or an already-built `QueueMessage` (its own id/metadata used as-is). `replyTo`, when given, sets the same `"replyTo"` metadata entry `QueueServer` already reads to route a reply — merged into the `QueueMessage` form's existing metadata, or into an otherwise-empty metadata map for the plain-`Map` form. All four share one `IllegalStateException` guard: if `atomicityScope` names a database that isn't configured, every overload rejects the call immediately rather than publishing unprotected.

**The two legacy, implicit-default-topic overloads were removed from `$Queue` entirely**, rather than left reachable and unprotected. `publish(message)`/`publish(message, replyTopic)` relied on a topic fixed at the broker client's own connect-time (`connect(host, port, topic)`), not a durable `destination` string — with nothing to route an outbox row by, a call through either could never carry this guarantee. Since every endpoint's `$Queue` needs to mean the same thing — "covered by the atomicity guarantee" — rather than ship a binding where some calls silently bypass it, `$Queue` exposes only the destination-based overloads above. `IQueueClient` itself still has the two legacy methods (other code paths, like the kernel's own system-event publishing, may still use them) — only the script-facing `$Queue` binding dropped them.

### 4.1 The dispatcher

`payos-server-queue` owns `payos_queue_outbox` entirely — same principle as `payos-database-audit-trail` owns `payos_audit_event`. `QueueOutboxDispatcher` is a plain background-thread worker (daemon, same shape as `payos-database-audit-trail`'s own `Writer`), paired with `QueueServer` rather than being a `servers[]` entry of its own: `QueueServer.start()` starts it right after its own broker connection and subscription succeed, reusing that same already-connected `IQueueClient` instead of opening a second broker connection; `QueueServer.stop()` stops it. Configure it via two optional keys on the same `protocol: "queue"` entry in `servers[]` — see [configuration/servers.md](../configuration/servers.md#queue-keys-queue).

Each cycle, for every known tenant and the platform database, the dispatcher reads `PENDING` rows (oldest first), calls `IQueueClient.publish(destination, message)` for each, and marks `DISPATCHED` on success or increments `attempts`/records `lastError` on failure. **No message is ever abandoned.** `maxAttempts` is a logging threshold only — past it, a failure is logged at `ERROR` instead of `WARN` so an operator notices a stuck message, but the row stays `PENDING` and keeps being retried indefinitely. There is no `FAILED` status and no dead-letter queue.

The dispatcher opens one dedicated session per tenant (`IDatabaseService.openDedicatedSessionFor(tenantId)`) and **reuses it across cycles** rather than reopening it every poll — the same connection-pool-friendly pattern `$Parallel` (`payos-job-scheduler`) already uses. A session that throws is closed and transparently reopened on the next cycle; that tenant's failure never stops the dispatcher or affects any other tenant (fail-open). Platform-scoped rows go through `PayOSConfig.getPlatformDatabaseService()` directly — already an appropriately-scoped singleton.

Metrics: `queue.outbox.pending` and `queue.outbox.exceeded_max_attempts` (both gauges, tagged by `tenantId`) — enough for an operator to see how many messages are waiting and how many have passed the logging threshold, without needing a direct SQL query.

### 4.2 Guarantee, stated precisely

- **Nothing is ever published if the `$DB`/`$PlatformDB` transaction didn't commit** — guaranteed by construction, same mechanism as `$Audit`.
- **Everything that committed is published, eventually, at least once** — not instantly, and a crash landing exactly between "published to the broker" and "marked `DISPATCHED`" can republish a message a second time. The consumer must tolerate an occasional duplicate (idempotence) — the standard, accepted trade-off of this pattern, in exchange for not building a real 2PC/XA transaction.

## 5. Startup: who owns table creation, and what happens when it fails

Neither table is referenced by the kernel at all — not a mapping class, not a `.hbm.xml`, not a creation call. Each owning module calls the shared `IDatabaseService.ensureSchemaFor(String... hibernateMappingFiles)` capability (`database-service`) for its own table, at its own startup, independent of whatever `ddl-auto` each tenant has configured for its own entities — an operator setting `ddl-auto: validate` in production to keep Hibernate from touching their own schema shouldn't also block a plugin's own infrastructure table. `ensureSchemaFor` is idempotent: a table that already exists under `shared-schema` isolation (where multiple tenants point at the same physical table) is never recreated or altered a second time.

**The failure treatment is asymmetric, by explicit choice:**

- **Audit: fatal.** `payos-database-audit-trail` throws `FatalException` (`payos-foundation`) if it cannot guarantee `payos_audit_event` exists — this propagates to the kernel's already-existing `catch (Exception e) { ...; System.exit(1); }` around audit-trail initialization, stopping PayOS from starting at all. No kernel code change was needed for this; the module only had to actually throw.
- **Queue: non-fatal.** `QueueOutboxDispatcher.start()` has no notion of "server" at all — a schema-creation failure there is a plain `IllegalStateException`. It is `QueueServer.start()` (the actual `IServer`) that already catches any exception during its own startup and wraps it as `UnableToStartServerException` — the same treatment any other `servers[]` entry failure already gets, with nothing dispatcher-specific added to the kernel. PayOS keeps booting without that queue server (and its paired dispatcher). The practical consequence: `$Queue.publish(destination, message)` stays available to scripts and keeps writing outbox rows, but nothing drains them until that server starts successfully.

## 6. What didn't change

- `$DB` itself — its own transaction mechanics are unchanged; it's the one carrying the whole guarantee for `atomicityScope: "tenant"`.
- The point-to-point request/response mechanism already in `payos-server-queue` (`QueueServer`, `mappings.json`, `replyTo`) — this document only changes how `$Queue.publish(destination, message)` behaves on the **publishing** side, never the receiving side.
- `IQueueClient`/`NatsQueueClient` — the dispatcher calls them exactly as `QueueBinding` used to, directly.

## 7. Where each piece lives

| Piece | Module | Notes |
| --- | --- | --- |
| `atomicityScope` field, `$DB`/`$PlatformDB` lockdown | `payos` (kernel) | `ApiResource`/`ApiLoader`, `ReadOnlyDatabaseBinding`, `ApiResourceHandler` |
| `AuditBinding` direct write | `payos` (kernel) | No reference to any audit mapping — generic entity-name write only |
| `QueueBinding` outbox write | `payos` (kernel) | No reference to any outbox mapping — generic entity-name write only; four destination-based overloads (plain `Map`/`QueueMessage` × with/without `replyTo`), legacy overloads removed from the binding entirely |
| `IDatabaseService.ensureSchemaFor(...)` | `payos-foundation` / `database-service` | Shared capability, used only by the two modules below |
| `FatalException` | `payos-foundation` | Audit-only |
| `payos_audit_event` mapping, table creation, system-event writer/exporter | `payos-database-audit-trail` | The `database` category of `IAuditLogger` — a new, coexisting category, not a replacement for `payos-buffered-audit-trail` |
| `payos_queue_outbox` mapping, table creation, `QueueOutboxDispatcher` | `payos-server-queue` | Paired with `QueueServer`, not a separate `servers[]` entry |

## Next

- [Developer: data access (`$DB`)](../developer/data-access.md)
- [Developer: platform database usage (`$PlatformDB`)](../developer/platform-database-usage.md)
- [Developer: queue messaging (`$Queue`)](../developer/queue-messaging.md)
- [Configuration: audit trail](../configuration/audit-trail.md) — the `database` category's parameters.
- [Configuration: servers](../configuration/servers.md) — the dispatcher's `maxAttempts`/`pollIntervalSeconds` keys.
