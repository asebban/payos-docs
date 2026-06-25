# Extensibility

PayOS adds behavior **without modifying the rigid core**. There are seven distinct extension mechanisms. Knowing which is which — and how each is discovered — is essential.

## The seven mechanisms at a glance

| Mechanism | Directory | Discovery | Used for |
| --- | --- | --- | --- |
| **Capabilities** | application `base.path` | Installed via `cpm`, declared with `"category": "capability"` | Reusable, self-contained extensions (APIs, pages, menus, libraries, hooks) that applications can inherit from; activatable without redeploying the runtime; enables composable application architectures |
| **Application resource inheritance** | application `base.path` | `extends` field in application config | Applications can inherit APIs, pages, menus, libraries, and hooks from capabilities or from other applications. They can override behavior of inherited application without any impact on it |
| **Internal hooks** | application `hook/` directory | Registered in `hooks` configuration | JavaScript scripts that intercept lifecycle events (API_BEFORE_EXECUTE, API_AFTER_EXECUTE, API_ON_ERROR, etc.) in the request processing pipeline; observe and modify request/response context. |
| **Service connectors** | `connectors-dir` | Java `ServiceLoader` (SPI) | Database, queue, secret, and webhook providers. |
| **Java extensions** | `extensions-dir` | `Java.type('…')` from scripts | Arbitrary Java libraries callable from JavaScript (e.g. jPOS). |
| **Transport providers** | bundled / classpath | `ServiceLoader<ServerProvider>` | New protocols (`http`, `tcp`, `queue`, …). |
| **TCP codec/handler plugins** | `tcp-handlers-dir` | JAR scanning for concrete `TcpMessageDecoder/Encoder/Handler` | Custom wire-format parsing for the TCP transport; decodes bytes to `Request`, encodes `Response` to bytes, and provides message-specific processing logic. |

## 1. Capabilities (extending applications)

Capabilities extend the **guest layer**, not the core. A capability is an application with `category: "capability"` that other applications reference through `extends`. It can be installed and activated **without redeploying the runtime**.

- **Resolution gating** — `ResourceLocator` only resolves a capability's resources when
  `IActivationStore.isActive(capabilityId, requestingAppId, tenantId)` is true.
- **State** — installation and activation state live under `{configDir}/.capabilities/`
  (`registry.json`, `activation.json`, append-only `events.ndjson`).
- **Lifecycle** — install / uninstall / activate / deactivate run capability lifecycle
  hooks and the relevant [system events](eventing-webhooks.md).
- **Management** — performed with [`cpm`](../cli-tools/cpm.md).

## 2. Application resource inheritance

Applications can inherit resources (APIs, pages, menus, libraries, hooks) from capabilities or other applications via the `extends` field in their configuration:

```json
{
  "id": "my-app",
  "extends": ["core-capability", "auth-capability"],
  "apis": [...]
}
```

The `ResourceLocator` resolves resources in this order:

1. The application's own `base.path`
2. Each entry in `extends`, in declaration order
3. Built-in fallbacks (if any)

This enables composable application architectures where common functionality (authentication, menus, utility APIs) is defined once in a capability and reused across applications.

## 3. Internal hooks

Internal hooks are JavaScript files placed in an application's `hook/` directory. They intercept request processing lifecycle events:

```
app-base-path/
  hook/
    validate-request.js     ← registered for API_BEFORE_EXECUTE
    audit-response.js       ← registered for API_AFTER_EXECUTE
    handle-error.js         ← registered for API_ON_ERROR
```

Hooks are registered in the application's configuration:

```json
{
  "hooks": [
    {
      "name": "validate-request",
      "event": "API_BEFORE_EXECUTE",
      "script": "hook/validate-request.js"
    }
  ]
}
```

Available lifecycle events include:
- `API_BEFORE_EXECUTE` — before API script execution
- `API_AFTER_EXECUTE` — after successful execution
- `API_ON_ERROR` — when an error occurs
- Capability lifecycle events (install, uninstall, activate, deactivate)

Hooks have access to the request/response context and can modify it before the next pipeline stage. See [eventing-webhooks.md](eventing-webhooks.md) for the complete event list.

## 4. Service connectors (SPI)

A connector is a JAR in `connectors-dir`. `ConnectorLoader` builds a `URLClassLoader` over those JARs at bootstrap and registers it via `PayOSConfig. setConnectorClassLoader(...)`. Each service is then discovered with `ServiceLoader` against that classloader and selected by a `type` string.

| Service | SPI factory | `type()` examples | Script binding |
| --- | --- | --- | --- |
| Database | `IDatabaseServiceFactory` | (kernel-managed, no type) | `$DB` |
| Queue | `IQueueClientFactory` | `nats` | `$Queue` |
| Secrets | `ISecretProviderFactory` | `filesystem`, `vault` | `$Secrets` |
| Webhooks | `IWebhookDispatcherFactory` | `http` | `$WebHooks` (dispatch) |

### How a connector is selected

```
QueueServiceInitializer / SecretServiceInitializer / WebhookServiceInitializer
  └─ XxxClients.create(type, config)
        ├─ ServiceLoader<IXxxFactory>  (over the connector classloader)
        ├─ match factory whose type() equals the configured type (normalized to lowercase)
        └─ factory.create(config) → register in PayOSConfig
```

