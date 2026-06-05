# Extensibility

PayOS adds behavior **without modifying the rigid core**. There are three distinct
extension mechanisms plus the capability system. Knowing which is which — and how each is
discovered — is essential.

## The three mechanisms at a glance

| Mechanism | Directory | Discovery | Implements a PayOS interface? | Used for |
| --- | --- | --- | --- | --- |
| **Service connectors** | `connectors-dir` | Java `ServiceLoader` (SPI) | Yes — an SPI factory | Database, queue, secret, webhook providers; JDBC drivers. |
| **Java extensions** | `extensions-dir` | `Java.type('…')` from scripts | No | Arbitrary Java libraries callable from JavaScript (e.g. jPOS). |
| **Transport providers** | classpath / bundled | `ServiceLoader<ServerProvider>` | Yes — `ServerProvider` | New protocols (`http`, `https`, `tcp`, `queue`). |

A fourth mechanism, **capabilities**, extends *applications* (not the core) and is covered
at the end of this document.

## 1. Service connectors (SPI)

A connector is a JAR in `connectors-dir`. `ConnectorLoader` builds a `URLClassLoader` over
those JARs at bootstrap and registers it via `PayOSConfig.setConnectorClassLoader(...)`.
Each service is then discovered with `ServiceLoader` against that classloader and selected
by a `type` string.

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

1. Implement the SPI factory (e.g. `ISecretProviderFactory`) and its product
   (`ISecretProvider`).
2. Return a unique `type()` (e.g. `"vault"`).
3. Register the factory in `META-INF/services/<fully-qualified-SPI-interface>`.
4. Build a JAR and drop it (with its dependencies) into `connectors-dir`.
5. Reference its `type` in the relevant configuration block.

The shipped connectors are the reference implementations:
`queue-service-nats`, `webhook-service-http`, `secret-service-filesystem`,
`secret-service-vault`, and `database-service`. See
[configuration/extensions-connectors.md](../configuration/extensions-connectors.md) for
directory resolution and [build-and-release/module-map.md](../build-and-release/module-map.md)
for artifacts.

> **Kernel rule:** connector implementations live in standalone modules; only the SPI
> interface stays in the kernel.

## 2. Java extensions

An extension is **any** Java library JAR placed in `extensions-dir`. `ExtensionLoader`
builds a `URLClassLoader` over those JARs and registers it as the **host class loader** of
the scripting sandbox. Scripts then reach the classes via:

```javascript
const ISOMsg = Java.type('org.jpos.iso.ISOMsg');
const msg = new ISOMsg();
```

Key differences from connectors:

- An extension implements **no PayOS interface** — it is ordinary Java.
- It is invoked **from scripts** with `Java.type()`, not auto-discovered by the kernel.
- Access is still bounded by the [scripting sandbox](scripting-engine.md): blocked classes
  (e.g. `java.lang.System`) remain unreachable.

This is the mechanism for integrating libraries such as **jPOS** for ISO 8583 message
handling. Developer guidance and examples: [developer/java-extensions.md](../developer/java-extensions.md).

### Directory resolution (both loaders)

`connectors-dir` and `extensions-dir` are each resolved in this precedence order:

1. JVM system property (`-Dconnectors-dir=…` / `-Dextensions-dir=…`),
2. environment variable (`PAYOS_CONNECTORS_DIR` / `PAYOS_EXTENSIONS_DIR`),
3. the bootstrap settings key (`connectors-dir` / `extensions-dir`).

## 3. Transport providers

A transport adds a protocol by implementing `ma.s2m.payos.servers.ServerProvider`:

```java
String  protocol();                                       // e.g. "tcp"
IServer create(Map<String,Object> serverConfig);          // build the server
```

`Servers` discovers providers through `ServiceLoader<ServerProvider>` and dispatches by
protocol name. The shipped transports register themselves in
`META-INF/services/ma.s2m.payos.servers.ServerProvider`:

| Module | Providers | Protocols |
| --- | --- | --- |
| `payos-server-http` | `HttpServerProvider`, `HttpsServerProvider` | `http`, `https` |
| `payos-server-tcp` | `TcpServerProvider` | `tcp` |
| `payos-server-queue` | `QueueServerProvider` | `queue` |

A new server should extend the abstract `Server` and delegate business handling to
`super.processRequest(appId, request)`, integrating through
`Servers.start(host, port, protocol)`. The TCP server additionally exposes its own
**codec/handler plugin** model (`TcpMessageDecoder/Encoder/Handler`) loaded from
`tcp-handlers-dir` — see [configuration/servers.md](../configuration/servers.md).

## 4. Capabilities (extending applications)

Capabilities extend the **guest layer**, not the core. A capability is an application with
`category: "capability"` that other applications reference through `extends`. It can be
installed and activated **without redeploying the runtime**.

- **Resolution gating** — `ResourceLocator` only resolves a capability's resources when
  `IActivationStore.isActive(capabilityId, requestingAppId, tenantId)` is true.
- **State** — installation and activation state live under `{configDir}/.capabilities/`
  (`registry.json`, `activation.json`, append-only `events.ndjson`).
- **Lifecycle** — install / uninstall / activate / deactivate run capability lifecycle
  hooks and the relevant [system events](eventing-webhooks.md).
- **Management** — performed with [`cpm`](../cli-tools/cpm.md).

## Summary

| You want to… | Use |
| --- | --- |
| Add a database/queue/secret/webhook backend | a **connector** in `connectors-dir` |
| Call an arbitrary Java library from a script | an **extension** in `extensions-dir` |
| Add a new network protocol | a **transport provider** |
| Add reusable app behavior without redeploying | a **capability** |

## Next

- [Eventing & webhooks](eventing-webhooks.md) — the events capabilities and the pipeline emit.
- [Developer: Java extensions](../developer/java-extensions.md) — using `Java.type()`.
