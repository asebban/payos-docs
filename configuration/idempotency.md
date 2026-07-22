# Idempotency configuration

The `idempotency` block controls `IdempotencyService`, which caches API responses by a client-supplied idempotency key so mutating requests (e.g. payment operations) are safe to retry. The service and pipeline integration are described in [architecture/security-architecture.md](../architecture/security-architecture.md#idempotency); request-flow placement is in [architecture/request-processing.md](../architecture/request-processing.md).

## Shape

```json
{
  "idempotency": {
    "enabled": true,
    "ttlSeconds": 86400,
    "headerName": "X-Idempotency-Key",
    "failOnAbsenceOfIdempotencyKey": true,
    "storeType": "memory",
    "storeRedis": {
      "host": "127.0.0.1",
      "port": 6379,
      "password": "...",
      "database": 0,
      "tls": false,
      "keyPrefix": "payos:idempotency:"
    }
  }
}
```

`storeRedis` is only read when `storeType` is `redis` — with `storeType: "memory"` (the default)
it is ignored. Keeping it present in `bootstrap.json` even while running on `memory` is fine:
switching `storeType` to `redis` later doesn't require adding the block from scratch.

## Keys

From `IConfigSpec.Idempotency`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `true` | Enable the idempotency service. When `false`, `checkIdempotency` is a no-op and `storeResponse` never caches. |
| `ttlSeconds` | `86400` (24h) | How long a cached response stays valid for replay. |
| `headerName` | `X-Idempotency-Key` | Request header carrying the idempotency key. |
| `failOnAbsenceOfIdempotencyKey` | `true` | When `true`, a request with a missing/blank idempotency key is rejected (`400 Bad Request`) before script execution. When `false`, such a request simply proceeds without an idempotency check (no rejection, no caching). |
| `storeType` | `memory` | Store backend: `memory` or `redis`. See [Store backends](#store-backends) below. |
| `storeRedis` | — | Redis connection block, read only when `storeType` is `redis`. See [Redis store configuration](#redis-store-configuration). |

## Store backends

`storeType` selects the `IIdempotencyStore` implementation, resolved by `IdempotencyStores.resolve(...)`
(`payos-kernel`, `ma.s2m.payos.idempotency`) — the same pattern used for `security.sessionStoreType`
(see [session store](../developer/session-store-redis-design.md)):

| `storeType` | Implementation | Requirements |
| --- | --- | --- |
| `memory` (default) | `InMemoryIdempotencyStore` | None — single-node only, records lost on restart. |
| `redis` | `RedisIdempotencyStore` (module `idempotency-service-redis`) | The `idempotency-service-redis` jar must be on the classpath (already added to `payos-runtime`'s dependencies); resolved via `ServiceLoader.load(IIdempotencyStoreFactory.class)`. |

An unknown `storeType` (typo, or a backend module missing from the classpath) throws
`IdempotencyStoreException` at startup — deliberately not a silent fallback to `memory`.

> A database-backed store (`DatabaseIdempotencyStore`, `storeType: "database"`) existed briefly
> and was removed: it required a Hibernate dynamic-map mapping to be declared by some application
> for the `payos_idempotency` entity (this system's tables are all Hibernate dynamic-map entities,
> not annotated Java classes — see [developer/data-access.md](../developer/data-access.md)), and,
> more seriously, `ApiResourceHandler` calls `checkIdempotency(request)` **before** the per-tenant
> database scope is opened (`setCurrentTenant`/`beginRequestScope` run later), so the read path had
> no correct tenant context in a multi-tenant deployment. It was never exercised in a real
> deployment. If a persistent-but-not-Redis backend is needed again, both issues need fixing
> first — not just re-adding the store class.

### Redis store configuration

From `IConfigSpec.Idempotency.Redis` (block key `storeRedis`), read by `RedisIdempotencyStoreFactory`:

| Key | Default | Purpose |
| --- | --- | --- |
| `host` | `localhost` | Redis host. |
| `port` | `6379` | Redis port. |
| `password` | — | Redis auth password (optional). |
| `database` | `0` | Redis logical database index. |
| `tls` | `false` | Enable TLS for the Redis connection. |
| `keyPrefix` | `payos:idempotency:` | Key prefix for stored idempotency records — namespaced identically to `session-service-redis`'s `payos:session:`, so the same Redis instance/cluster can be shared between distributed stores without key collisions. |

Records are stored with Redis' native expiry (`SETEX`/`SET ... EX ttlSeconds`), so `evictExpired()`
is a no-op for this backend — Redis has already removed the key once `ttlSeconds` elapses.

## Resolution order

Each key is resolved in this order, first match wins:

```
idempotency.<key> in bootstrap.json
  → system property (payos.idempotency.<key>)
  → environment variable (PAYOS_IDEMPOTENCY_<KEY>)
  → built-in default
```

| Key | System property | Environment variable |
| --- | --- | --- |
| `enabled` | `payos.idempotency.enabled` | `PAYOS_IDEMPOTENCY_ENABLED` |
| `ttlSeconds` | `payos.idempotency.ttlSeconds` | `PAYOS_IDEMPOTENCY_TTL_SECONDS` |
| `headerName` | `payos.idempotency.headerName` | `PAYOS_IDEMPOTENCY_HEADER_NAME` |
| `failOnAbsenceOfIdempotencyKey` | `payos.idempotency.failOnAbsenceOfIdempotencyKey` | `PAYOS_IDEMPOTENCY_FAIL_ON_ABSENCE_OF_IDEMPOTENCY_KEY` |
| `storeType` | `payos.idempotency.storeType` | `PAYOS_IDEMPOTENCY_STORE_TYPE` |

The system property / environment variable fallback predates the `bootstrap.json` block and is
kept for backward compatibility; prefer configuring `idempotency` in `bootstrap.json` for new
deployments. `storeRedis` sub-keys have no system property / environment variable fallback — they
are only read from the `bootstrap.json` block by `RedisIdempotencyStoreFactory`.

## Wiring

`BootServer` constructs `IdempotencyConfig(PayOSConfig.settings)`, then resolves the store via
`IdempotencyStores.resolve(idempotencyConfig.getStoreType(), idempotencyConfig.getStoreConfig())`
— `memory` is resolved directly in `payos-kernel`; any other type (`redis`) goes through
`ServiceLoader.load(IIdempotencyStoreFactory.class)`. The resulting `IdempotencyService` is
registered via `PayOSConfig.setIdempotencyService(...)`. `ApiResourceHandler` calls
`checkIdempotency(request)` before script execution and `storeResponse(request, response)` after
a successful run.

## Next

- [Architecture: security](../architecture/security-architecture.md#idempotency)
- [Architecture: request processing](../architecture/request-processing.md)
