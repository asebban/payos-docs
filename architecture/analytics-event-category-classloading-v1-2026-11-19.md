Created: 2026-11-19
Last updated: 2026-11-19
Version: v1

# Event-category SPI resolution and classloading, across kernel and TSS

This document explains a classloading subtlety uncovered while building `payos-analytics-queue` (a queue-backed `IAnalyticsRecorder` — see [developer/analytics-queue-guide-v2-2026-11-19.md](../developer/analytics-queue-guide-v2-2026-11-19.md)): PayOS actually has **two different families** of SPI resolution, which rely on classloaders in two different ways, and only one of the two families was, before this change, reliably wired up in standalone `payos-job-scheduler` (TSS). Nothing here is specific to Analytics — the same mechanics apply to every "single implementation override" event category (`IAnalyticsRecorder`, `IEventStore`, `IMetricsRecorder`, `IIntegrationEventPublisher`, `IDiagnosticsRecorder`) described in [event-category-payload-contracts-v8-2026-09-14.md](../developer/event-category-payload-contracts-v8-2026-09-14.md).

## Two SPI families

**Family A — single-implementation override, ambient context classloader.** Used by every event-category facade (`AnalyticsRecorder`, `Diagnostics`, and the equivalents for `IEventStore`/`IMetricsRecorder`/`IIntegrationEventPublisher`) via `ma.s2m.payos.spi.SingleImplementationResolver`. Resolution happens exactly once per JVM, lazily, the first time the facade class is touched, by calling the single-argument `java.util.ServiceLoader.load(Class)` overload — which uses `Thread.currentThread().getContextClassLoader()` at that exact moment, implicitly. There is no explicit classloader parameter anywhere in this family's call chain.

**Family B — type-keyed selection, explicit classloader.** Used by `AuditLoggers` (`IAuditLogger`, selected by `audit-trail.implementation.category`), `QueueClients`/`NotificationClients` (`IQueueClientFactory`/`INotificationServiceFactory`, selected by a configured `type`), and `SecretProviders` (`ISecretProvider`). These all call the two-argument `ServiceLoader.load(Class, ClassLoader)` overload, explicitly passing `ma.s2m.payos.config.PayOSConfig#getServiceAdapterClassLoader()` — a dedicated `URLClassLoader` built by `ServiceAdapterLoader` over every `.jar` in the configured `service-adapters-dir`, parented on whatever the context classloader was at the time it was built.

The distinction matters because Family A's resolution is **ambient** (it just reads whatever `getContextClassLoader()` happens to return right then, wherever that classloader came from) while Family B's is **explicit** (it always asks for the same, deliberately-constructed classloader, by name, regardless of ambient state). A jar sitting in `service-adapters-dir` is automatically visible to Family B, everywhere, always. It is visible to Family A only if, at the moment the relevant facade class first gets touched, the current thread's context classloader happens to be (or descend from) the `service-adapters-dir` classloader.

## Where the ambient classloader is actually promoted

**Kernel (`BootServer`):** `BootServer.main`'s startup sequence calls `Thread.currentThread().setContextClassLoader(PayOSConfig.getServiceAdapterClassLoader())` once ([`BootServer.java:221`](../../payos/src/main/java/ma/s2m/payos/BootServer.java)), with **no matching restore** — the promotion lasts for the rest of the main thread's life, and every thread pool created afterward (the HTTP transport's workers, embedded TSS's Quartz pool, ...) inherits that context classloader at its own creation time, by default JVM behavior. Since `$Analytics`/every other Family-A-backed `$`-binding is only ever touched later, during request/script handling, long after this promotion, a `payos-analytics-queue.jar` dropped into `service-adapters-dir` is reliably found here.

**TSS embedded mode (`TaskSchedulerRuntimeIntegration`):** runs inside the exact same JVM, started by the kernel's own `BootServer` sequence, after the promotion above — so it inherits it for free. No extra wiring was needed.

