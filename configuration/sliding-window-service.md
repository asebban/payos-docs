# Sliding window counter service configuration

The `sliding-window-service` block configures the optional exact sliding-window event counter abstraction (`ISlidingWindowCounter`, `payos-kernel`, package `ma.s2m.payos.ratelimit`) used for quota/rate-limit style checks — e.g. "no more than N calls per tenant per hour" — that must not allow a burst to straddle two windows. Unlike `ICacheStore#increment`, which implements a fixed/tumbling window (the TTL is set once, on key creation, and never resets, so a burst at the boundary between two windows can briefly exceed twice the intended limit), `ISlidingWindowCounter` counts events within a trailing window ending "now": the window slides continuously rather than resetting abruptly. Like [`cache-service`](cache-service.md), this service is **disabled by default**: with the block absent, or `enabled` not `true`, `PayOSConfig.getSlidingWindowCounter()` stays `null` and there is no silent in-memory fallback — callers must handle a `null` counter themselves.

## Shape

```json
{
  "sliding-window-service": {
    "enabled": true,
    "storeType": "redis",
    "storeRedis": {
      "host": "127.0.0.1",
      "port": 6379,
      "password": "...",
      "database": 0,
      "tls": false,
      "keyPrefix": "payos:slidingwindow:"
    }
  }
}
```

or

```json
{
  "sliding-window-service": {
    "enabled": true,
    "storeType": "memory"
  }
}
```

`storeRedis` is only read when `storeType` is `redis` — with `storeType: "memory"` (the default when the block is enabled but `storeType` is omitted) it is ignored. Keeping it present in `bootstrap.json` even while running on `memory` is fine: switching `storeType` to `redis` later doesn't require adding the block from scratch.

## Keys

