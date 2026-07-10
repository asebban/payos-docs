Created: 2026-07-10
Last updated: 2026-07-10
Version: v1

# Event Category Payload Contracts — Proposal

> Status: Proposal (not yet an ADR, not yet implemented beyond the audit trail).
> Supersedes the "one stable envelope for all event types" design principle in [`observability-event-contract-proposal.md`](observability-event-contract-proposal.md). That document's field-level detail (source/actor/correlation/transport/resource blocks, the existing-emitter mapping table, the mandatory/conditional field governance table) remains a useful reference when fleshing out the per-category contracts below — but it should not be implemented as one unified envelope.

## Governing principle

PayOS emits several fundamentally different kinds of events. Each kind gets its **own dedicated abstraction** — a typed interface plus a static facade with a swappable default implementation — instead of a single generic/shared logger that inspects a payload to decide where an event goes.

**Why:** the categories below have mutually incompatible non-functional guarantees. The audit trail must never lose an event, is immutable/WORM, and has regulator-mandated retention. Event-sourcing needs strict per-aggregate ordering and total completeness — one missing event corrupts replay. Analytics tolerates sampling and best-effort delivery. Metrics need low-cardinality dimensions or the backend falls over. Collapsing all of this behind one router that picks a sink by inspecting the payload shape hides intent at the call site (a developer reading `log(x)` can't tell which guarantees apply without reading routing logic) and forces one schema to compromise across consumers with conflicting needs.

**Reference implementation already built:** `ma.s2m.payos.security.IAuditLogger` / `AuditEvent` / `AuditLogger` (static facade, swappable via `AuditLogger.setInstance(...)`) / `Slf4jAuditLogger` (default impl), in `payos/src/main/java/ma/s2m/payos/security/`. Every new category below should follow this exact shape: `I<Category><Noun>` interface, a static facade class holding an `AtomicReference` to the active implementation, and one default implementation.

## Event categories identified

1. **Regulatory / PCI-DSS audit trail** — already implemented (`IAuditLogger`).
2. **Analytics** — business/usage tracking for BI, funnels, conversion metrics.
3. **Event-sourcing / state-reconstruction** — a durable, ordered, replayable fact stream sufficient to rebuild an entity's state after data corruption.
4. **Observability / metrics** — numeric time series for dashboards and alerting (distinct from both file logs and the audit trail).
5. **Inter-service integration / domain events** — external choreography (e.g. `PaymentCompleted` triggering webhooks or reactions in other bounded contexts); distinct from event-sourcing, which is about internal replay, not external notification.

Every category's payload includes one field that is **intentionally unrestricted in shape** so callers can attach whatever data is relevant to that specific event without waiting for a schema change — except metrics, where that freedom must be deliberately narrowed (see §4).

---

## 1. Regulatory audit trail — `AuditEvent` (existing, reference shape)

Interface: `IAuditLogger` · Facade: `AuditLogger` · Default impl: `Slf4jAuditLogger` (JSON to the `AUDIT` logger category).

| Field | Type | Notes |
|---|---|---|
| `timestamp` | Instant | auto-set to `Instant.now()` unless overridden |
| `event` | String | event type, e.g. `AUTH_SUCCESS` |
| `userId`, `tenantId`, `appId`, `correlationId`, `path` | String | standard context, `"-"` when absent |
| `result` | String | `SUCCESS` / `FAILURE` / `DENIED` / `ACTIVE` / `INFO` — drives log level in `Slf4jAuditLogger` |
| **`extra`** | **Map\<String,Object\>** | **free-form section** |

No changes proposed here — this is the template the other four follow.

---

## 2. Analytics — `AnalyticsEvent`

Interface: `IAnalyticsRecorder` (to build) · Facade: `AnalyticsRecorder` · Default impl: TBD (likely a batching/buffering SLF4J or HTTP-shipping implementation).

| Field | Type | Notes |
|---|---|---|
| `eventId` | UUID | dedup key for downstream/BI pipelines |
| `name` | String | e.g. `payment_initiated`, `checkout_started` |
| `timestamp` | Instant | |
| `tenantId` | String | |
| `correlationId` | String | ties back to the originating request/trace |
| `actorId` | String (nullable) | user/system that triggered the event; nullable for anonymous/system events |
| `source` | String | emitting app/service |
| **`properties`** | **Map\<String,Object\>** | **free-form section** — event-specific dimensions (amount, currency, plan, channel, ...) |

No strict ordering requirement; sampling and client-side buffering are acceptable. Schema can evolve freely per `name` without a version bump, since analytics consumers are expected to tolerate unknown/missing properties.

```json
{
  "eventId": "b6b1...",
  "name": "payment_initiated",
  "timestamp": "2026-07-10T09:12:04Z",
  "tenantId": "acme",
  "correlationId": "corr-123",
  "actorId": "john.doe",
  "source": "payos-runtime",
  "properties": { "amount": 1500, "currency": "MAD", "channel": "card" }
}
```

---

## 3. Event-sourcing / state-reconstruction — `StateEvent`

Interface: `IEventStore` (to build; append + replay, not just "log") · Facade: `EventStore` · Default impl: TBD (must be durable — this is the one category where a plain SLF4J default is not acceptable as a real backing store, only as a placeholder/no-op during early development).

| Field | Type | Notes |
|---|---|---|
| `eventId` | UUID | globally unique |
| `streamId` / `aggregateId` | String | which entity, e.g. `payment-8391` |
| `aggregateType` | String | e.g. `Payment`, `ConnectorLifecycle` |
| `sequenceNumber` | long | **strictly increasing per `streamId`** — drives ordering and optimistic concurrency |
| `eventType` | String | the domain fact, e.g. `PaymentAuthorized` |
| `occurredAt` | Instant | when the fact actually happened |
| `recordedAt` | Instant | when persisted — may differ from `occurredAt` on replay/backfill |
| `schemaVersion` | int | **mandatory** — this stream may be replayed years later |
| `tenantId`, `correlationId` | String | |
| **`data`** | **Map\<String,Object\>** | **free-form section — but see the rule below** |

**Rule, not a suggestion:** unlike the other categories, `data` is not "extra context" — it is the payload the entire reconstruction depends on. Once `schemaVersion = N` has been emitted for an `eventType`, its shape is frozen. A new field requires bumping `schemaVersion`, never a silent, in-place change to what an existing key means — otherwise replay silently reconstructs the wrong state with no visible error.

```json
{
  "eventId": "e2f4...",
  "streamId": "payment-8391",
  "aggregateType": "Payment",
  "sequenceNumber": 3,
  "eventType": "PaymentAuthorized",
  "occurredAt": "2026-07-10T09:12:04.500Z",
  "recordedAt": "2026-07-10T09:12:04.512Z",
  "schemaVersion": 1,
  "tenantId": "acme",
  "correlationId": "corr-123",
  "data": { "amount": 1500, "currency": "MAD", "authCode": "00", "providerRef": "auth_9f2b" }
}
```

---

## 4. Observability / metrics — `MetricSample`

Interface: `IMetricsRecorder` (to build) · Facade: `MetricsRecorder` · Default impl: TBD (Micrometer/OpenTelemetry-backed, or a no-op until wired).

| Field | Type | Notes |
|---|---|---|
| `name` | String | dot-namespaced, e.g. `connector.execution.duration` |
| `timestamp` | Instant | |
| `type` | enum | `counter` \| `gauge` \| `histogram` |
| `value` | double | |
| `unit` | String | `ms`, `count`, `bytes`, ... |
| `tenantId` | String (optional) | often aggregated away for cardinality reasons |
| `correlationId` | String (optional) | exemplar-only, not a queryable dimension |
| **`tags`** | **Map\<String,String\>** | **free-form section — deliberately constrained** |

**This is the one category where "free-form" must be bridled.** `tags` must stay low-cardinality, string-only key/value pairs (e.g. `connectorType=CardNetwork`, `outcome=SUCCESS`) — never a raw ID, a timestamp, or free text. A metrics backend (Prometheus and similar) degrades or falls over under high-cardinality or unbounded-value tags; that's an operational constraint this category has that none of the other four do, and it should be enforced at the call site, not discovered in production.

```json
{
  "name": "connector.execution.duration",
  "timestamp": "2026-07-10T09:12:04Z",
  "type": "histogram",
  "value": 42.0,
  "unit": "ms",
  "tags": { "connectorType": "CardNetwork", "outcome": "SUCCESS" }
}
```

---

## 5. Inter-service integration / domain events — `IntegrationEvent`

Interface: `IIntegrationEventPublisher` (to build) · Facade: `IntegrationEventPublisher` · Default impl: TBD (likely NATS/queue-backed, given PayOS's existing `IQueueClient` abstraction). Modeled on [CloudEvents](https://cloudevents.io/) (CNCF standard) for interoperability with external consumers.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | unique |
| `type` | String | versioned, reverse-DNS style, e.g. `ma.s2m.payos.payment.completed.v1` |
| `source` | String | producer identity |
| `time` | Instant | |
| `tenantId`, `correlationId` | String | |
| `schemaVersion` | int (or embedded in `type`) | |
| **`data`** | **Map\<String,Object\>** | **free-form section** — payload delivered to consumers |

Ordering is less critical than for event-sourcing — at-least-once delivery plus an idempotency key at the consumer is generally sufficient — but schema-versioning discipline matters just as much, since external consumers (webhooks, other services) cannot always be redeployed in lockstep with the producer.

```json
{
  "id": "f1a2...",
  "type": "ma.s2m.payos.payment.completed.v1",
  "source": "payos-runtime",
  "time": "2026-07-10T09:12:05Z",
  "tenantId": "acme",
  "correlationId": "corr-123",
  "schemaVersion": 1,
  "data": { "paymentId": "pay_8391", "amount": 1500, "currency": "MAD" }
}
```

---

## Shared conventions across all five

- Minimal common envelope: `timestamp`, `tenantId`, `correlationId` appear in every category — by convention, not by forced inheritance. Each interface/facade stays independent so retention, delivery guarantees, and sink choice can evolve per category without touching the others.
- Every category's free-form section stores only Jackson-serializable values (String, numeric, Boolean, nested Map/List) — matching `AuditEvent.Builder.field(...)`'s existing constraint.
- Naming convention for future categories: `I<Category><Noun>` interface (e.g. `IAnalyticsRecorder`), `<Category><Noun>` static facade (e.g. `AnalyticsRecorder`), `<Impl><Category><Noun>` default implementation (e.g. `Slf4jAnalyticsRecorder` or similar) — mirroring `IAuditLogger`/`AuditLogger`/`Slf4jAuditLogger` exactly.

## Open questions for whoever picks this up

- Default (non-audit) implementations are all "TBD" above — likely SLF4J-based placeholders to start, consistent with how `IAuditLogger` began, swapped for real sinks (analytics pipeline, durable event store, metrics backend, message queue) as each category is actually wired into a consumer.
- `IEventStore`'s default implementation is the one that probably can't ship as a no-op/log-only placeholder for very long, since its entire purpose is durable replay — worth flagging explicitly when scoping the first story that builds it.
