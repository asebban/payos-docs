Created: 2026-09-11
Last updated: 2026-09-11
Version: 1

# Audit trail

Operational guide for the pluggable, durable PCI-DSS audit-trail capability: buffered delivery, filesystem persistence, integrity verification, operator query, and the metrics/failure signals that make degraded audit persistence visible instead of silent. For the configuration reference see [configuration/audit-trail.md](../configuration/audit-trail.md). This capability sits behind the same `IAuditLogger` call surface described in [observability.md](observability.md) — no audit-producing call site changes when you enable it.

## How it works

```text
Caller -> AuditLogger.logEvent(event)
            -> BusinessKeysPolicy.sanitize(...)        [strips unapproved businessKeys entries]
            -> resolved IAuditLogger.logEvent(event)   [buffered-store by default]
                 -> buffer.offer(event)                [non-blocking, off the request path]
                      accepted  -> background Writer thread drains it -> IAuditTrailStore.append(...)
                      rejected -> optional bounded growth, retry offer
                                  still rejected -> synchronous IAuditTrailStore.append(...)
                                                     on the calling thread (never blocks/fails
                                                     the audited business operation either way)
```

The default `filesystem` store persists one JSON line per record to `root/tenant=<id>/date=YYYY-MM-DD/records.jsonl`, deduplicated by `eventId`, with a parallel per-partition SHA-256 hash chain (`integrity/chain.jsonl`) for tamper evidence.

## Defaults out of the box

An unconfigured deployment (`audit-trail` absent from `bootstrap.json`) still gets a working, durable audit trail: `buffered-store` bound to `filesystem`, root defaulting to the process's current working directory (with a WARN naming it — **set `store.filesystem.root` explicitly for any real deployment**, the cwd fallback is a zero-config convenience, not a recommended production path). See [configuration/audit-trail.md#zero-config-default](../configuration/audit-trail.md#zero-config-default) for exactly when this activates versus gracefully degrades to the plain SLF4J audit log line.

## Buffer overflow and the background writer

A single background thread continuously drains the buffer to the store, off the request path. When the buffer is full (optionally after one bounded growth attempt), the *calling* thread persists that one event to the store synchronously instead — this is the only point where an audit-trail operation can add latency to a request, and only under sustained buffer pressure. Both the writer and the overflow path follow `continue-and-signal`: a persistence failure is logged and metered, **never retried**, and never blocks or fails the audited business operation. See [Known limitations](#known-limitations) for what that means for audit completeness during a store outage.

## Integrity verification

`IAuditTrailStore.verify(VerificationScope)` recomputes and checks the hash chain for one tenant/day partition, detecting record modification, removal, and reordering. There is no scheduled/automatic verification job — an operator (or an operational script) calls `verify(...)` on demand, typically via `AuditTrailQueries`'s companion `IAuditTrailStore` reference or a dedicated tool built against it. A broken result is also logged at WARN by the store itself (naming the scope and the exact position/`eventId` where the check failed), independent of whatever the caller does with the returned `VerificationResult` — so a broken chain is never silently missed even if the caller only checks the boolean.

## Operator query

`ma.s2m.payos.security.AuditTrailQueries.find(AuditTrailQuery)` (`payos` kernel) queries the currently-active store by tenant, a required time range, and optional exact-match filters (`correlationId`, `userId`, `eventType`, `result`, `businessKeys`). There is no tenant-facing or HTTP-exposed query API — this is a Java-level capability for internal operator tooling to call directly; building an actual internal query UI/CLI against it is a separate concern. `AuditTrailQuery.of(tenantId, from, to, limit)` requires an explicit `limit` (no hidden default) and returns `AuditTrailQueryResult.truncated() == true` if more matches existed than `limit` allowed.

## Metrics

See [metrics-catalog-v2-2026-09-11.md](metrics-catalog-v2-2026-09-11.md) for the full inventory (buffer occupancy/growth/overflow, store append/verify/query duration and failures). At minimum, alert on:

- `payos.audit.buffer.overflow.persistence.failed` (any increment) — an audit event was accepted and then lost (no store bound, or the synchronous overflow write itself failed).
- `payos.audit.buffer.writer.append.failed` (any increment) — a *buffered* (non-overflow) event was accepted, drained by the writer, and then lost when persisting it failed. The caller received a normal, successful return for this event with no way to know it was later lost — this is the single most severe failure category in the pipeline.
- `payos.audit.store.verify.result{outcome="broken"}` (any increment) — tamper/corruption detected in a partition someone explicitly verified.

## Known limitations

- **No retry on persistence failure.** Both the background writer and the synchronous overflow path log + meter a failure and move on — they do not retry, backoff, or re-queue. During a sustained store outage, every buffered and overflow event fails to persist until the store recovers; this is a deliberate MVP tradeoff (the architecture's own sequence diagram has no retry branch), not an oversight. Monitor the metrics above to catch this promptly.
- **`AuditLogger.getInstance().logEvent(...)` bypasses the business-keys allowlist.** `AuditLogger.logEvent(AuditEvent)` (the static facade) sanitizes `businessKeys` before delegating; calling a method directly on `AuditLogger.getInstance()`'s returned `IAuditLogger` skips that step. No known production call site does this today (verified by a full-codebase search when this was found), but nothing structurally prevents it. Documented on `AuditLogger.getInstance()`'s own Javadoc.
- **`queue-backed` is not implemented.** The extension point is proven to work (an independent, non-buffering category can be resolved and share the store contract without any kernel changes), but no real broker-backed implementation exists yet. Build one only when a deployment genuinely needs broker-mediated audit delivery.
- **No cross-node write coordination.** The filesystem store's hash chain is `partition-local` and assumes a single writer per tenant/day partition. Multiple PayOS nodes writing to the *same* shared filesystem root concurrently is out of scope — an OS-level file lock or a different backend would be needed for that topology.
- **The `event-id.idx` dedup index is not a real index.** It stores one `eventId` per line for crash-recovery deduplication only — no byte offset, no acceleration for "retrieve record by eventId." A future query story that wants fast point lookups will need to redesign this file's format.
- **Sensitive-field redaction is a fixed, non-configurable denylist.** `AuditEvent`'s `extra` field auto-redacts values under sensitive-looking key names (`SensitiveFieldMasker`, shared with connector payload/config masking) — deployment-specific sensitive terms cannot be added without a code change. `businessKeys` has no denylist at all; it relies entirely on the allowlist plus its scalar-only structural constraint.

## Configuration failure modes

| Situation | Behavior |
| --- | --- |
| `audit-trail` absent entirely | Zero-config default attempt; failure degrades silently to `Slf4jAuditLogger` (WARN logged). |
| `audit-trail.implementation.category` present but blank | Startup fails (`UnableToStartServerException`). |
| Configured category/backend matches zero providers | Startup fails, naming the type and `service-adapters-dir`. |
| Configured category/backend matches more than one provider | Startup fails, naming every colliding class — no silent first-match selection. |
| `store.filesystem.partitioning`/`store.integrity.mechanism`/`store.integrity.chain-scope` set to anything but the one supported value | Startup fails (`IllegalArgumentException`) when the store initializes. |

## Next

- [configuration/audit-trail.md](../configuration/audit-trail.md) — every key, default, and the config shape.
- [metrics-catalog-v2-2026-09-11.md](metrics-catalog-v2-2026-09-11.md) — full metric inventory.
- [observability.md](observability.md) — the always-on audit log line and diagnostics this capability sits alongside.
