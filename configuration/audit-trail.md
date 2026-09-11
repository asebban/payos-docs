Created: 2026-09-11
Last updated: 2026-09-11
Version: 1

# Audit trail configuration

The `audit-trail` block selects and configures the pluggable, durable PCI-DSS audit-trail capability: which `IAuditLogger` delivery category is active, which `IAuditTrailStore` backend it persists to, and which business-context lookup keys are approved for indexing. It is distinct from the always-on `AuditLogger.logApiExecution(...)` log line described in [operations/observability.md](../operations/observability.md) — this block controls a separate, durable evidence pipeline behind the same `IAuditLogger` call surface.

For the operational picture (defaults, metrics, known limitations, troubleshooting) see [operations/audit-trail.md](../operations/audit-trail.md). For the design rationale see the audit-trail architecture decision record referenced from that page.

## Shape

```json
{
  "audit-trail": {
    "implementation": {
      "category": "buffered-store",
      "parameters": {
        "buffer": {
          "initial-capacity": 10000,
          "max-capacity": 50000,
          "growth-percentage": 25,
          "growth-enabled": true
        }
      }
    },
    "store": {
      "backend": "filesystem",
      "filesystem": {
        "root": "/var/payos/audit-trail",
        "partitioning": "tenant-plus-day"
      },
      "integrity": {
        "mechanism": "hash-chain",
        "chain-scope": "partition-local"
      }
    },
    "business-keys": {
      "approved": ["paymentId", "merchantReference"]
    }
  }
}
```

**The entire `audit-trail` block is optional.** If it is absent from `bootstrap.json` entirely, the runtime attempts a zero-config default — see [Zero-config default](#zero-config-default) below. Every sub-block within it (`store`, `business-keys`) is also independently optional even when `audit-trail`/`implementation` is present — see each section.

## Keys

From `IConfigSpec.AuditTrail` (`payos-foundation`):

| Key | Default | Purpose |
| --- | --- | --- |
| `implementation.category` | — (required if `audit-trail` is present) | Selects the `IAuditLogger` provider by matching its `type()`. Ships with one implementation: `buffered-store` (`payos-buffered-audit-trail`). |
| `implementation.parameters` | `{}` | Opaque, category-specific block passed through as-is to the selected provider's `initialize(...)`. Shape below is `buffered-store`'s. |
| `store.backend` | — (optional) | Selects the `IAuditTrailStore` provider by matching its `type()`. Ships with one implementation: `filesystem` (`payos-audit-trail-store-filesystem`). When absent, no store is bound — see [Operating without a store](#operating-without-a-store-backend). |
| `business-keys.approved` | `[]` | Allowlist of `businessKeys` names an `AuditEvent` may index. See [Business keys allowlist](#business-keys-allowlist). |

### `buffered-store` category parameters (`implementation.parameters.buffer`)

| Key | Default | Purpose |
| --- | --- | --- |
| `initial-capacity` | `10000` | Starting size of the bounded in-process buffer. |
| `max-capacity` | value of `initial-capacity` | Upper bound the buffer may grow to. |
| `growth-percentage` | `0` | Percentage the buffer grows by (capped at `max-capacity`) when full and growth is enabled. |
| `growth-enabled` | `false` | When `false`, a full buffer goes straight to synchronous overflow persistence — see [operations/audit-trail.md](../operations/audit-trail.md#buffer-overflow-and-the-background-writer). |

### `filesystem` store parameters (`store.filesystem`)

| Key | Default | Purpose |
| --- | --- | --- |
| `root` | current working directory (`user.dir`), with a WARN log | Base directory for the `tenant=<id>/date=YYYY-MM-DD/` layout. Set this explicitly in any real deployment — the cwd fallback exists so the zero-config default (below) has somewhere to write, not as a recommended production value. |
| `partitioning` | `tenant-plus-day` | Only value supported; any other explicit value throws `IllegalArgumentException` at startup. |

### Integrity parameters (`store.integrity`)

| Key | Default | Purpose |
| --- | --- | --- |
| `mechanism` | `hash-chain` | Only value supported; any other explicit value throws at startup. |
| `chain-scope` | `partition-local` | Only value supported; any other explicit value throws at startup. Each tenant/day partition has its own independent hash chain — there is no cross-partition or cross-node sequencing. |

## Zero-config default

If the `audit-trail` block is absent from `bootstrap.json` entirely, `AuditTrailInitializer` attempts to activate `buffered-store` bound to the `filesystem` backend, both with empty parameters — every default in the tables above applies. This is deliberately best-effort: if the `payos-buffered-audit-trail`/`payos-audit-trail-store-filesystem` adapter jars aren't on `service-adapters-dir` (they are separate, optional modules — a minimal `payos-runtime` build may not include them), activation fails silently into the pre-existing `Slf4jAuditLogger` default and logs a WARN naming the reason. An *explicit* `audit-trail` block that fails to resolve (bad category/backend, ambiguous match) fails startup loudly instead — see [operations/audit-trail.md](../operations/audit-trail.md#configuration-failure-modes).

## Operating without a store backend

`store.backend` can be left unset even when `implementation.category` is configured. Without a bound store, `buffered-store` behaves as it did before durable persistence existed: the buffer accepts events but nothing drains it, and a full buffer's overflow event is logged and metered as lost (`audit.buffer.overflow.persistence.failed{reason="no-store"}`) rather than persisted. This is a legitimate configuration for a deployment that only wants the existing SLF4J-style audit log line, not the durable evidence pipeline.

## Business keys allowlist

`business-keys.approved` is a flat array of key names. `AuditEvent.businessKeys` entries not on this list are stripped (not the whole event — just that entry) before the event reaches the resolved logger, with a WARN log naming the rejected key. An absent or empty `approved` list means **every** `businessKeys` entry is rejected — deny-by-default, so a new business key is never silently indexed until a deployment operator explicitly approves it. This applies uniformly to the zero-config default path too.

`businessKeys` values must be scalar (`String`, `Integer`, `Long`, `Double`, `BigDecimal`, or `Boolean`) — this is enforced structurally by `AuditEvent.Builder.businessKey(...)` regardless of the allowlist, not a deployment-configurable rule.

## Resolution and adapter placement

Unlike most `bootstrap.json` blocks, `audit-trail` keys have **no** system property / environment variable fallback — they are read only from the `bootstrap.json` block. Both `implementation.category` and `store.backend` are matched against `IAuditLogger.type()`/`IAuditTrailStore.type()` via `ServiceLoader` using `PayOSConfig.getServiceAdapterClassLoader()` — the matching adapter jar (`payos-buffered-audit-trail`, `payos-audit-trail-store-filesystem`, or a custom implementation) must be discoverable through `service-adapters-dir` (or already bundled into the running `payos-runtime` fat jar). Zero matches or more than one match for an *explicitly* configured category/backend fails startup with an actionable error naming the missing/ambiguous type.

## Hot reload

`AuditTrailInitializer.initialize(...)` runs again on every configuration hot-reload, resolving a fresh provider instance each time. The previously-active `IAuditLogger`'s `shutdown()` hook is called right after the new one takes over, so a category that owns a background resource (`buffered-store`'s writer thread) doesn't leak across reloads.

## Next

- [operations/audit-trail.md](../operations/audit-trail.md) — how the feature behaves operationally, metrics, known limitations, troubleshooting.
- [operations/metrics-catalog-v2-2026-09-11.md](../operations/metrics-catalog-v2-2026-09-11.md) — full audit-trail metric inventory.
- [operations/observability.md](../operations/observability.md) — the always-on audit log line this capability sits behind.