### Writing a connector

1. Implement the SPI factory (e.g. `ISecretProviderFactory`) and its product (`ISecretProvider`).
2. Return a unique `type()` (e.g. `"vault"`).
3. Register the factory in `META-INF/services/<fully-qualified-SPI-interface>`.
4. Build a JAR and drop it (with its dependencies) into `connectors-dir`.
5. Reference its `type` in the relevant configuration block.

The shipped connectors are the reference implementations: `queue-service-nats`, `webhook-service-http`, `secret-service-filesystem`, `secret-service-vault`, and `database-service`. See [configuration/extensions-connectors.md](../configuration/extensions-connectors.md) for directory resolution and [build-and-release/module-map.md](../build-and-release/module-map.md) for artifacts.

> **Kernel rule:** connector implementations live in standalone modules; only the SPI
> interface stays in the kernel.

## 5. Java extensions

An extension is **any** Java library JAR placed in `extensions-dir`. `ExtensionLoader` builds a `URLClassLoader` over those JARs and registers it as the **host class loader** of the scripting sandbox. Scripts then reach the classes via:

```javascript
const ISOMsg = Java.type('org.jpos.iso.ISOMsg');
const msg = new ISOMsg();
```

Key differences from connectors:

- An extension implements **no PayOS interface** — it is ordinary Java.
- It is invoked **from scripts** with `Java.type()`, not auto-discovered by the kernel.
- Access is still bounded by the [scripting sandbox](scripting-engine.md): blocked classes (e.g. `java.lang.System`) remain unreachable.

This is the mechanism for integrating libraries such as **jPOS** for ISO 8583 message handling. Developer guidance and examples: [developer/java-extensions.md](../developer/java-extensions.md).

### Directory resolution (both loaders)

`connectors-dir` and `extensions-dir` are each resolved in this precedence order:

1. JVM system property (`-Dconnectors-dir=…` / `-Dextensions-dir=…`),
2. environment variable (`PAYOS_CONNECTORS_DIR` / `PAYOS_EXTENSIONS_DIR`),
3. the bootstrap settings key (`connectors-dir` / `extensions-dir`).

## 6. Transport providers

A transport adds a protocol by implementing `ma.s2m.payos.servers.ServerProvider`:

```java
String  protocol();                                       // e.g. "tcp"
IServer create(Map<String,Object> serverConfig);          // build the server
```

`Servers` discovers providers through `ServiceLoader<ServerProvider>` and dispatches by protocol name. The shipped transports register themselves in `META-INF/services/ma.s2m.payos.servers.ServerProvider`:

| Module | Providers | Protocols |
| --- | --- | --- |
| `payos-server-http` | `HttpServerProvider`, `HttpsServerProvider` | `http`, `https` |
| `payos-server-tcp` | `TcpServerProvider` | `tcp` |
| `payos-server-queue` | `QueueServerProvider` | `queue` |

A new server should extend the abstract `Server` and delegate business handling to `super.processRequest(appId, request)`, integrating through `Servers.start(host, port, protocol)`.

## 7. TCP codec/handler plugins

The TCP transport (`payos-server-tcp`) exposes a plugin architecture for custom wire-format parsing. Plugins are JARs in `tcp-handlers-dir` containing implementations of:

- **`TcpMessageDecoder`** — decodes raw bytes from the socket into a `Request` object
- **`TcpMessageEncoder`** — encodes a `Response` object back to bytes for the wire
- **`TcpMessageHandler`** — provides message-specific processing logic

The TCP server scans `tcp-handlers-dir` at startup and selects the first concrete implementation of each interface. If none is found, it falls back to backward-compatible
defaults.

### Directory resolution

`tcp-handlers-dir` is resolved in this precedence order:

1. JVM system property (`-Dtcp.handlers.dir=…`),
2. environment variable (`TCP_HANDLERS_DIR`),
3. the bootstrap settings key (`tcp.handlers.dir`).

### Writing a TCP plugin

1. Implement `TcpMessageDecoder`, `TcpMessageEncoder`, and/or `TcpMessageHandler`.
2. Package the implementations (and dependencies) into a JAR.
3. Drop the JAR into `tcp-handlers-dir`.
4. Restart the server — the TCP transport will discover and load the plugin.

See [configuration/servers.md](../configuration/servers.md) for the complete TCP configuration reference and plugin development details.

## Summary

| You want to… | Use |
| --- | --- |
| Add reusable app behavior without redeploying | a **capability** |
| Share APIs/pages/menus between applications | **application resource inheritance** (`extends`) |
| Intercept request lifecycle events | **internal hooks** |
| Add a database/queue/secret/webhook backend | a **service connector** in `connectors-dir` |
| Call an arbitrary Java library from a script | a **Java extension** in `extensions-dir` |
| Add a new network protocol | a **transport provider** |
| Parse custom wire formats for TCP | a **TCP codec/handler plugin** in `tcp-handlers-dir` |

## Next

- [Eventing & webhooks](eventing-webhooks.md) — the events capabilities and the pipeline emit.
- [Developer: Java extensions](../developer/java-extensions.md) — using `Java.type()`.
