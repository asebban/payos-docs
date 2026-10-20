Created: 2026-10-18
Last updated: 2026-10-19
Version: 3 — supersedes v2: flags that at least one `servers[]` queue entry is required for `$Queue` to work at all, independently of `queue-service` — see [Queue keys](#queue-keys-queue). See also the new end-to-end [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md).

# Servers configuration

The `servers` array declares the transport listeners PayOS starts. Each entry is started via
`Servers.start(host, port, protocol[, config])`, which selects a `ServerProvider` by
protocol through `ServiceLoader`. The transport architecture is in
[architecture/request-processing.md](../architecture/request-processing.md) and
[architecture/extensibility.md](../architecture/extensibility.md).

## Shape

```json
{
  "servers": [
    { "protocol": "https", "host": "0.0.0.0", "port": 8443,
      "keystore": "config/keystore.p12", "keystorePassword": "${KS_PASS}",
      "keystoreType": "PKCS12", "keyAlias": "payos", "keyPassword": "${KEY_PASS}" },

    { "protocol": "tcp", "host": "0.0.0.0", "port": 9000,
      "tcp-handlers-dir": "tcp-handlers" },

    { "protocol": "queue", "host": "localhost", "port": 4222,
      "type": "nats", "consumer-topic": "payos.requests",
      "maxAttempts": 10, "pollIntervalSeconds": 10 }
  ]
}
```

## Common keys (all protocols)

From `IConfigSpec.Servers.Server`:

| Key | Purpose |
| --- | --- |
| `protocol` | `http`, `https`, `tcp`, or `queue`. Selects the `ServerProvider`. |
| `host` | Bind address. |
| `port` | Bind port. |

## TLS keys (`https`)

| Key | Purpose |
| --- | --- |
| `keystore` | Path to the keystore. |
| `keystorePassword` | Keystore password. |
| `keystoreType` | e.g. `PKCS12`, `JKS`. |
| `keyAlias` | Alias of the server key. |
| `keyPassword` | Key password. |

HTTPS adds HSTS and serves cookies with `Secure`. See
[security-architecture](../architecture/security-architecture.md).

## TCP keys (`tcp`)

| Key | Purpose |
| --- | --- |
| `tcp-handlers-dir` | Directory scanned for `TcpMessageDecoder` / `TcpMessageEncoder` / `TcpMessageHandler` plugin JARs. |

Resolution order for the handlers dir: **system property → `TCP_HANDLERS_DIR` env var →
this server key**. The first concrete plugin implementation wins; otherwise UTF-8 defaults
apply. See [architecture/extensibility.md](../architecture/extensibility.md).

## Queue keys (`queue`)

**At least one `servers[]` entry with `protocol: "queue"` is required for `$Queue` (the script binding) to work at all** — even if nothing actually needs to consume a message from it. `$Queue.publish(...)` is injected into every request whenever `queue-service` (the publisher block) is configured, entirely independently of this `servers[]` entry — but `payos_queue_outbox` is only ever created by `QueueOutboxDispatcher.start()`, which only ever runs from `QueueServer.start()`, i.e. only when this entry exists. A deployment with `queue-service` configured but no `servers[]` queue entry gets `$Queue` injected and callable, but `publish(...)` fails with a raw `org.hibernate.MappingException: Unknown entity` the first time it's called. See [architecture §5.3](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#53-auditqueue-are-injected-unconditionally--their-table-isnt-created-unless-something-else-is-also-configured).

| Key | Default | Purpose |
| --- | --- | --- |
| `type` | — | Queue connector type (e.g. `nats`). |
| `consumer-topic` | `default-topic` | Topic the server consumes requests from (consumer topic). |
| `maxAttempts` | `10` | Transactional Outbox dispatcher (`QueueOutboxDispatcher`, paired with this same server — see [architecture/atomicity-db-audit-queue-v9-2026-10-19.md §4.1](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#41-the-dispatcher)): a logging threshold only, never an abandon threshold — past it, a stuck message is logged at `ERROR` instead of `WARN` but keeps being retried indefinitely. |
| `pollIntervalSeconds` | `10` | How often the dispatcher polls `payos_queue_outbox` for `PENDING` rows. |

The queue transport consumes request envelopes and replies on the reply topic. See
[queue-service.md](queue-service.md) for the connector itself. The same entry also starts the
Transactional Outbox dispatcher that drains `$Queue.publish(destination, message)`'s writes —
not a separate `servers[]` entry, see
[architecture/atomicity-db-audit-queue-v9-2026-10-19.md §4](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#4-queue--the-transactional-outbox-pattern).

### Required: register the outbox mapping file

Creating `payos_queue_outbox` is not enough by itself. `payos-server-queue` creates it through a throwaway `SessionFactory` scoped only to its own mapping file — this does **not** make `"PayosQueueOutbox"` a resolvable entity-name on the tenant's own, ongoing `$DB` `SessionFactory` that `QueueBinding` actually writes through. See [architecture §5.1](../architecture/atomicity-db-audit-queue-v9-2026-10-19.md#51-ensureschemafor-creates-the-table--it-does-not-register-the-entity) for the full mechanism.

**For a tenant's own `$DB`:** add `mappings/PayosQueueOutbox.hbm.xml` (a classpath resource shipped inside the `payos-server-queue` jar) to the `mapping-files` array of at least one `applications[]` entry that tenant is authorized for — see [configuration/json-configuration-reference.md §3.2](json-configuration-reference.md). This is **not** a `database-service.mapping-files` key; the kernel never reads one for a tenant's own database. One authorized application per tenant is enough — mapping files are unioned across every application a tenant can reach.

**For the platform database:** add the same file to `platform-database-service.mapping-files` directly — see [configuration/platform-database-service.md](platform-database-service.md#keys). Unlike the tenant case, this key *is* read directly, with no `applications[]` indirection.

**Forgetting this step is caught at `start()`, not at the first `$Queue.publish(...)` call.** Right after creating the table, `QueueOutboxDispatcher` probes whether `"PayosQueueOutbox"` is actually resolvable on that same `SessionFactory` — a deployment that forgot the mapping-files entry gets a clear `IllegalStateException` naming the missing file and the exact configuration to fix. Like any other `start()` failure here, this is non-fatal to the rest of PayOS (§5.2 of the architecture doc): the affected queue server (and its paired dispatcher) simply doesn't start.

### `payos_queue_outbox` table schema

Normally nothing to act on: `ensureSchemaFor(...)` creates this table automatically at startup (above), independent of a tenant's own `ddl-auto`. Pre-create it by hand instead when policy requires DBA-approved DDL ahead of any deployment, or when a tenant's connection has no DDL privilege at all (`ddl-auto: validate`/`none` everywhere, including for this table). The authoritative definition is the Hibernate mapping shipped in the `payos-server-queue` jar (`mappings/PayosQueueOutbox.hbm.xml`) — the table below mirrors it exactly; the SQL beneath is one worked PostgreSQL rendering of it, not a second source of truth.

| Column | Property | Hibernate type | Nullable | Notes |
| --- | --- | --- | --- | --- |
| `id` | `id` | `string` | No (PK) | Assigned by `QueueBinding` — never generated by the database. |
| `tenant_id` | `tenantId` | `string` | Yes | Null for a platform-scoped row (`atomicityScope: "platform"`). |
| `destination` | `destination` | `string` | **No** | |
| `payload` | `payload` | `text` | Yes | The plain-`Map` overloads' JSON-serialized payload, or the `QueueMessage` form's own payload as-is. |
| `metadata` | `metadataJson` | `text` | Yes | Jackson-serialized JSON; no native JSON column (dynamic-map mapping). Carries `replyTo` when set. |
| `status` | `status` | `string` | **No** | `"PENDING"` or `"DISPATCHED"` — no `"FAILED"`/dead-letter status (§4.2 of the architecture doc). |
| `attempts` | `attempts` | `integer` | **No** | |
| `created_at` | `createdAt` | `timestamp` | **No** | |
| `last_attempted_at` | `lastAttemptedAt` | `timestamp` | Yes | Null until the dispatcher's first attempt. |
| `last_error` | `lastError` | `text` | Yes | Truncated to 1000 characters by the dispatcher before being stored. |

```sql
CREATE TABLE payos_queue_outbox (
    id                 VARCHAR(255) NOT NULL PRIMARY KEY,
    tenant_id          VARCHAR(255),
    destination        VARCHAR(255) NOT NULL,
    payload            TEXT,
    metadata           TEXT,
    status             VARCHAR(255) NOT NULL,
    attempts           INTEGER NOT NULL,
    created_at         TIMESTAMP NOT NULL,
    last_attempted_at  TIMESTAMP,
    last_error         TEXT
);
-- Optional, not created by ensureSchemaFor: an index on (status, created_at) speeds up the
-- dispatcher's own "PENDING rows, oldest first" scan at high volume.
CREATE INDEX idx_payos_queue_outbox_pending ON payos_queue_outbox (status, created_at);
```

`VARCHAR(255)` reflects Hibernate's default length for an unqualified `string` type under most dialects (including PostgreSQL, the example above) — the mapping sets no explicit `length`, so this is what `ensureSchemaFor` itself would generate; some dialects default differently, and an operator free to choose their own length is safe to widen it (never narrow it below what a real value could need). Table and column names are fixed by the mapping — do not rename them.

## HTTP endpoints & Swagger UI

The HTTP/HTTPS transport exposes built-in endpoints (`/health`, `/me`, `/callback`,
`/logout`, `/stop`, `/openapi.yaml`, `/swagger/**`) in addition to application routes
`/{appId}/[api|page|component|menu]/**`. The full list is in
[reference/http-endpoints.md](../reference/http-endpoints.md).

Swagger UI is configured with the `swaggerUI` block:

| Key | Default | Purpose |
| --- | --- | --- |
| `local-only` | `true` | Restrict Swagger UI to localhost. |
| `openapi-yaml` | — | Path to the OpenAPI document served at `/openapi.yaml`. |

Generate the OpenAPI document with [`pdoc`](../cli-tools/pdoc.md) — see
[developer/api-documentation.md](../developer/api-documentation.md).

## Security headers (HTTP)

The HTTP transport sets hardened headers on responses: `X-Content-Type-Options: nosniff`,
`X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store...`,
`Pragma: no-cache`, and a strict CSP (`default-src 'none'; frame-ancestors 'none';
base-uri 'none'`), plus HSTS on HTTPS. CORS is resolved app → tenant → global (see
[security-oidc.md](security-oidc.md)).

## Next

- [developer/atomicity-setup-guide.md](../developer/atomicity-setup-guide.md) — ordered end-to-end setup for `$DB`/`$PlatformDB` + `$Audit` + `$Queue`, including the `servers[]` queue entry + mapping-file steps above.
- [security-oidc.md](security-oidc.md) — authentication & CORS.
- [reference/http-endpoints.md](../reference/http-endpoints.md) — endpoint catalog.
- [developer/queue-setup-guide.md](../developer/queue-setup-guide.md) — configuring and using the `queue` transport (consumer side) alongside the publisher side, with a full worked example.
