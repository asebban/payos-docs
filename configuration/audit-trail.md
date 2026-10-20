Created: 2026-09-11
Last updated: 2026-10-19
Version: 8 — supersedes v7: `$Audit.logEvent(...)` no longer refuses the call when the active category can't offer atomicity — it always succeeds, atomic only when `category: "database"` is active (the only one that reads `AtomicWriteContext`), best-effort otherwise, exactly like a system event. `category: "database"` is now a recommendation for the atomicity guarantee, not a requirement to use `$Audit` at all. See [`database` category parameters](#database-category-parameters-implementationparameters).

# Audit trail configuration

The `audit-trail` block selects and configures the pluggable, durable PCI-DSS audit-trail capability: which `IAuditLogger` delivery category is active, which `IAuditTrailStore` backend it persists to, and which business-context lookup keys are approved for indexing. It is distinct from the always-on `AuditLogger.logApiExecution(...)` log line described in [operations/observability.md](../operations/observability.md) — this block controls a separate, durable evidence pipeline behind the same `IAuditLogger` call surface. It also governs the kernel's own system-event logging path (`logAuthSuccess`, `logApiExecution`, ...), and — only when `category: "database"` is active — whether `$Audit`'s own writes additionally become atomic with a script's transaction; see [architecture: atomicity across `$DB`, `$Audit`, and `$Queue`](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#3-audit--always-works-atomicity-is-a-bonus-only-some-implementations-provide) for how the two relate.

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
| `implementation.category` | — (required if `audit-trail` is present) | Selects the `IAuditLogger` provider by matching its `type()`. Ships with two coexisting implementations: `buffered-store` (`payos-buffered-audit-trail`) and `database` (`payos-database-audit-trail`) — see below. Neither replaces the other; pick the one matching your deployment's durability/latency trade-off. |
| `implementation.parameters` | `{}` | Opaque, category-specific block passed through as-is to the selected provider's `initialize(...)`. Shape depends on the chosen category — both are documented below. |
| `store.backend` | — (optional) | Selects the `IAuditTrailStore` provider by matching its `type()`. Ships with one implementation: `filesystem` (`payos-audit-trail-store-filesystem`). When absent, no store is bound — see [Operating without a store](#operating-without-a-store-backend). |
| `business-keys.approved` | `[]` | Allowlist of `businessKeys` names an `AuditEvent` may index. See [Business keys allowlist](#business-keys-allowlist). |

### `buffered-store` category parameters (`implementation.parameters.buffer`)

| Key | Default | Purpose |
| --- | --- | --- |
| `initial-capacity` | `10000` | Starting size of the bounded in-process buffer. |
| `max-capacity` | value of `initial-capacity` | Upper bound the buffer may grow to. |
| `growth-percentage` | `0` | Percentage the buffer grows by (capped at `max-capacity`) when full and growth is enabled. |
| `growth-enabled` | `false` | When `false`, a full buffer goes straight to synchronous overflow persistence — see [operations/audit-trail.md](../operations/audit-trail.md#buffer-overflow-and-the-background-writer). |

### `database` category parameters (`implementation.parameters`)

**Select this category if `$Audit` (the script binding) needs its atomicity guarantee — but `$Audit` works either way.** `$Audit.logEvent(...)` is injected into every request unconditionally, independently of this block, and it never writes to a table itself: it builds a generic `AuditEvent` and calls the kernel's own `AuditLogger.logEvent(AuditEvent)` — the same call site system events use — after setting an ambient session. Only this `"database"` category actually reads that session and writes through it, joining the script's transaction; `"buffered-store"` (the **zero-config default** when `audit-trail` is absent entirely) correctly never looks for it, since an in-memory buffer has no transaction to join — it just logs the event the normal way instead. Either way, the call succeeds: a deployment that stays on `"buffered-store"` keeps `$Audit` fully usable, it just doesn't get the atomicity guarantee. See [architecture §3](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#3-audit--always-works-atomicity-is-a-bonus-only-some-implementations-provide) and [§5.3](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#53-auditqueue-are-injected-unconditionally--and-behave-very-differently-when-their-dependency-is-missing).

`payos-database-audit-trail` writes every event — both the kernel's own system events and, separately and atomically, `$Audit`'s own writes (see the architecture note above) — directly into a `payos_audit_event` table, one row per event, rather than buffering in memory. A background `Writer` thread then exports already-committed rows to the configured `store` backend (the same `IAuditTrailStore` mechanism `buffered-store` uses) and purges them afterward, once past the configured retention.

```json
{
  "audit-trail": {
    "implementation": {
      "category": "database",
      "parameters": {
        "disabled-event-types": ["API_EXEC"],
        "retention-days": 90,
        "poll-interval-seconds": 10
      }
    }
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `disabled-event-types` | `[]` | System event types (as emitted, e.g. `"API_EXEC"`) to suppress entirely — never written to `payos_audit_event`, never exported. Scoped to the kernel's own frequent system events only; a script's own `$Audit.logEvent(...)` call is never filtered by this list. |
| `retention-days` | `90` | How long an **already-exported** row stays in `payos_audit_event` before the background writer purges it. A row that hasn't been exported yet is never purged, regardless of age — purging before export would lose the event before it reaches the store. |
| `poll-interval-seconds` | `10` | How often the background writer checks for unexported rows and runs the retention purge. |

The table itself is created automatically at startup, independent of whatever `ddl-auto` a tenant has configured for its own entities — if it cannot be created for some tenant or the platform database, startup aborts (`FatalException`), since this category treats a lost audit event as unacceptable. See [architecture: atomicity across `$DB`, `$Audit`, and `$Queue` §5](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#5-startup-who-owns-table-creation-and-what-happens-when-it-fails).

### Required: register the mapping file

Creating the table is not enough by itself. `payos-database-audit-trail` creates `payos_audit_event` through a throwaway `SessionFactory` scoped only to its own mapping file — this does **not** make `"PayosAuditEvent"` a resolvable entity-name on the tenant's own, ongoing `$DB` `SessionFactory` that `$Audit.logEvent(...)` actually writes through. See [architecture §5.1](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#51-ensureschemafor-creates-the-table--it-does-not-register-the-entity) for the full mechanism.

**For a tenant's own `$DB`:** add `mappings/PayosAuditEvent.hbm.xml` (a classpath resource shipped inside the `payos-database-audit-trail` jar) to the `mapping-files` array of at least one `applications[]` entry that tenant is authorized for — see [configuration/json-configuration-reference.md §3.2](json-configuration-reference.md). This is **not** a `database-service.mapping-files` key; the kernel never reads one for a tenant's own database. One authorized application per tenant is enough — mapping files are unioned across every application a tenant can reach.

**For the platform database:** add the same file to `platform-database-service.mapping-files` directly — see [configuration/platform-database-service.md](platform-database-service.md#keys). Unlike the tenant case, this key *is* read directly, with no `applications[]` indirection.

**Forgetting this step is caught at startup, not at the first `$Audit.logEvent(...)` call.** Right after creating the table, this module probes whether `"PayosAuditEvent"` is actually resolvable on that same `SessionFactory` — a deployment that forgot the mapping-files entry gets a clear `FatalException` naming the missing file and the exact configuration to fix, instead of a raw `org.hibernate.MappingException: Unknown entity` surfacing the first time a script calls `$Audit`.

### `payos_audit_event` table schema

Normally nothing to act on: `ensureSchemaFor(...)` creates this table automatically at startup (above), independent of a tenant's own `ddl-auto`. Pre-create it by hand instead when policy requires DBA-approved DDL ahead of any deployment, or when a tenant's connection has no DDL privilege at all (`ddl-auto: validate`/`none` everywhere, including for this table). The authoritative definition is the Hibernate mapping shipped in the `payos-database-audit-trail` jar (`mappings/PayosAuditEvent.hbm.xml`) — the table below mirrors it exactly; the SQL beneath is one worked PostgreSQL rendering of it, not a second source of truth.

| Column | Property | Hibernate type | Nullable | Notes |
| --- | --- | --- | --- | --- |
| `event_id` | `eventId` | `string` | No (PK) | Assigned by the caller — never generated by the database. |
| `schema_version` | `schemaVersion` | `integer` | Yes | |
| `event_type` | `eventType` | `string` | **No** | e.g. `"API_EXEC"`, `"SESSION_CREATED"`. |
| `tenant_id` | `tenantId` | `string` | Yes | Null for a platform-scoped event (`logStartup`, or `atomicityScope: "platform"`). |
| `correlation_id` | `correlationId` | `string` | Yes | |
| `app_id` | `appId` | `string` | Yes | |
| `path` | `path` | `string` | Yes | |
| `user_id` | `userId` | `string` | Yes | Masked by `SensitiveFieldMasker` before this point when sensitive. |
| `result` | `result` | `string` | Yes | e.g. `"SUCCESS"`, `"ERROR"`. |
| `operation` | `operation` | `string` | Yes | |
| `resource_type` | `resourceType` | `string` | Yes | |
| `resource_id` | `resourceId` | `string` | Yes | |
| `business_keys` | `businessKeysJson` | `text` | Yes | Jackson-serialized JSON; no native JSON column (dynamic-map mapping). |
| `fields` | `fieldsJson` | `text` | Yes | Same JSON-as-text treatment as `business_keys`. |
| `created_at` | `createdAt` | `timestamp` | **No** | |
| `exported_at` | `exportedAt` | `timestamp` | Yes | Null until the background `Writer` exports the row to the configured `store`. |

```sql
CREATE TABLE payos_audit_event (
    event_id        VARCHAR(255) NOT NULL PRIMARY KEY,
    schema_version  INTEGER,
    event_type      VARCHAR(255) NOT NULL,
    tenant_id       VARCHAR(255),
    correlation_id  VARCHAR(255),
    app_id          VARCHAR(255),
    path            VARCHAR(255),
    user_id         VARCHAR(255),
    result          VARCHAR(255),
    operation       VARCHAR(255),
    resource_type   VARCHAR(255),
    resource_id     VARCHAR(255),
    business_keys   TEXT,
    fields          TEXT,
    created_at      TIMESTAMP NOT NULL,
    exported_at     TIMESTAMP
);
-- Optional, not created by ensureSchemaFor: an index on (tenant_id, exported_at) speeds up the
-- background writer's own "find unexported rows" scan at high volume; neither column is unique.
CREATE INDEX idx_payos_audit_event_unexported ON payos_audit_event (tenant_id, exported_at);
```

`VARCHAR(255)` reflects Hibernate's default length for an unqualified `string` type under most dialects (including PostgreSQL, the example above) — the mapping sets no explicit `length`, so this is what `ensureSchemaFor` itself would generate; some dialects default differently, and an operator free to choose their own length is safe to widen it (never narrow it below what a real value could need, especially `event_id`/`correlation_id`/`fields` if your `businessKeys`/`fields` payloads are large). Table and column names are fixed by the mapping — do not rename them.

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

- [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md) — ordered end-to-end setup for `$DB`/`$PlatformDB` + `$Audit` + `$Queue`, including the `category: "database"` + mapping-file steps above.
- [architecture/atomicity-db-audit-queue-v9-2026-10-19.md](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md) — `$Audit`'s own strict, in-transaction write (distinct from this block, which governs system events and, for the `database` category, where `$Audit`'s writes also land).
- [operations/audit-trail.md](../operations/audit-trail.md) — how the feature behaves operationally, metrics, known limitations, troubleshooting.
- [operations/metrics-catalog-v2-2026-09-11.md](../operations/metrics-catalog-v2-2026-09-11.md) — full audit-trail metric inventory.
- [operations/observability.md](../operations/observability.md) — the always-on audit log line this capability sits behind.
