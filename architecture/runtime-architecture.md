# Runtime architecture

How a PayOS process **boots**, **loads and merges configuration**, **initializes services**,
and **stays current** through hot-reload. The bootstrap layer is owned by the kernel module
`payos` (`ma.s2m.payos`).

## Entry point: `BootServer`

`ma.s2m.payos.BootServer` is the executable main class (declared in the `payos-runtime`
shade plugin manifest). It accepts a single argument:

```bash
java -jar payos-runtime-<version>.jar --bundle-path <bundle-directory|payos.json>
```

If `--bundle-path` is omitted, the configuration is resolved from the current directory or
the classpath. The bundle directory rooted at `payos.json` is the unit operators deploy
(see [operations/deployment.md](../operations/deployment.md)).

## Boot sequence

```
BootServer.main(args)
  │
  ├─ ConfigLoader.loadServerConfig([payos.json])
  │     ├─ read raw bytes
  │     ├─ CryptoService.decryptIfEncrypted(raw)      ← supports encrypted config
  │     ├─ EnvVarResolver.resolve(map)                ← ${ENV} substitution
  │     ├─ resolve relative paths against payos.json location
  │     └─ merge config directory files (loadConfigByDirectory)
  │
  ├─ store result in PayOSConfig.settings (volatile Map<String,Object>)
  │
  ├─ ConnectorLoader.initialize(settings)             ← connectors-dir → connector classloader
  ├─ ExtensionLoader.initialize(settings)             ← extensions-dir → extension classloader
  │
  ├─ initialize services (registered into PayOSConfig):
  │     ├─ DatabaseServiceInitializer  → setDatabaseService(...)
  │     ├─ QueueServiceInitializer     → setQueueClient(...)
  │     ├─ SecretServiceInitializer    → setSecretProvider(...)
  │     ├─ WebhookServiceInitializer   → setWebhookDispatcher(...)
  │     └─ (idempotency, activation store, …)
  │
  ├─ for each entry in settings["servers"]:
  │     Servers.start(host, port, protocol, serverConfig)   ← ServiceLoader<ServerProvider>
  │
  └─ ConfigWatcher(directories).start()               ← hot-reload
```

Startup failures are surfaced as explicit checked exceptions (for example `UnableToStartServerException`); they are **not** silently swallowed.

## Configuration loading & merging

`ConfigLoader` produces the effective configuration in this order:

1. **Bundle entrypoint** — `payos.json` (constant `IConfigSpec.RUNTIME_CONFIG_FILE`), usually containing `configDir`.
2. **Decryption** — if the file is encrypted, `CryptoService.getInstance().decryptIfEncrypted(...)`    transparently decrypts it. This lets operators ship encrypted config (see    [operations/bundle-encryption.md](../operations/bundle-encryption.md)).
3. **Environment substitution** — `EnvVarResolver.resolve(...)` replaces environment references so secrets and host-specific values stay out of the file.
4. **Config directory merge** — JSON files under the configured config directory (`IConfigSpec.CONFIG_DIRECTORY = "configDir"`), typically including `bootstrap.json`, are merged into the effective runtime configuration.
5. **Path resolution** — `ConfigLoader` computes `IConfigSpec.RUNTIME_BASE_DIRECTORY` from the directory that contains `payos.json`, stores it as the effective `runtimeBaseDir`, and resolves relative paths from that base.

The full key catalog is in the [configuration reference](../configuration/README.md) and the exhaustive index in [reference/configuration-keys.md](../reference/configuration-keys.md).

## The global registry: `PayOSConfig`

`ma.s2m.payos.config.PayOSConfig` is the process-wide registry. It holds configuration and
the singleton services that the request pipeline and scripts depend on.

| Member | Type | Purpose |
| --- | --- | --- |
| `settings` | `volatile Map<String,Object>` | Effective bootstrap configuration. |
| `applications` | `volatile Map<String,Application>` | Cache of loaded applications. |
| `dataSources` | `Map<String,DataSource>` | Database sources per application. |
| `setDatabaseService` / `getDatabaseService` | `IDatabaseService` | The `$DB` provider. |
| `setQueueClient` / `getQueueClient` | `IQueueClient` | The `$Queue` provider. |
| `setSecretProvider` / `getSecretProvider` | `ISecretProvider` | The `$Secrets` provider. |
| `setWebhookDispatcher` / `getWebhookDispatcher` | `IWebhookDispatcher` | The webhook dispatcher. |
| `setActivationStore` / `getActivationStore` | `IActivationStore` | Capability activation state. |
| `setIdempotencyService` / `getIdempotencyService` | `IdempotencyService` | Idempotent response cache. |
| `setConnectorClassLoader` / `getConnectorClassLoader` | `ClassLoader` | Loads connector SPI JARs. |
| `setExtensionClassLoader` / `getExtensionClassLoader` | `ClassLoader` | Loads extension JARs for `Java.type()`. |
| `setRuntimeBaseDirectory` / `getRuntimeBaseDirectory` | `String` | Effective bundle base path for relative resolution. |

Services are registered **once** at bootstrap (and again on reload). The `volatile`
references mean readers always see the latest map without locking.

## Hot-reload: `ConfigWatcher`

`ma.s2m.payos.config.ConfigWatcher` watches the configuration directories using the Java NIO
`WatchService` and reacts to `CREATE`, `MODIFY`, and `DELETE` events.

```
ConfigWatcher(Collection<Path> directories)
  .addListener(ConfigChangeListener)   ← BootServer registers reloadConfiguration()
  .start()
```

On a change, `BootServer.reloadConfiguration()`:

1. reloads and re-merges the configuration,
2. **atomically swaps** the `settings`/`applications` maps (volatile assignment),
3. reinitializes services as needed, and
4. invalidates caches (for example the hook cache).

This is why the kernel rule *"avoid static one-time config assumptions"* exists: any code
that reads configuration must read it through `PayOSConfig.settings` so that a reload takes
effect. Operational behavior is described in [operations/hot-reload.md](../operations/hot-reload.md).

## Class-loader hierarchy

```
application classloader
        ▲ parent
extension classloader   (extensions-dir, used by Java.type())
        ▲ parent
connector classloader   (connectors-dir, used by ServiceLoader SPI)
        ▲ parent
runtime/app classloader (the shaded payos-runtime JAR)
```

`ConnectorLoader` and `ExtensionLoader` each build a `URLClassLoader` over the JARs in their
directory and register it in `PayOSConfig`. The directories are resolved in this precedence
order:

1. JVM system property (`-Dconnectors-dir=…` / `-Dextensions-dir=…`),
2. environment variable (`PAYOS_CONNECTORS_DIR` / `PAYOS_EXTENSIONS_DIR`),
3. the bootstrap settings key (`connectors-dir` / `extensions-dir`).

See [extensibility.md](extensibility.md) for what each loader enables.

## Next

- [Request processing](request-processing.md) — what happens after the servers are up.
- [Configuration reference](../configuration/README.md) — every key `ConfigLoader` understands.
