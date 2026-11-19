Created: 2026-11-19
Last updated: 2026-11-19
Version: v3

# `analytics-service` configuration

Configures the publish destination for `payos-analytics-queue`, the queue-backed `IAnalyticsRecorder` implementation for the Analytics event category. See [developer/analytics-queue-guide-v2-2026-11-19.md](../developer/analytics-queue-guide-v2-2026-11-19.md) for the full developer guide, including how this module is installed and why it needs no broker connection of its own.

## Shape

Every Analytics implementation is configured the same way, regardless of which one is active: a `type` discriminator naming the implementation, and a `parameters` object whose keys are specific to that implementation. `payos-analytics-queue` is the `queue` type.

```json
{
  "analytics-service": {
    "type": "queue",
    "parameters": {
      "destination": "payos.analytics"
    }
  }
}
```

## Keys

| Key | Type | Default | Purpose |
| --- | --- | --- | --- |
| `type` | String | — | Which `IAnalyticsRecorder` implementation this block configures. `queue` selects `payos-analytics-queue`. |
| `parameters.destination` | String | `payos.analytics` | `queue`-type-specific. The queue destination/subject every `AnalyticsEvent` is published to. |

The whole `analytics-service` block is optional — if it is absent, or `parameters.destination` is blank, the default `payos.analytics` is used. These keys are declared once, shared across the codebase, as `IConfigSpec.AnalyticsService` (`TYPE`, `PARAMETERS`) and `IConfigSpec.AnalyticsService.Queue` (`TYPE`, `DESTINATION`, `DEFAULT_DESTINATION`) in `payos-foundation` — `payos-analytics-queue`'s `QueueAnalyticsRecorder` reads through these constants rather than its own local copies.

## Why there is (almost) nothing broker-specific to configure here

`payos-analytics-queue` deliberately opens no broker connection of its own. It publishes through `ma.s2m.payos.config.PayOSConfig#getQueueClient()`, the same already-connected `IQueueClient` the `$Queue` scripting binding uses — wired up entirely from the existing [`queue-service`](queue-service.md) block (in the kernel) or the TSS-specific queue configuration (standalone TSS — see [task-scheduling-system-v11-2026-11-15.md](task-scheduling-system-v11-2026-11-15.md)). `type`/`parameters` here only ever name the Analytics implementation and its own keys, never the broker: changing the broker is entirely a matter of changing `queue-service.configuration.type` (or TSS's own queue block), with zero changes needed here.

## Related references

- [Analytics queue guide](../developer/analytics-queue-guide-v2-2026-11-19.md) — full developer guide.
- [`queue-service`](queue-service.md) — the broker connection this module reuses, in the kernel.
- [`task-scheduling-system`](task-scheduling-system-v11-2026-11-15.md) — TSS's own, independent queue connection, used the same way in standalone/embedded TSS.
- [Architecture: event-category classloading across kernel and TSS](../architecture/analytics-event-category-classloading-v1-2026-11-19.md) — where to physically place `payos-analytics-queue.jar` so it is actually discovered via SPI in each deployment mode.
