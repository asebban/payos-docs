# Hot reload

PayOS applies configuration changes **without a restart**. A `ConfigWatcher` monitors the
configuration files and, on change, drives `BootServer.reloadConfiguration()` to atomically
swap the live settings and reinitialize services. This page explains what reloads, how it is
safe, and what to watch.

## How it works

```
ConfigWatcher (NIO WatchService over the config paths)
   CREATE / MODIFY / DELETE event
      → BootServer.reloadConfiguration()
           - reload + merge config (same pipeline as bootstrap)
           - atomically swap PayOSConfig.settings / applications / dataSources
           - reinitialize services (DB, queue, secrets, webhooks)
           - invalidate the hook cache
```

`PayOSConfig.settings` is a **volatile** map and the swap is atomic, so in-flight requests
either see the old or the new configuration consistently — never a half-applied state.

## Disabling hot reload

Hot reload is **enabled by default**. To disable it, set `config-hot-reload-enabled` to
`false` in `payos.json` or `bootstrap.json`:

```json
{
  "config-hot-reload-enabled": false
}
```

When disabled, the `ConfigWatcher` is not started and configuration changes require a full
restart.

### When to disable

- **High-security environments** where configuration changes must go through a controlled
  deployment pipeline with explicit restarts.
- **Production environments** where changes are managed exclusively via container orchestration
  or configuration management tools and automatic reload is not desired.
- **Debugging scenarios** where stable configuration is required to reproduce an issue.
- **Regulated environments** where all changes must be auditable through deployment records
  rather than runtime file modifications.

## What reloads

- **Settings** from `payos.json` (bundle entrypoint) and the merged runtime configuration files
  under `configDir` (typically `config/bootstrap.json`).
- **Applications** registry (`applications`).
- **Data sources** and service initializations.
- **Hooks** — the hook cache is invalidated so new/changed hooks take effect.

## Safe service reinitialization

When the database configuration changes, the previous Hibernate `SessionFactory` is **retired
gracefully**: it is closed after
`database-service.retired-session-factory-close-delay-seconds` to let in-flight work drain.
This avoids tearing connections out from under active requests. See
[configuration/database-service.md](../configuration/database-service.md).

## What to expect operationally

- Edits to watched files are picked up automatically — no `/stop` + relaunch needed for
  configuration changes.
- The watcher logs reload events; monitor these to confirm a change was applied.
- Avoid assumptions of static, one-time configuration in custom code — values can change at
  runtime. (Project convention: do not introduce static one-time config assumptions.)

## When a restart is still required

- Replacing **connector/extension JARs** on the classpath (the classloaders are built at
  startup).
- Changing the set of **server listeners** in ways that require rebinding ports.

For these, perform a controlled restart (e.g. `/stop` then relaunch, or your service
manager). See [deployment.md](deployment.md).

## Next

- [deployment.md](deployment.md)
- [observability.md](observability.md)
- [architecture/runtime-architecture.md](../architecture/runtime-architecture.md)
