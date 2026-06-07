# PayOS Documentation V1.0.0-RELEASE

Authoritative, code-grounded documentation for the **PayOS** platform — an API-first,
multi-tenant, deployment-agnostic runtime for building financial applications.

It is structured by audience so that each reader can go straight to what they need without loosing track through unrelated material. Every theme lives in exactly **one** place; other documents link to it rather than repeating it.

---

## Navigation by audience

### Everyone — start here

| Document | Purpose |
| --- | --- |
| [Overview](overview/README.md) | What PayOS is, the module ecosystem, the technology stack, and the glossary. |

### Architects

| Document | Purpose |
| --- | --- |
| [Architecture](architecture/README.md) | Platform design, runtime, request processing, scripting sandbox, multi-tenancy, security, data, extensibility, eventing, and Architecture Decision Records. |

### Developers (building applications on PayOS)

| Document | Purpose |
| --- | --- |
| [Developer guide](developer/README.md) | Getting started, the application model, writing API scripts, the script binding reference, data access, secrets, messaging, i18n, hooks, Java extensions, debugging, and API documentation. |
| [Configuration reference](configuration/README.md) | Exhaustive reference of every configuration key (`payos.json` / `bootstrap.json`). |

### Operators (deploying and running PayOS)

| Document | Purpose |
| --- | --- |
| [Operations guide](operations/README.md) | Deployment, bundle encryption, secrets management, observability, and configuration hot-reload. |
| [CLI tools](cli-tools/README.md) | Reference for `apm`, `cpm`, `ppm`, `spm`, `edc`, `pdoc`, and the documentation hub. |

### Maintainers / release engineers

| Document | Purpose |
| --- | --- |
| [Build & release](build-and-release/README.md) | Maven conventions, the parent POM and BOM, versioning rules, and the module map. |

### Reference (cross-cutting appendices)

| Document | Purpose |
| --- | --- |
| [Reference](reference/README.md) | Exhaustive indexes: configuration keys, script bindings, system events, and built-in HTTP endpoints. |

---

### Main technical features of PayOS

| Document | Purpose |
| --- | --- |
| [Main technical features](./technical-features.md) | Main technical attributes and features that distinguishes PayOS. |

## How this documentation is organized

```
payos-docs/
├── overview/            Orientation for all readers
├── architecture/        Internal design (architects)
│   └── adr/             Architecture Decision Records
├── developer/           Building applications (developers)
├── configuration/       Configuration key reference (developers + operators)
├── operations/          Running PayOS in production (operators)
├── cli-tools/           Command-line tool reference (developers + operators)
├── build-and-release/   Build, versioning, module map (maintainers)
└── reference/           Cross-cutting indexes and appendices
```

### Separation of concerns (how redundancy is avoided)

A single subject such as *secrets* is intentionally covered from several distinct angles,
each owned by exactly one document:

| Angle | Owner |
| --- | --- |
| Why the SPI exists and how it plugs into the kernel | [architecture/extensibility.md](architecture/extensibility.md) |
| Which configuration keys configure it | [configuration/secret-service.md](configuration/secret-service.md) |
| How a script uses `$Secrets` | [developer/secrets-usage.md](developer/secrets-usage.md) |
| How an operator provisions and rotates secrets | [operations/secrets-management.md](operations/secrets-management.md) |
| The `spm` command-line interface | [cli-tools/spm.md](cli-tools/spm.md) |

Each document covers its own concern only and links to the others.

---

## Source of truth

All facts in this documentation are derived from the PayOS source modules under the
workspace root, notably:

- `payos` — the kernel (`ma.s2m.payos`)
- `payos-server-http`, `payos-server-tcp`, `payos-server-queue` — transport servers
- `database-service`, `queue-service-nats`, `webhook-service-http` — service providers
- `payos-secret-api`, `secret-service-filesystem`, `secret-service-vault` — secrets
- `payos-runtime` — the runnable distribution
- `payosv2-packer`, `payos-pm`, `pdoc` — tooling
- `payos-parent`, `payos-bom` — build coordination

When the source code and this documentation disagree, **the source code wins** — please
open an issue so the documentation can be corrected.
