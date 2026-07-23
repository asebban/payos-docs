# Sliding window counter usage (`$SlidingWindow`)

When the [sliding window counter service](../configuration/sliding-window-service.md) is configured and enabled, scripts receive the `$SlidingWindow` binding — a `SlidingWindowBinding` that wraps the resolved `ISlidingWindowCounter` (`memory` or `redis`, depending on `sliding-window-service.storeType`). Unlike `$Cache.increment`, which implements a fixed/tumbling window that can let a burst straddle two windows and briefly exceed the intended limit, the underlying counter tracks an exact trailing window of events ending "now". This page covers usage from JavaScript; backend selection and Redis connection settings are in [configuration/sliding-window-service.md](../configuration/sliding-window-service.md).

## Read-only: no `recordAndCount`, no `reset`

`ISlidingWindowCounter` has three operations (`recordAndCount`, `count`, `reset`), but `SlidingWindowBinding` exposes only one of them to scripts: `count`. A script can inspect the current usage for a quota/rate-limit decision, but it cannot record a new event or wipe the counter — that is a deliberate restriction, not an oversight, so a script bug (or a malicious script) can't inflate its own quota headroom or reset another flow's counter.

```javascript
var callsThisHour = $SlidingWindow.count("calls", 3600000); // windowMillis, not seconds
```

If your use case needs the recording side too, it isn't reachable from scripts today — see [If you need to record events](#if-you-need-to-record-events) below.

## Tenant scoping is automatic

Like `$Cache`, `SlidingWindowBinding` captures the current request's tenant once and namespaces every key with it before calling the counter. A script never passes a tenant explicitly, and two tenants both calling `$SlidingWindow.count("calls", ...)` never read each other's counts:

```javascript
var callsThisHour = $SlidingWindow.count("calls", 3600000);   // reads "<tenantId>:calls"
```

## Checking a quota

`windowMillis` is milliseconds, not seconds (unlike `$Cache.put`/`increment`'s `ttlSeconds`) — a common mistake is reusing a seconds value here:

```javascript
function execute(request, controlData) {
    var oneHourInMillis = 60 * 60 * 1000;
    var callsThisHour = $SlidingWindow.count("api-calls", oneHourInMillis);

    if (callsThisHour >= 1000) {
        $Errors.badRequest("Hourly quota exceeded");
    }

    // ... proceed with the request
}
```

`count` never records anything — calling it repeatedly with the same key and window does not change what it returns (aside from events aging out of the window between calls, or being recorded by whatever Java-side code is doing the recording).

## Error handling

`count` can throw `SlidingWindowCounterException` (unchecked) on a backend failure — e.g. a Redis connection error:

```javascript
try {
    var callsThisHour = $SlidingWindow.count("api-calls", 3600000);
} catch (e) {
    $Logger.warn("sliding window count failed: " + e.getMessage());
    // decide how to behave when the quota check itself is unavailable
}
```

## If `$SlidingWindow` is missing

`$SlidingWindow` is injected only when `sliding-window-service.enabled` is `true` in `bootstrap.json` — see [configuration/sliding-window-service.md](../configuration/sliding-window-service.md). There is **no built-in fallback**: if the block is absent or `enabled` is not `true`, `$SlidingWindow` is simply not in scope, and a script referencing it fails with a `ReferenceError`. Guard any code path that might run without the service configured, e.g. by checking `typeof $SlidingWindow !== "undefined"` before use.

## If you need to record events

Nothing in the current script bindings calls `recordAndCount` — a script can read a counter (`$SlidingWindow.count`) but nothing records the events it's counting. Today, recording an event is only possible from Java code holding a reference to the resolved `ISlidingWindowCounter` (e.g. `PayOSConfig.getSlidingWindowCounter().recordAndCount(...)` from a future hook or middleware). If your use case needs a script to both record and check a quota in one call, that would require a separate, explicitly-write-capable binding — not something `$SlidingWindow` provides by design.

## Next

- [Configuration: sliding window counter service](../configuration/sliding-window-service.md) — `storeType`, Redis connection keys, backend guarantees.
- [Cache usage](cache-usage.md) — `$Cache.increment`, the fixed-window alternative, and where it's still the right tool (e.g. when a script needs to both record and read in one call).
- [Tenant quota enforcement](tenant-quota-enforcement.md) — the platform's own built-in consumer of `ISlidingWindowCounter`, and why it's a different counter from `$SlidingWindow`.
- [Scripting bindings reference](scripting-bindings.md) — every `$` binding, including `$SlidingWindow`.
