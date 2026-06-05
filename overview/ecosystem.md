# Ecosystem & module map

PayOS is a set of independently versioned Maven modules. They fall into five groups:
the **kernel**, the **transport servers**, the **service providers**, the **distribution**,
and the **tooling / build coordination** modules.

## At a glance

| Module | Artifact | Group | Role |
| --- | --- | --- | --- |
| `payos` (kernel) | `ma.s2m.payos:payos-kernel` | Kernel | Core runtime: bootstrap, config, request pipeline, scripting, security, applications, SPI contracts. |
| `payos-server-http` | `ma.s2m.payos:payos-server-http` | Transport | Undertow-based HTTP/HTTPS server (`http`, `https`). |
| `payos-server-tcp` | `ma.s2m.payos:payos-server-tcp` | Transport | TCP server with a plugin (codec/handler) architecture (`tcp`). |
| `payos-server-queue` | `ma.s2m.payos:payos-server-queue` | Transport | Queue-driven request/response server (`queue`). |
| `database-service` | `ma.s2m:dynamic-database-service` | Service | `IDatabaseService` implementation — multi-tenant dynamic data access (`$DB`). |
| `queue-service-nats` | `ma.s2m.payos:queue-service-nats` | Service | `IQueueClient` implementation over NATS (`$Queue`, type `nats`). |
| `webhook-service-http` | `ma.s2m.payos:webhook-service-http` | Service | `IWebhookDispatcher` over HTTP (type `http`). |
| `payos-secret-api` | `ma.s2m.payos:payos-secret-api` | Service (API) | The secret-provider SPI and model types. |
| `secret-service-filesystem` | `ma.s2m.payos:secret-service-filesystem` | Service | File-based `ISecretProvider` (AES-256-GCM, type `filesystem`). |
| `secret-service-vault` | `ma.s2m.payos:secret-service-vault` | Service | HashiCorp Vault `ISecretProvider` (KV v2, type `vault`). |
| `payos-runtime` | `ma.s2m.payos:payos-runtime` | Distribution | The runnable, all-in-one server JAR (`BootServer`). |
| `payosv2-packer` | `ma.s2m:payosv2-packer` | Tooling | `edc` — bundle encryption/decryption CLI. |
| `payos-pm` | `ma.s2m.payos:payos-pm` | Tooling | `apm` / `cpm` / `ppm` — application, capability, and product package managers. |
| `pdoc` | `ma.s2m.payos:pdoc` | Tooling | Static OpenAPI documentation generator. |
| `payos-parent` | `ma.s2m.payos:payos-parent` | Build | Maven parent POM: Java 21 baseline, plugin and dependency versions. |
| `payos-bom` | `ma.s2m.payos:payos-core-bom` | Build | Bill of Materials: owns the kernel version contract. |

> The exact pinned versions live in [build-and-release/module-map.md](../build-and-release/module-map.md).
> The frontend (`nuxt-app`) and product/management apps are out of scope for this runtime
> documentation.

## How the modules fit together

```
                          payos-parent (Maven conventions, versions)
                                       │ inherited by
                                       ▼
            payos-bom ──────▶ payos-kernel ◀────── SPI contracts:
        (owns kernel version)   (the core)         IServer / ServerProvider
                                   ▲                IDatabaseService
              ┌────────────────────┼────────────┐  IQueueClient(Factory)
              │                    │            │  IWebhookDispatcher(Factory)
   transport servers        service providers   │  ISecretProvider(Factory)
   ─ payos-server-http      ─ database-service   │
   ─ payos-server-tcp       ─ queue-service-nats │
   ─ payos-server-queue     ─ webhook-service-http
                            ─ secret-service-filesystem
                            ─ secret-service-vault
              │                    │            │
              └──────────┬─────────┴────────────┘
                         ▼
                   payos-runtime  (shades kernel + servers + services → runnable JAR)
                         ▲
        tooling (separate executables, operate on a runtime bundle):
        ─ payos-pm:  apm / cpm / ppm     ─ pdoc       ─ payosv2-packer: edc
```

### The three plug-in mechanisms

PayOS extends without modifying the core through three distinct mechanisms. Knowing which
is which is essential; they are fully described in
[architecture/extensibility.md](../architecture/extensibility.md).

| Mechanism | Directory | Discovery | Used for |
| --- | --- | --- | --- |
| **Service connectors** | `connectors-dir` | Java `ServiceLoader` (SPI) | Database, queue, secret, and (some) webhook providers. |
| **Java extensions** | `extensions-dir` | `Java.type('…')` from scripts | Arbitrary Java libraries callable from JavaScript (e.g. jPOS). |
| **Transport providers** | bundled / classpath | `ServiceLoader<ServerProvider>` | New protocols (`http`, `tcp`, `queue`, …). |

### Deployment units

Applications are packaged and deployed at three granularities, managed by the
[package-manager CLIs](../cli-tools/README.md):

| Unit | Managed by | Description |
| --- | --- | --- |
| **Application** | `apm` | A single application: APIs, pages, menus, libraries, i18n. |
| **Capability** | `cpm` | A self-contained extension that other apps `extends`, installable/activatable without redeploying the runtime. |
| **Product** | `ppm` | A bundle of applications plus shared server configuration. |

See the [glossary](glossary.md) for precise definitions.

## Next

- [Technology stack](technology-stack.md) — the pinned versions behind these modules.
- [Architecture](../architecture/README.md) — how the kernel actually processes a request.
