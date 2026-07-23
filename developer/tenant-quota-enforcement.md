# Tenant quota enforcement

PayOS can reject a tenant's requests once it exceeds a configured rate — `multitenancy.tenants[].quotas.requestsPerMinute` (or the `default-tenant-quotas` fallback), enforced by `TenantPolicyService` before your application's script even runs. This page explains what you observe as an application developer and how the counting mechanism behind it works; configuring the quota values themselves is covered in [configuration/multi-tenancy.md](../configuration/multi-tenancy.md).

## What you see when a tenant is over quota

A request that exceeds the tenant's `requestsPerMinute` limit is rejected with `429 Too Many Requests` — before `loadControlData`/`execute` runs, so your script never sees the request at all:

```
HTTP/1.1 429 Too Many Requests
```

There's nothing to catch or handle in your script for this — it's enforced entirely in the platform's request pipeline, in `TenantPolicyService.enforceAndOpenScope`, ahead of the scripting engine being set up.

## The counting mechanism you're behind (and why it changed)

Historically, the quota counter was a simple in-JVM per-minute bucket (`RateWindow`): a fixed one-minute window that resets abruptly at each minute boundary. That has a known weakness — a burst of traffic split across a boundary (e.g. many requests at :59 and many more at :00) could momentarily let through close to double the configured limit.

Quota enforcement now prefers an exact sliding-window counter (`ISlidingWindowCounter`) when one is available, and only falls back to the old fixed-window bucket if it isn't:

1. If [`sliding-window-service`](../configuration/sliding-window-service.md) is configured, the exact sliding window is used, and (with `storeType: "redis"`) the quota is enforced **globally across every instance** of the deployment — not per-instance.
2. If it isn't configured but the `sliding-window-counter-memory` module happens to be on the classpath (true by default), the same exact sliding-window algorithm is still used, just JVM-local (each instance counts its own traffic).
3. Only if neither is available does it fall back to the legacy `RateWindow` bucket.

You don't select any of this — it's resolved automatically at startup. In practice, almost every deployment today lands on tier 1 or 2. The only thing you can actually configure is whether tier 1 applies: enable `sliding-window-service` (ideally with `storeType: "redis"`) if you need the quota to be correct across a multi-instance deployment, rather than approximately-per-instance.

## This is not the same counter you can read from a script

The tenant quota counter and the `$SlidingWindow` script binding (see [sliding-window-usage.md](sliding-window-usage.md)) can share the same underlying `ISlidingWindowCounter` instance when `sliding-window-service` is configured — but they use different keys, so you can't read your own tenant's quota usage through `$SlidingWindow`:

- `TenantPolicyService` records under the bare tenant id as the key (e.g. `"acme"`), with a fixed 60-second window.
- `$SlidingWindow.count(key, windowMillis)` always prefixes the key with the tenant id and a colon (e.g. `$SlidingWindow.count("calls", ...)` reads `"acme:calls"`), so it never reads or writes the exact key the quota mechanism uses.

If you need a script to see how close a tenant is to some limit, track that yourself with your own key (e.g. `$Cache.increment("calls", 1, 3600)` or `$SlidingWindow.count("calls", ...)` alongside your own recording) — you cannot introspect the platform's own tenant-quota counter from a script.

## Next

- [Configuration: multi-tenancy](../configuration/multi-tenancy.md#how-quotas-are-enforced-backend-selection) — configuring `requestsPerMinute`, per-tenant overrides, and the backend-selection logic in more detail.
- [Configuration: sliding window counter service](../configuration/sliding-window-service.md) — enabling the `redis` backend for a quota that's correct across instances.
- [Sliding window counter usage](sliding-window-usage.md) — using `$SlidingWindow` for your own quota/rate-limit logic from scripts (a separate counter from the platform's tenant quota).
