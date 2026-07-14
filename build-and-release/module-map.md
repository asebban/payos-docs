# Module map

The canonical list of PayOS modules — what each one is, the artifact it produces, and how it
fits together. Versions are in [versioning.md](versioning.md).

## Core

| Module | Artifact | Role |
| --- | --- | --- |
| `payos-parent` | parent POM | Java 21 + dependency version management. |
| `payos-bom` | `payos-core-bom` | BOM that pins platform versions for consumers (owns `payos-kernel.version`). |
| `payos-kernel` | `payos-kernel` | The rigid core: config, servers abstraction, resources, scripting, security, multi-tenancy, hooks, SPIs. Entry point `ma.s2m.payos.BootServer`. |
| `payos-runtime` | runnable fat JAR | Shades the kernel + transports + standard connectors into one executable (`BootServer`). |

## Transports (`ServerProvider` implementations)

| Module | Protocol(s) | Key types |
| --- | --- | --- |
| `payos-server-http` | `http`, `https` | `HttpServerProvider`, `HttpsServerProvider`, `HttpServer`, `SwaggerUIHandler`. |
| `payos-server-tcp` | `tcp` | `TcpServerProvider`, `TcpServer`; plugin interfaces `TcpMessageDecoder/Encoder/Handler`. |
| `payos-server-queue` | `queue` | `QueueServerProvider`, `QueueServer`. |

## Service connectors (SPI implementations)

| Module | SPI | Type | Binding |
| --- | --- | --- | --- |
| `database-service` (`dynamic-database-service`) | `IDatabaseServiceFactory` | (kernel-managed) | `$DB` |
| `queue-service-nats` | `IQueueClientFactory` | `nats` | `$Queue` |
| `webhook-service-http` | `IWebhookDispatcherFactory` | `http` | `$WebHooks` |
| `secret-service-filesystem` | `ISecretProviderFactory` | `filesystem` | `$Secrets` |
| `secret-service-vault` | `ISecretProviderFactory` | `vault` | `$Secrets` |
| `payos-secret-api` | (SPI definitions) | — | (`ISecretProvider`, models, exceptions) |

> The webhook factory is discovered via a standard `ServiceLoader`; the database/queue/secret
> connectors are loaded from `connectors-dir` via the connector classloader. See
> [configuration/extensions-connectors.md](../configuration/extensions-connectors.md).

## Tooling

| Module | Command | Role |
| --- | --- | --- |
| `payos-pm` | `apm`, `cpm`, `ppm` | App/capability/product package managers. |
| `secret-service-filesystem` | `spm` | Filesystem secret manager CLI. |
| `payosv2-packer` | `edc` | Bundle encrypt/pack/unpack. |
| `pdoc` | `pdoc` | Static OpenAPI generator. |
| `payos-docshub` | `docshub` | Documentation aggregator. |

## What the runtime embeds

`payos-runtime` (1.8.0-RELEASE) shades:

- payos-kernel 1.8.0-RELEASE
- payos-server-http 1.2.0, payos-server-tcp 1.0.6, payos-server-queue 1.1.0-RELEASE
- dynamic-database-service 1.1.9-RELEASE
- queue-service-nats 1.1.0-RELEASE
- webhook-service-http 1.0.4-RELEASE
- payos-secret-api 1.0.0-RELEASE, secret-service-filesystem 1.1.0-RELEASE, secret-service-vault 1.1.0-RELEASE
- payos-notification-api 1.0.0-RELEASE, payos-notification-connector 1.1.0-RELEASE, connector-sdk 1.2.0-RELEASE

`secret-service-vault` is shaded directly into the runtime jar alongside
`secret-service-filesystem` — it is a direct `payos-runtime` dependency, not an
additional/alternative connector-jar-only backend. A different queue/DB backend would still
be deployed as connector JARs in `connectors-dir`.

## Dependency direction

```
consumers → payos-bom (versions)
runtime   → kernel + transports + connectors (shaded)
transports/connectors → kernel SPIs only (never depend on each other)
```

The kernel never depends on a connector implementation — only on its SPI interface.

## Next

- [versioning.md](versioning.md)
- [architecture/extensibility.md](../architecture/extensibility.md)
