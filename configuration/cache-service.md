# Cache service configuration

The `cache-service` block configures the optional distributed cache abstraction (`ICacheStore`, `payos-kernel`, package `ma.s2m.payos.cache`) used to store data shared across every instance of a bundle — or across different bundles running on the same cluster. Unlike [`idempotency`](idempotency.md) and the session store, this service is **disabled by default**: with the block absent, or `enabled` not `true`, `PayOSConfig.getCacheStore()` stays `null` and there is no silent in-memory fallback — callers must handle a `null` store themselves.

## Shape

```json
{
  "cache-service": {
    "enabled": true,
    "storeType": "redis",
    "storeRedis": {
      "host": "127.0.0.1",
      "port": 6379,
      "password": "...",
      "database": 0,
      "tls": false,
      "keyPrefix": "payos:cache:"
    }
  }
}
```
or

```json
{
  "cache-service": {
    "enabled": true,
    "storeType": "memory"
  }
}
```


`storeRedis` is only read when `storeType` is `redis` — with `storeType: "memory"` (the default when the block is enabled but `storeType` is omitted) it is ignored. Keeping it present in `bootstrap.json` even while running on `memory` is fine: switching `storeType` to `redis` later doesn't require adding the block from scratch.

## Keys

From `IConfigSpec.CacheService`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `false` | Enable the cache service. When absent or `false`, the block is ignored entirely and `PayOSConfig.getCacheStore()` returns `null` — there is no automatic memory fallback (unlike `idempotency`). |
| `storeType` | `memory` | Store backend: `memory` or `redis`. Only read when `enabled` is `true`. See [Store backends](#store-backends) below. |
| `storeRedis` | — | Redis connection block, read only when `storeType` is `redis`. See [Redis store configuration](#redis-store-configuration). |

## Store backends

`storeType` selects the `ICacheStore` implementation, resolved by `CacheStores.resolve(...)` (`payos-kernel`, `ma.s2m.payos.cache`) — the same `ServiceLoader`-based pattern used for `idempotency.storeType` (see [idempotency.md](idempotency.md#store-backends)):

| `storeType` | Implementation | Requirements |
| --- | --- | --- |
| `memory` (default) | `InMemoryCacheStore` (module `cache-service-memory`) | Single-node only, data lost on restart. `increment` is made atomic per key via `ConcurrentHashMap.compute` — safe against concurrent callers within the same JVM, but not shared across processes. The `cache-service-memory` jar must be on the classpath (already added to `payos-runtime`'s dependencies); resolved via `ServiceLoader.load(ICacheStoreFactory.class)`. |
| `redis` | `RedisCacheStore` (module `cache-service-redis`) | Shared across every process connected to the same Redis instance/cluster — the actual point of this abstraction. `increment` runs as a single Lua script (`EXISTS`+`INCRBY`+conditional `EXPIRE`), so it stays atomic across processes too, not just within one JVM. The `cache-service-redis` jar must be on the classpath (already added to `payos-runtime`'s dependencies); resolved via `ServiceLoader.load(ICacheStoreFactory.class)`. |

Unlike `idempotency`/session, **neither backend is built into `payos-kernel`** — both `memory` and `redis` are separate modules discovered purely through SPI. An unknown `storeType`, or a backend module missing from the classpath, throws `CacheStoreException` at startup — deliberately not a silent fallback.

Values are plain `String` — the caller is responsible for JSON-encoding/decoding structured data before calling `put`/`get`. `ICacheStore` itself has no built-in tenant-namespacing: a Java caller that needs entries to not collide across tenants prefixes the key itself (e.g. `tenantId + ":" + key`). Scripts don't need to do this — see [`$Cache` binding in scripts](#cache-binding-in-scripts) below, which namespaces by tenant automatically.

### Redis store configuration

From `IConfigSpec.CacheService.Redis` (block key `storeRedis`), read by `RedisCacheStoreFactory`:

| Key | Default | Purpose |
| --- | --- | --- |
| `host` | `localhost` | Redis host. |
| `port` | `6379` | Redis port. |
| `password` | — | Redis auth password (optional). |
| `database` | `0` | Redis logical database index. |
| `tls` | `false` | Enable TLS for the Redis connection. |
| `keyPrefix` | `payos:cache:` | Key prefix for cached entries — namespaced identically to `session-service-redis`'s `payos:session:` and `idempotency-service-redis`'s `payos:idempotency:`, so the same Redis instance/cluster can be shared between all three without key collisions. |

### `increment` semantics (both backends)

`increment(key, delta, ttlSeconds)` is meant for counters (e.g. rate/quota tracking) shared across instances:

- The TTL is applied **only when the call creates the key** — an increment on an existing counter never resets or extends its TTL. This is deliberate: it lets a caller implement a fixed-window counter (e.g. "max N calls per hour") by always passing the window's TTL, without accidentally sliding the window on every call.
- Both backends guarantee `increment` is atomic: `InMemoryCacheStore` via `ConcurrentHashMap.compute` (atomic within the JVM), `RedisCacheStore` via a single Lua script executed server-side (atomic across every process sharing that Redis).

## Resolution order

Unlike `idempotency`, `cache-service` keys have **no** system property / environment variable fallback — they are read only from the `bootstrap.json` block:

```
cache-service.<key> in bootstrap.json
  → built-in default
```

## Wiring

`BootServer` calls `CacheServiceInitializer.initialize(PayOSConfig.settings)` after the idempotency service is initialized. If `cache-service.enabled` is not `true`, the initializer logs and returns without touching `PayOSConfig` — `getCacheStore()` stays `null`. Otherwise it resolves the store via `CacheStores.resolve(storeType, config)` and registers it with `PayOSConfig.setCacheStore(store)`.

## `$Cache` binding in scripts

When a cache store is configured, `ApiResourceHandler` exposes it to every API script and hook as `$Cache` (`ma.s2m.payos.scripting.CacheBinding`, `payos` repo), right after `$Secrets` is registered. If `cache-service.enabled` is not `true`, `PayOSConfig.getCacheStore()` is `null` and `$Cache` is not put in scope at all — a script referencing `$Cache` in that case fails with a `ReferenceError`, not a silent no-op.

```javascript
// Store a value for 60 seconds
$Cache.put("quote:" + quoteId, JSON.stringify(quote), 60);

// Read it back — returns null (not undefined, not an exception) if missing or expired
const raw = $Cache.get("quote:" + quoteId);
const quote = raw ? JSON.parse(raw) : null;

// Atomic counter, e.g. an hourly quota — automatically scoped to the current tenant
const callsThisHour = $Cache.increment("calls", 1, 3600);
```

| Method | Signature | Description |
| --- | --- | --- |
| `put` | `put(key: string, value: string, ttlSeconds: long): void` | Stores `value` under `key`. `ttlSeconds <= 0` means no expiry. |
| `get` | `get(key: string): string \| null` | Returns the cached value, or `null` if the key is missing or expired. |
| `remove` | `remove(key: string): void` | Deletes the entry, if present. |
| `exists` | `exists(key: string): boolean` | Checks presence without returning the value. |
| `increment` | `increment(key: string, delta: long, ttlSeconds: long): long` | Atomically adds `delta` and returns the new total. `ttlSeconds` is applied only when the call creates the key — see [`increment` semantics](#increment-semantics-both-backends). |

Unlike the raw `ICacheStore` contract, `CacheBinding` captures the current request's tenant and prefixes every key with it before calling the store — the same way `SecretsBinding` scopes `ISecretProvider` calls. A script never passes a tenant explicitly, and two tenants calling `$Cache.put("quote", ...)` with the identical key never collide. There is currently no way for a script to intentionally share a `$Cache` entry across tenants; that's only possible from Java code calling `PayOSConfig.getCacheStore()` directly. Values are always plain strings — encode/decode structured data with `JSON.stringify`/`JSON.parse` in the script, same as the Java-side contract.

## Next

- [idempotency.md](idempotency.md) — the closest sibling feature (shared, TTL-based store), same SPI/resolver pattern.
- [json-configuration-reference.md](json-configuration-reference.md) — full config key reference across every block.
