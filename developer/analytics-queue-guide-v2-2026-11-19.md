Created: 2026-11-19
Last updated: 2026-11-19
Version: v2

# Analytics queue guide: `payos-analytics-queue`

This is the end-to-end developer guide for `payos-analytics-queue`, a queue-backed `IAnalyticsRecorder` implementation for the Analytics event category (see [event-category-payload-contracts-v8-2026-09-14.md §2](./event-category-payload-contracts-v8-2026-09-14.md#2-analytics--analyticsevent) for the `AnalyticsEvent` payload contract this module carries unchanged). It replaces the kernel's own default, `Slf4jAnalyticsRecorder` (which only logs), with one that actually publishes every event onto a message broker, and it works identically whether the recorder is reached from the kernel's own `$Analytics` binding or from `payos-job-scheduler`'s (TSS's) `$JobInsight` binding, in every TSS deployment mode.

## Why a separate repository

PayOS's "one dedicated interface+facade per event category" principle (see the governing-principle section of the payload-contracts doc linked above) means the Analytics category ships with only a minimal default implementation in the kernel, and any real delivery mechanism (batching, HTTP shipping, queue publishing, ...) is expected to live in its own module, discovered via `ServiceLoader` — exactly the same shape `payos-database-audit-trail`/`payos-buffered-audit-trail` already use for `IAuditLogger`. `payos-analytics-queue` is that module for the queue-publishing case.

## What it does

The module ships exactly one production class, `ma.s2m.payos.analytics.queue.QueueAnalyticsRecorder`, implementing `ma.s2m.payos.events.analytics.IAnalyticsRecorder` (package `ma.s2m.payos.events.analytics` in `payos-foundation` — note the `event-category-payload-contracts-v8-2026-09-14.md` §2 table still states the older `ma.s2m.payos.analytics` package name for both the interface and `Slf4jAnalyticsRecorder`; that was never updated when the classes moved under `ma.s2m.payos.events.analytics`, so treat this guide, not that table, as authoritative on package names).

On every `logEvent(AnalyticsEvent event)` call:

1. It fetches `ma.s2m.payos.config.PayOSConfig.getQueueClient()` — the exact same, already-connected `IQueueClient` instance the `$Queue` scripting binding publishes through, wired up from the existing `queue-service.configuration` block. The module opens **no broker connection of its own** and has no broker-specific code anywhere in it: whichever `IQueueClient` provider `queue-service.configuration.type` selects (NATS today, or any other provider later) is what this module publishes through, with zero code changes here.
2. If no queue client is configured (`PayOSConfig.getQueueClient()` returns `null`), it logs a warning and drops the event. This is deliberate, not a bug: per the `AnalyticsEvent` contract, this category "tolerates sampling and best-effort delivery" — it must never throw back into the caller (a script's `$Analytics` call, or TSS's `$JobInsight`).
3. Otherwise it serializes `event` via `AnalyticsEvent#toJson()` (unchanged — this module never touches the payload shape), wraps it in a `QueueMessage` with `{tenantId, name}` metadata, and calls `queueClient.publish(destination, message)`, where `destination` comes from the `analytics-service.parameters.destination` configuration key (default `payos.analytics`, declared as `IConfigSpec.AnalyticsService.Queue.DESTINATION`/`DEFAULT_DESTINATION`) — see [configuration/analytics-service-v3-2026-11-19.md](../configuration/analytics-service-v3-2026-11-19.md) for the full key reference.
4. Any exception from `publish(...)` (broker down, etc.) is caught and logged, never rethrown — same best-effort rule as step 2.

## Registering the override

The module registers itself via `META-INF/services/ma.s2m.payos.events.analytics.IAnalyticsRecorder`, pointing at `QueueAnalyticsRecorder`, resolved by `ma.s2m.payos.spi.SingleImplementationResolver` the same way every other event-category facade resolves its active implementation (first external `ServiceLoader`-discovered provider wins over the kernel's own bundled default — no kernel code change required).

One pre-existing kernel detail worth knowing about while you're here: the kernel's own bundled default registration file is named `META-INF/services/ma.s2m.payos.analytics.IAnalyticsRecorder` in `payos/src/main/resources`, which does **not** match the interface's real fully-qualified name (`ma.s2m.payos.events.analytics.IAnalyticsRecorder`). This is harmless for the kernel itself — `SingleImplementationResolver` falls back to its hardcoded `Slf4jAnalyticsRecorder` instance when `ServiceLoader` finds nothing, landing on the same class either way — but it means that filename is not a template to copy for an override. `payos-analytics-queue`'s own `META-INF/services` file uses the correct, real interface name, the same way `payos-buffered-audit-trail`/`payos-database-audit-trail` already do for `IAuditLogger`.

## Deploying it: three processes, three classloader stories

`AnalyticsRecorder`'s active instance is resolved exactly once per JVM, lazily, the first time anything touches the `AnalyticsRecorder` class — at that moment, `SingleImplementationResolver` calls the single-argument `ServiceLoader.load(IAnalyticsRecorder.class)`, which uses **whatever classloader is the current thread's context classloader at that instant**, not `PayOSConfig.getServiceAdapterClassLoader()` (a different, explicitly-passed classloader a handful of other SPI lookups use — see [architecture/analytics-event-category-classloading-v1-2026-11-19.md](../architecture/analytics-event-category-classloading-v1-2026-11-19.md) for the full mechanics and why the two differ). The practical consequence is that *where* you drop `payos-analytics-queue.jar` so it is actually found depends on which process is resolving `IAnalyticsRecorder`:

- **Kernel (`BootServer`/`payos-runtime`):** drop the jar into the configured `service-adapters-dir`. `BootServer` promotes that directory's classloader to be the main thread's context classloader early and permanently during boot, before `$Analytics` is ever touched, so this works with zero extra wiring.
- **TSS embedded mode** (`TaskSchedulerRuntimeIntegration`, running inside the same JVM as `payos-runtime`): same classloader promotion as the kernel, since it's the same process — no extra wiring either.
- **TSS standalone mode** (`StandaloneMain`, its own separate deployable/process): also works, but only because of a dedicated fix — see the next section. Without it, the jar would silently never be found there.

In all three cases, install the jar by dropping it into whichever `service-adapters-dir` that process's own `payos.json`/bootstrap configuration points at (standalone TSS reads the exact same key, from its own PayOS config file — see [configuration/task-scheduling-system-v11-2026-11-15.md](../configuration/task-scheduling-system-v11-2026-11-15.md)).

## The standalone-TSS fix, in detail

Two small, additive changes to `payos-job-scheduler` (the TSS module) make standalone mode behave like the kernel and like embedded mode for this feature, without permanently changing TSS's own ambient context classloader (which would have a much larger blast radius on everything else TSS loads):

1. **`StandaloneMain.runScheduler(...)`** now calls `PayOSConfig.setQueueClient(queueClient)` right after TSS connects its own `IQueueClient` (the one `IJobConfigSpec.Queue`/`task-scheduler.queue` configures). Standalone mode never exposed this client through `PayOSConfig` before — `PayOSConfig.getQueueClient()` always returned `null` there, so `QueueAnalyticsRecorder` would have silently dropped every event, warning-logged, forever, in that mode. Embedded mode never needed this fix because it shares the kernel's own `PayOSConfig.setQueueClient(...)` call.
2. **`StandaloneJobRegistration.registerJobs(...)`** now calls `ma.s2m.payos.events.analytics.AnalyticsRecorder.getInstance()` once, inside the `try` block that already, briefly, promotes the thread's context classloader to `PayOSConfig.getServiceAdapterClassLoader()` for Hibernate mapping resolution (restored in the matching `finally`). `AnalyticsRecorder`'s active instance is cached in a static field after its first resolution, so forcing that one resolution to happen inside this already-existing, narrow promotion window is enough — every later call to `AnalyticsRecorder.logEvent(...)`, from any Quartz worker thread, for the rest of the process's life, reuses the cached instance, regardless of that thread's own context classloader. Without this call, the class would instead first get touched lazily, on whichever Quartz worker thread first executes a script calling `$JobInsight`/`$Analytics` — a thread created later, by `bootstrap.start()`, which never inherits the promoted classloader, so the override would never be found.

Both changes are additive and were verified against the full `payos-job-scheduler` test suite (325 tests, no regressions) plus the new module's own 5-test suite.

## Configuration

See [configuration/analytics-service-v3-2026-11-19.md](../configuration/analytics-service-v3-2026-11-19.md) for the full key reference. In short: nothing new to configure for the broker connection itself (it reuses `queue-service.configuration` in the kernel, or the TSS-specific queue block in TSS — see above); the Analytics-specific keys are `analytics-service.type` (`queue`, selecting this module) and the optional `analytics-service.parameters.destination` string (default `payos.analytics`).

## Testing

`QueueAnalyticsRecorderTest` (in the module's own `src/test`) covers: publishing to the default destination, publishing to a configured destination, never throwing when no queue client is configured, never throwing when the broker publish itself fails, and ignoring a `null` event — using a hand-written `FakeQueueClient`/`FailingQueueClient` rather than a real broker, consistent with this module having zero broker-specific code to exercise.

## Related references

- [Event category payload contracts](./event-category-payload-contracts-v8-2026-09-14.md) — the `AnalyticsEvent` shape and the governing "one abstraction per event category" principle this module follows.
- [Configuration: `analytics-service`](../configuration/analytics-service-v3-2026-11-19.md) — the configuration key reference.
- [Architecture: event-category classloading across kernel and TSS](../architecture/analytics-event-category-classloading-v1-2026-11-19.md) — the full classloader mechanics behind the "deploying it" section above.
- [Configuration: `queue-service`](../configuration/queue-service.md) — the broker connection this module reuses.
- [Task Scheduling System: developer guide](./task-scheduler-guide-v21-2026-11-19.md) — `$JobInsight`, which already calls `AnalyticsRecorder.logEvent(...)` the same way `$Analytics` does.