**TSS standalone mode (`StandaloneMain`), before this change:** standalone TSS does build its own service-adapter classloader (`TenantDatabaseRegistryLoader.initializeServiceAdapters(...)` calls `ServiceAdapterLoader.initialize(...)` with the same `service-adapters-dir` key, read from TSS's own PayOS config file) — but it only ever *uses* it in two narrow, **restored** windows:

1. `StandaloneJobRegistration.registerJobs(...)`: wraps Hibernate session-factory construction in `thread.setContextClassLoader(serviceAdapterClassLoader)` / `finally { thread.setContextClassLoader(previous) }`, because Hibernate resolves a mapping file that isn't on disk as a classpath resource through the thread context classloader.
2. `TenantDatabaseRegistryLoader.applyConfigLoaders(...)`: passes the classloader explicitly to `ServiceLoader.load(IConfigLoader.class, ...)` — Family B style, no ambient promotion involved at all.

Neither window is still active by the time a Quartz worker thread actually executes a job script and calls `$JobInsight`/`AnalyticsRecorder.logEvent(...)` — those threads are spun up later, inside `bootstrap.start()`, well after `registerJobs(...)` already restored the previous (plain application) classloader in its `finally` block. A queue-backed `IAnalyticsRecorder` dropped into `service-adapters-dir` would therefore never be found by standalone TSS: `AnalyticsRecorder`'s one-time resolution would happen lazily, on a Quartz worker thread, under the wrong (unpromoted) context classloader, and silently fall back to the kernel's bundled `Slf4jAnalyticsRecorder` default.

## The fix: force early resolution inside the existing promotion window

Two options were considered for closing this gap:

- **Permanently promote the context classloader in `StandaloneMain`, mirroring `BootServer`'s un-restored `setContextClassLoader` call.** Rejected: this would change the ambient classloader for the rest of the standalone TSS process's life, affecting every other piece of code in that process that happens to consult the context classloader for anything — a much larger blast radius than the one SPI lookup this was meant to fix, for a process whose own design principle is to stay a minimal, independently-deployable module.
- **Force `AnalyticsRecorder`'s one-time resolution to happen inside the existing, narrow, already-restored promotion window.** Chosen. `AnalyticsRecorder.INSTANCE` is a static `AtomicReference`, populated exactly once, the first time any code references the class (ordinary JVM class-initialization semantics) — not re-resolved on every `logEvent(...)` call. Forcing that first touch to happen while the "correct" classloader is briefly current is therefore sufficient for the resolved instance to then be reused, unconditionally, by every later caller on every later thread, regardless of what their own context classloader is.

The actual change, in `StandaloneJobRegistration.registerJobs(...)`, inside the already-existing `try` block:

```java
thread.setContextClassLoader(PayOSConfig.getServiceAdapterClassLoader());
try {
    multiTenantDatabaseService = TenantDatabaseRegistryLoader.createDatabaseService(tenantRegistry);
    platformDatabaseService = DatabaseServiceInitializer.initializePlatform(payosSettings);
    AnalyticsRecorder.getInstance(); // forces one-time SPI resolution now, under this classloader
} finally {
    thread.setContextClassLoader(previousContextClassLoader);
}
```

This generalizes, unchanged, to any other Family-A facade a future standalone-TSS feature might need resolved from a `service-adapters-dir` jar (`IEventStore`, `IMetricsRecorder`, `IIntegrationEventPublisher`, `IDiagnosticsRecorder`) — each would just need its own one-line `getInstance()`/equivalent warm-up call added to the same block, if and when standalone TSS actually needs to override one of them.

## A related, separate fix: `PayOSConfig.getQueueClient()` in standalone TSS

Orthogonal to the classloading issue above, but needed for the same end-to-end feature: `QueueAnalyticsRecorder` (Family A's `IAnalyticsRecorder` override) publishes through `PayOSConfig.getQueueClient()`, the same shared client `$Queue` uses — and standalone TSS, before this change, never called `PayOSConfig.setQueueClient(...)` at all, even though it does connect its own, independent `IQueueClient` (via `IJobConfigSpec.Queue`/`QueueClientConnector`, deliberately never routed through the kernel's own `QueueClients` helper, to avoid coupling standalone TSS to `PayOSConfig`/the kernel for that connection). The fix is a single additive line in `StandaloneMain.runScheduler(...)`, right after that connection succeeds: `PayOSConfig.setQueueClient(queueClient)`. This makes `PayOSConfig.getQueueClient()` behave identically in embedded mode (where it already worked, since embedded mode shares the kernel's own call) and standalone mode, which is what `QueueAnalyticsRecorder` — and any other future `PayOSConfig.getQueueClient()`-based consumer — relies on.

## Summary table

| Deployment | `service-adapters-dir` visible to Family A? | `PayOSConfig.getQueueClient()` populated? |
| --- | --- | --- |
| Kernel (`BootServer`) | Yes — permanent promotion at boot | Yes — set by `QueueServiceInitializer` |
| TSS embedded | Yes — inherits the kernel's promotion | Yes — inherits the kernel's `PayOSConfig` |
| TSS standalone, before this change | No | No |
| TSS standalone, after this change | Yes — forced early resolution in `StandaloneJobRegistration` | Yes — set in `StandaloneMain` |

## Related references

- [Analytics queue guide](../developer/analytics-queue-guide-v2-2026-11-19.md) — the module these fixes enable end to end.
- [Configuration: `analytics-service`](../configuration/analytics-service-v3-2026-11-19.md).
- [Event category payload contracts](../developer/event-category-payload-contracts-v8-2026-09-14.md) — the governing principle behind every Family-A category.
- [Extensibility](./extensibility.md) — `service-adapters-dir`/`extensions-dir` in the broader plugin-discovery picture.
- [Task Scheduling System class diagram](./job-scheduler-class-diagram-v24-2026-11-15.md) §17 — the TSS-specific `queue`-trigger classloading mechanism (`QueueClientConnector`'s own `ADAPTERS_DIR`), a related but separate, Family-B-style mechanism.
