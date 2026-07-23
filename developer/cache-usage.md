# Cache usage (`$Cache`)

When the [cache service](../configuration/cache-service.md) is configured and enabled, scripts receive the `$Cache` binding — a `CacheBinding` that wraps the resolved `ICacheStore` (`memory` or `redis`, depending on `cache-service.storeType`). It exposes `put`/`get`/`remove`/`exists`/`increment` for storing plain-string data shared across every instance of a bundle or, with the `redis` backend, across different bundles on the same cluster. This page covers usage from JavaScript; backend selection and Redis connection settings are in [configuration/cache-service.md](../configuration/cache-service.md).

## Tenant scoping is automatic

`ICacheStore` itself has no tenant concept (by design — see [configuration/cache-service.md](../configuration/cache-service.md)), but `CacheBinding` captures the current request's tenant once and namespaces every key with it before calling the store, the same way `$Secrets` scopes `ISecretProvider` calls. A script never passes a tenant explicitly, and two tenants using the identical key name (e.g. both calling `$Cache.put("quote", ...)`) never collide:

```javascript
$Cache.put("quote:" + quoteId, JSON.stringify(quote), 60);   // stored under "<tenantId>:quote:<quoteId>"
```

There is currently no way for a script to intentionally share a `$Cache` entry across tenants — every key a script sees is implicitly scoped to its own tenant. (Cross-tenant/cross-bundle sharing is still possible from Java code that calls `PayOSConfig.getCacheStore()` directly, bypassing `CacheBinding`.)

## Storing and reading a value

Values are always plain strings — encode/decode structured data yourself with `JSON.stringify`/`JSON.parse`:

```javascript
function execute(request, controlData) {
    $Cache.put("quote:" + controlData.quoteId, JSON.stringify(controlData.quote), 60); // 60s TTL

    var raw = $Cache.get("quote:" + controlData.quoteId);   // String, or null if missing/expired
    var quote = raw ? JSON.parse(raw) : null;
    return quote;
}
```

`ttlSeconds <= 0` means the entry never expires:

```javascript
$Cache.put("feature-flags", JSON.stringify(flags), 0);
```

## Removing and checking presence

```javascript
$Cache.remove("quote:" + quoteId);

if ($Cache.exists("quote:" + quoteId)) {
    // ...
}
```

## Atomic counters (`increment`)

`increment(key, delta, ttlSeconds)` is for counters shared across instances — quotas, rate limits, simple metrics — where a plain read-then-write from a script would race against other instances:

```javascript
// Fixed-window hourly quota — automatically scoped to the current tenant
var callsThisHour = $Cache.increment("api-calls", 1, 3600);
if (callsThisHour > 1000) {
    $Errors.badRequest("Hourly quota exceeded");
}
```

The `ttlSeconds` you pass is applied **only the first time the key is created** — calling `increment` again on the same key never resets or extends its expiry. This is what makes a fixed window work: pass the window length every time, and the window starts counting from the first call after the previous window expired, not from your most recent call. Both backends (`InMemoryCacheStore` via `ConcurrentHashMap.compute`, `RedisCacheStore` via a server-side Lua script) guarantee `increment` is atomic, so concurrent calls from different requests — or, with the `redis` backend, different instances — never lose an update.

## Error handling

All five methods can throw `CacheStoreException` (unchecked) on a backend failure — e.g. a Redis connection error, or `increment` called on a key whose existing value isn't a valid number:

```javascript
try {
    $Cache.increment("counter", 1, 60);
} catch (e) {
    $Logger.warn("cache increment failed: " + e.getMessage());
    // fall back to behaving as if the counter were unavailable
}
```

## If `$Cache` is missing

`$Cache` is injected only when `cache-service.enabled` is `true` in `bootstrap.json` — see [configuration/cache-service.md](../configuration/cache-service.md). Unlike some other optional bindings, there is **no built-in fallback**: if the block is absent or `enabled` is not `true`, `$Cache` is simply not in scope, and a script referencing it fails with a `ReferenceError`. Guard any code path that might run without the service configured, e.g. by checking `typeof $Cache !== "undefined"` before use.

## Next

- [Configuration: cache service](../configuration/cache-service.md) — `storeType`, Redis connection keys, backend guarantees.
- [Scripting bindings reference](scripting-bindings.md) — every `$` binding, including `$Cache`.
