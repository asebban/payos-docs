# Extensions & connectors

PayOS loads plugin code from the filesystem at runtime through dedicated classloaders. Two
plugin families exist: **connectors** (SPI service backends — database, queue, secrets) and
**extensions** (Java libraries callable from scripts via `Java.type()`). A third, transport
plugins for TCP, is configured per server. This page covers the discovery paths and
classloader hierarchy; the design rationale is in
[architecture/extensibility.md](../architecture/extensibility.md).

> **Not to be confused with the connector *framework*.** The "connectors" on this page are SPI
> backend plugins (database/queue/secret factories) selected by `connectors-dir` and already
> wired into `BootServer`. PayOS also has a separate, newer **connector framework** for
> business/payment connectors invoked from scripts via `$Connector(...)`, configured through
> `connectors.json` and a `META-INF/connector.properties` descriptor — see
> [connector-framework-parameters-v2-2026-07-12.md] (connector-framework-parameters-v2-2026-07-12.md).
> The two mechanisms share the word "connector" but are otherwise independent.

## Discovery paths

| Plugin family | Bootstrap key | Env var | System property |
| --- | --- | --- | --- |
| Connectors (SPI backends) | `connectors-dir` | `PAYOS_CONNECTORS_DIR` | (set) |
| Extensions (script-callable Java) | `extensions-dir` | `PAYOS_EXTENSIONS_DIR` | (set) |
| TCP codec/handler plugins | `tcp-handlers-dir` (per server) | `TCP_HANDLERS_DIR` | (set) |

**Resolution order for each:** system property → environment variable → bootstrap/server
key. This lets operators override locations per environment without editing the bundle.

```json
{
  "connectors-dir": "connectors",
  "extensions-dir": "extensions"
}
```

## Classloader hierarchy

PayOS layers classloaders so that application code can see extensions, which can see
connectors, which can see the runtime:

```
application  →  extension  →  connector  →  runtime
```

- `ConnectorLoader` builds the connector classloader from JARs in `connectors-dir`; the
  result is set via `PayOSConfig.setConnectorClassLoader(...)`.
- `ExtensionLoader` builds the extension classloader from JARs in `extensions-dir`; set via
  `PayOSConfig.setExtensionClassLoader(...)`. This classloader is also used as the GraalVM
  context host classloader so `Java.type()` can reach whitelisted extension classes.

## What goes where

| Put it in connectors when… | Put it in extensions when… |
| --- | --- |
| It implements an SPI factory (`IDatabaseServiceFactory`, `IQueueClientFactory`, `ISecretProviderFactory`). | Scripts need to call its classes via `Java.type()`. |
| It is selected by a config `type` (e.g. `nats`, `vault`). | It is a helper library / protocol stack (e.g. jPOS). |
| It needs a JDBC driver or broker client. | — |

> The webhook dispatcher is the exception: it is discovered via a **standard** `ServiceLoader`
> (not the connector classloader) and selected by `webhooks.dispatcher`. See
> [webhook-service.md](webhook-service.md).

## TCP transport plugins

For the `tcp` transport, decoder/encoder/handler plugins are scanned from `tcp-handlers-dir`.
The first concrete implementation of each interface wins; UTF-8 defaults apply otherwise. See
[servers.md](servers.md) and
[architecture/extensibility.md](../architecture/extensibility.md).

## Operational placement

Operators stage the right JARs (and their transitive dependencies) into these directories per
deployment. The fat runtime JAR already embeds the standard connectors; additional or
alternative backends are dropped into `connectors-dir`. See
[operations/deployment.md](../operations/deployment.md).

## Next

- [Architecture: extensibility](../architecture/extensibility.md)
- [developer/java-extensions.md](../developer/java-extensions.md)
- [operations/deployment.md](../operations/deployment.md)