From `IConfigSpec.SlidingWindowService`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `false` | Enable the sliding window counter service. When absent or `false`, the block is ignored entirely and `PayOSConfig.getSlidingWindowCounter()` returns `null` — there is no automatic memory fallback. |
| `storeType` | `memory` | Store backend: `memory` or `redis`. Only read when `enabled` is `true`. See [Store backends](#store-backends) below. |
| `storeRedis` | — | Redis connection block, read only when `storeType` is `redis`. See [Redis store configuration](#redis-store-configuration). |

## Store backends

`storeType` selects the `ISlidingWindowCounter` implementation, resolved by `SlidingWindowCounters.resolve(...)` (`payos-kernel`, `ma.s2m.payos.ratelimit`) — the same `ServiceLoader`-based pattern used for `cache-service.storeType` (see [cache-service.md](cache-service.md#store-backends)):

| `storeType` | Implementation | Requirements |
| --- | --- | --- |
| `memory` (default) | `InMemorySlidingWindowCounter` (module `sliding-window-counter-memory`) | Single-node only, data lost on restart. Each key holds a per-key `ArrayDeque` of event timestamps guarded by its own monitor, so the prune+record+count sequence is atomic within the JVM but not shared across processes — in a multi-instance deployment, each instance enforces its own quota independently (effectively `N ×` the intended limit across `N` instances). The `sliding-window-counter-memory` jar must be on the classpath (already added to `payos-runtime`'s dependencies); resolved via `ServiceLoader.load(ISlidingWindowCounterFactory.class)`. |
| `redis` | `RedisSlidingWindowCounter` (module `sliding-window-counter-redis`) | Shared across every process connected to the same Redis instance/cluster — the only backend that actually enforces a single global quota across a multi-instance deployment. Each key is a Redis sorted set (`ZSET`) scored by event timestamp; prune (`ZREMRANGEBYSCORE`), record (`ZADD`), refresh expiry (`PEXPIRE`), and count (`ZCARD`) run as a single Lua script, so the whole sequence is atomic across every instance sharing that Redis. The `sliding-window-counter-redis` jar must be on the classpath (already added to `payos-runtime`'s dependencies); resolved via `ServiceLoader.load(ISlidingWindowCounterFactory.class)`. |

Neither backend is built into `payos-kernel` — both `memory` and `redis` are separate modules discovered purely through SPI, exactly like `cache-service`. An unknown `storeType`, or a backend module missing from the classpath, throws `SlidingWindowCounterException` at startup — deliberately not a silent fallback.

### Redis store configuration

From `IConfigSpec.SlidingWindowService.Redis` (block key `storeRedis`), read by `RedisSlidingWindowCounterFactory`:

| Key | Default | Purpose |
| --- | --- | --- |
| `host` | `localhost` | Redis host. |
| `port` | `6379` | Redis port. |
| `password` | — | Redis auth password (optional). |
| `database` | `0` | Redis logical database index. |
| `tls` | `false` | Enable TLS for the Redis connection. |
| `keyPrefix` | `payos:slidingwindow:` | Key prefix for sliding-window sorted sets — namespaced identically in spirit to `session-service-redis`'s `payos:session:`, `idempotency-service-redis`'s `payos:idempotency:`, and `cache-service-redis`'s `payos:cache:`, so the same Redis instance/cluster can be shared between all of them without key collisions. |

## `recordAndCount` / `count` semantics (both backends)

`ISlidingWindowCounter` has three operations:

- `recordAndCount(key, windowMillis)` — records one event now, then returns the number of events for `key` within the trailing `windowMillis` window ending now (including the one just recorded).
- `count(key, windowMillis)` — returns the same trailing-window count, without recording a new event.
- `reset(key)` — discards every recorded event for `key`.

Both `recordAndCount` and `count` prune events older than the window on every call, so the count is always exact for the window ending at the instant of the call — there is no fixed reset boundary the way `ICacheStore#increment`'s TTL has one. Both backends guarantee `recordAndCount` is atomic: `InMemorySlidingWindowCounter` via a per-key monitor, `RedisSlidingWindowCounter` via a single Lua script executed server-side (atomic across every process sharing that Redis).

## Resolution order

Like `cache-service`, `sliding-window-service` keys have **no** system property / environment variable fallback — they are read only from the `bootstrap.json` block:

```
sliding-window-service.<key> in bootstrap.json
  → built-in default
```

## Wiring

`BootServer` calls `SlidingWindowServiceInitializer.initialize(PayOSConfig.settings)` right after the cache service is initialized. If `sliding-window-service.enabled` is not `true`, the initializer logs and returns without touching `PayOSConfig` — `getSlidingWindowCounter()` stays `null`. Otherwise it resolves the counter via `SlidingWindowCounters.resolve(storeType, config)` and registers it with `PayOSConfig.setSlidingWindowCounter(counter)`.

## `$SlidingWindow` binding in scripts

When a counter is configured, `ApiResourceHandler` exposes it to every API script and hook as `$SlidingWindow` (`ma.s2m.payos.scripting.SlidingWindowBinding`, `payos` repo), right after `$Cache` is registered. Unlike `$Cache`, this binding is **read-only**: it exposes only `count(key, windowMillis)` — `recordAndCount` and `reset` are not exposed at all, so a script can inspect the current usage for a quota decision but cannot record a new event or wipe the counter. See [developer/sliding-window-usage.md](../developer/sliding-window-usage.md) for full usage and rationale.

```javascript
// Check how many calls this tenant has made in the trailing hour, without recording a new one
const callsThisHour = $SlidingWindow.count("calls", 3600000);
if (callsThisHour >= 1000) {
    $Errors.badRequest("Hourly quota exceeded");
}
```

Like `$Cache`, `SlidingWindowBinding` namespaces every key by the current tenant automatically — a script never passes a tenant explicitly, and never collides with another tenant reusing the same key name. There is currently no binding that lets a script record an event (`recordAndCount`) — that has to happen from Java code (e.g. a future hook/middleware) calling `PayOSConfig.getSlidingWindowCounter()` directly.

## Built-in consumer: tenant quota enforcement

This counter isn't only reachable from scripts — `TenantPolicyService` (`payos-kernel`) uses it to enforce the platform's own per-tenant `requestsPerMinute` quota (see [multi-tenancy.md](multi-tenancy.md#how-quotas-are-enforced-backend-selection)). When `sliding-window-service` is configured, tenant quota checks automatically switch from the legacy fixed-window `RateWindow` counter to this exact sliding window, using the same configured backend (so `storeType: "redis"` makes tenant quotas correct across a multi-instance deployment, not just `$Cache`/`$SlidingWindow` data). This happens transparently — there is nothing extra to configure beyond `sliding-window-service` itself.

## Next

- [cache-service.md](cache-service.md) — the closest sibling feature (pluggable `memory`/`redis` backend, same optional/no-fallback/SPI pattern), and the fixed-window alternative (`ICacheStore#increment`) this counter is meant to improve on for quota/rate-limit use cases.
- [multi-tenancy.md](multi-tenancy.md#how-quotas-are-enforced-backend-selection) — the built-in tenant quota mechanism that uses this counter automatically when configured.
- [developer/sliding-window-usage.md](../developer/sliding-window-usage.md) — using `$SlidingWindow` from scripts.
- [developer/tenant-quota-enforcement.md](../developer/tenant-quota-enforcement.md) — what tenant quota enforcement looks like from an application developer's perspective.
- [json-configuration-reference.md](json-configuration-reference.md) — full config key reference across every block.
