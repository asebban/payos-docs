Created: 2026-10-22
Last updated: 2026-10-22
Version: 1

# `tenant-config-loader-database` architecture

How `tenant-config-loader-database` — an optional, independently-deployable module — declares `multitenancy.tenants` from a database table, without the kernel ever taking a compile-time dependency on it. Configuration keys are in [configuration/tenant-config-loader-database.md](../configuration/tenant-config-loader-database-v1-2026-10-22.md); developer usage (worked example, full column mapping) is in [developer/tenant-config-loader-database.md](../developer/tenant-config-loader-database-v1-2026-10-22.md).

## The `IConfigLoader` SPI

`IConfigLoader` (`payos-foundation`) is the generic extension point this module implements:

```java
public interface IConfigLoader {
    Map<String, Object> loadConfig(Map<String, Object> currentConfig) throws Exception;
}
```

Same discovery mechanism as `IDatabaseServiceFactory`/`IQueueClientFactory`/`IRuntimeIntegration` — `ServiceLoader` against `PayOSConfig.getServiceAdapterClassLoader()`, so the kernel never depends on an implementation; only the implementing module depends on the kernel/foundation. Zero providers found is a fully-supported "no extra config sources" case, not an error. Every discovered loader's result is a tree shaped exactly like a hand-written `payos.json`/config-directory JSON file (e.g. a `multitenancy.tenants.<tenantId>` entry) — never a flat/prefixed key space — and is **deep-merged on top of the configuration already accumulated from JSON, with the loader's keys winning on conflict**.

## Where this runs in boot

```mermaid
sequenceDiagram
    participant Boot as BootServer.main()
    participant CL as ConfigLoader.loadServerConfig()
    participant SAL as ServiceAdapterLoader
    participant SPI as ServiceLoader&lt;IConfigLoader&gt;
    participant TCL as TenantConfigLoader

    Boot->>CL: load payos.json + config directory JSON
    CL->>CL: accumulate JSON-derived config
    CL->>SAL: ServiceAdapterLoader.initialize(settings)
    Note over SAL: builds the service-adapters-dir<br/>classloader (so TenantConfigLoader's<br/>own jar is on it)
    CL->>SPI: ServiceLoader.load(IConfigLoader.class, serviceAdapterClassLoader)
    SPI->>TCL: loadConfig(currentConfig)
    TCL->>TCL: open short-lived JDBC connection<br/>(reuses platform-database-service's<br/>own connection details)
    TCL->>TCL: SELECT * FROM &lt;table&gt;,<br/>build multitenancy.tenants.* per row
    TCL-->>SPI: contributed config tree
    SPI-->>CL: ConfigMerger.deepMerge(config, contributed)
    CL->>CL: re-normalize runtime-relative paths<br/>(idempotent on already-resolved values)
    CL-->>Boot: final merged config
```

The `IConfigLoader` pass runs **after** `ServiceAdapterLoader.initialize(...)`, never before — the loader's own jar has to be on the classloader `ServiceLoader` searches. Paths/JDBC URLs are re-normalized a second time after the merge, since a loader's contribution can introduce new relative paths; `normalizeRuntimeRelativePaths` is idempotent on values already resolved, so this second pass only affects what the loader just contributed.

## Two independent pieces

`TenantConfigLoader` (the `IConfigLoader` above) is a **one-shot read** — it has no notion of "live" by itself. It runs once per config load: at boot, or whenever something else already triggers a hot-reload (e.g. editing a watched `.json` file, which re-runs the whole `ConfigLoader.loadServerConfig()` pipeline, which re-invokes every discovered `IConfigLoader` again, this one included).

`TenantRegistryChangePoller` is a **separate, independently opt-in** `IRuntimeIntegration` — discovered via the exact same `service-adapters-dir` classloader, but through `BootServer`'s own `ServiceLoader<IRuntimeIntegration>` loop rather than the `IConfigLoader` pass. It starts a background thread that polls `SELECT MAX(<changeDetectionColumn>) FROM <table>` on an interval; when the value changes, it rewrites a small, dedicated sentinel file (`tenant-config-loader-poll-trigger.json`, otherwise inert) inside the bundle's watched config directory. That write is all it takes — `ConfigWatcher` already watches that directory for any `.json` file change and, on a match, re-runs the full `loadServerConfig()` pipeline. The poller does not invent a second, parallel reload path; it only has to produce one filesystem event PayOS's existing hot-reload machinery already reacts to.

Both pieces call into `TenantRegistrySupport` (package-private, shared) for the registry JDBC connection and generic config-map helpers, so connection resolution exists in exactly one place regardless of which of the two triggered the read.

### Why a dedicated, short-lived connection instead of `$PlatformDB`'s pool

`TenantConfigLoader` runs during the configuration-loading phase, which completes before `DatabaseServiceInitializer.initializePlatform(...)` ever builds `$PlatformDB`'s own Hikari-backed runtime pool. There is no pool to reuse yet at this point in boot — the loader instead reads `platform-database-service.configuration`'s raw connection fields (`url`/`username`/`password`/`driver_class`) directly off the config map and opens a plain `DriverManager.getConnection(...)` for the duration of one query, then closes it. This is why `platform-database-service` must be *configured* (the connection details must exist in the merged config) even though the loader never actually touches `$PlatformDB`'s own pool.

## Fatal vs. non-fatal

Unlike `payos-database-audit-trail` (which throws `FatalException` and stops boot entirely if `payos_audit_event` can't be guaranteed — see [atomicity-db-audit-queue-v10-2026-10-20.md §5.2](./atomicity-db-audit-queue-v10-2026-10-20.md#52-fatal-vs-non-fatal-by-explicit-choice)), a failing `IConfigLoader` is **never fatal to boot** — `ConfigLoader.loadServerConfig()` logs the exception and continues without that loader's contribution, the same "optional subsystem" policy `IRuntimeIntegration#start` already follows. A misconfigured or unreachable tenant registry does not take down the rest of PayOS; it just means the tenants that would have come from that table are simply absent, exactly as if the module weren't installed at all. The same policy applies to `TenantRegistryChangePoller`: a failed poll is logged at `WARN` and retried on the next interval, never fatal, never stops the poller thread.

## Module/class reference

| Concern | Class | Module |
| --- | --- | --- |
| SPI contract | `ma.s2m.payos.config.IConfigLoader` | `payos-foundation` |
| Discovery + deep-merge | `ma.s2m.payos.config.ConfigLoader#loadServerConfig` | `payos` (kernel) |
| One-shot `multitenancy.tenants` contribution | `ma.s2m.payos.config.loader.database.TenantConfigLoader` | `tenant-config-loader-database` |
| Config key spec | `ma.s2m.payos.config.loader.database.ITenantConfigLoaderSpec` | `tenant-config-loader-database` |
| Shared connection/helpers | `ma.s2m.payos.config.loader.database.TenantRegistrySupport` (package-private) | `tenant-config-loader-database` |
| Live-reload trigger | `ma.s2m.payos.config.loader.database.TenantRegistryChangePoller` | `tenant-config-loader-database` |

## Next

- [Configuration: tenant-config-loader-database](../configuration/tenant-config-loader-database-v1-2026-10-22.md) — the full key reference.
- [Developer: tenant-config-loader-database](../developer/tenant-config-loader-database-v1-2026-10-22.md) — worked example, column mapping, live-reload caveats.
- [Runtime architecture](runtime-architecture.md) — the broader bootstrap/config-loading/hot-reload picture this fits into.
- [Multi-tenancy](multi-tenancy.md) — the `multitenancy.tenants` shape this loader contributes into.
