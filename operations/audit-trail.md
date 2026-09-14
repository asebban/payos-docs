Created: 2026-09-11
Last updated: 2026-09-14
Version: 3

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

The default `filesystem` store persists one JSON line per record to `root/tenant=<id>/date=YYYY-MM-DD/records.jsonl`, deduplicated by `eventId`, with a parallel per-partition SHA-256 hash chain (`integrity/chain.jsonl`) for tamper evidence. It also maintains three query-acceleration indexes at append time — see [Query acceleration (indexes)](#query-acceleration-indexes) below.

## Query acceleration (indexes)

The `filesystem` store writes on-disk indexes alongside every append, so `find(...)` (see [Operator query](#operator-query)) can avoid a full partition scan when a query narrows by `eventId`, `correlationId`, or `businessKeys`:

- **`index/event-id.idx`** — one `"<eventId>\t<byteOffset>"` line per record, where `byteOffset` is the exact position of that record's line in `records.jsonl`. A query filtered by `eventId` scans only this small, JSON-free file to find the offset, then seeks directly into `records.jsonl` for that one line — no need to parse every preceding record.
- **`index/fields/correlationId.idx`** — one `"<byteOffset>\t<jsonValue>"` line per record. The only fixed-field index built so far: `userId`/`operation`/`resourceType`/`resourceId` were considered (same shape, same mechanism) but deliberately deferred — each additional indexed field is one more small file write per `append()`, and that write-amplification cost wasn't judged worth it beyond `correlationId`, the dimension operators actually search by most (tracing one request end to end).
- **`index/business-keys/key=<name>.idx`** — one `"<byteOffset>\t<jsonValue>"` line per record carrying that `businessKeys` entry, for every key actually present on an appended event (whatever survived `BusinessKeysPolicy`'s allowlist).

A query can combine `correlationId` and one or more `businessKeys` filters — `find(...)` looks up each one's index and intersects the resulting offset sets (every specified criterion must match, same AND semantics as the in-memory filter), seeking directly to each surviving candidate instead of scanning every record in the partition. `eventId`, being maximally selective on its own, always takes precedence when set.

Index writes are **best-effort and non-blocking to the primary write**: if writing a `correlationId` or `business-keys` index entry fails (a key name unsafe for a filename, a transient I/O error), that one entry is logged and skipped — the audit record itself, already durably appended before indexing runs, is never lost or delayed because of it. The `event-id.idx` write is not best-effort (it shares the same write step as the record and the integrity chain, unchanged from before this capability existed), since crash-recovery dedup already depended on it being reliable.

**Known limitation, self-healing:** every index here is built going forward only — a partition is never retroactively backfilled for records already on disk. A tenant/day partition that straddles the exact moment a given index is deployed (some records written by an older version, some by this one) can miss a pre-upgrade record on an index-accelerated query for that one partition; `find(...)` detects a pre-upgrade `event-id.idx` line (missing the offset column) and falls back to a full scan for the whole partition rather than risk a false "not found," but there is no equivalent detection for the `fields`/`business-keys` indexes, since those files did not exist at all before each capability shipped. This is bounded to at most one day per deployment (the next day's partition starts clean) and is accepted the same way this module already accepts "no retry on persistence failure" and "no cross-node write coordination" as documented MVP trade-offs rather than building a full retroactive index-rebuild mechanism.

## Defaults out of the box

An unconfigured deployment (`audit-trail` absent from `bootstrap.json`) still gets a working, durable audit trail: `buffered-store` bound to `filesystem`, root defaulting to the process's current working directory (with a WARN naming it — **set `store.filesystem.root` explicitly for any real deployment**, the cwd fallback is a zero-config convenience, not a recommended production path). See [configuration/audit-trail.md#zero-config-default](../configuration/audit-trail.md#zero-config-default) for exactly when this activates versus gracefully degrades to the plain SLF4J audit log line.

## Buffer overflow and the background writer

A single background thread continuously drains the buffer to the store, off the request path. When the buffer is full (optionally after one bounded growth attempt), the *calling* thread persists that one event to the store synchronously instead — this is the only point where an audit-trail operation can add latency to a request, and only under sustained buffer pressure. Both the writer and the overflow path follow `continue-and-signal`: a persistence failure is logged and metered, **never retried**, and never blocks or fails the audited business operation. See [Known limitations](#known-limitations) for what that means for audit completeness during a store outage.

## Integrity verification

`IAuditTrailStore.verify(VerificationScope)` recomputes and checks the hash chain for one tenant/day partition, detecting record modification, removal, and reordering. There is no scheduled/automatic verification job — an operator (or an operational script) calls `verify(...)` on demand, typically via `AuditTrailQueries`'s companion `IAuditTrailStore` reference or a dedicated tool built against it. A broken result is also logged at WARN by the store itself (naming the scope and the exact position/`eventId` where the check failed), independent of whatever the caller does with the returned `VerificationResult` — so a broken chain is never silently missed even if the caller only checks the boolean.

## Operator query

`ma.s2m.payos.security.AuditTrailQueries.find(AuditTrailQuery)` (`payos` kernel) queries the currently-active store by tenant, a required time range, and optional exact-match filters (`correlationId`, `userId`, `eventType`, `result`, `eventId`, `businessKeys`). There is no tenant-facing or HTTP-exposed query API — this is a Java-level capability for internal operator tooling to call directly; building an actual internal query UI/CLI against it is a separate concern. `AuditTrailQuery.of(tenantId, from, to, limit)` requires an explicit `limit` (no hidden default) and returns `AuditTrailQueryResult.truncated() == true` if more matches existed than `limit` allowed. `eventId`, `correlationId`, and `businessKeys` are the filters the `filesystem` store can accelerate via its on-disk indexes (see [Query acceleration (indexes)](#query-acceleration-indexes)) — the other filters always require reading each index-selected (or, absent an index, every) candidate record to check.

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
- **Indexes are not retroactively backfilled.** See [Query acceleration (indexes)](#query-acceleration-indexes)'s own "Known limitation" paragraph — a partition written across the exact moment this capability is deployed can silently skip a pre-upgrade record on an index-accelerated query for that one day only.
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
