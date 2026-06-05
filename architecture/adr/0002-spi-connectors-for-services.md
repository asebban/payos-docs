# ADR-0002 — Pluggable services via SPI connectors

**Status:** Accepted

## Context

PayOS must run on-premise, on private cloud, and on public cloud, integrating with whatever
database, message broker, secret store, and webhook infrastructure each deployment provides.
Bundling every possible backend into the kernel would bloat the artifact, force a kernel
release for each new backend, and pin dependency versions globally.

## Decision

Define each service as a kernel **SPI** and ship implementations as standalone **connector**
JARs discovered at runtime through Java's `ServiceLoader`:

| Service | SPI factory | Selected by |
| --- | --- | --- |
| Database | `IDatabaseServiceFactory` | kernel-managed |
| Queue | `IQueueClientFactory` | `type` (e.g. `nats`) |
| Secrets | `ISecretProviderFactory` | `type` (e.g. `filesystem`, `vault`) |
| Webhooks | `IWebhookDispatcherFactory` | `webhooks.dispatcher` (e.g. `http`) |

Connectors are loaded from `connectors-dir` via a dedicated `URLClassLoader`. Only the SPI
interface lives in the kernel; implementations live outside it.

## Consequences

- **Positive:** the same kernel supports many backends; a new backend is a new JAR, not a
  kernel release; JDBC drivers and broker clients stay out of the fat JAR.
- **Positive:** deployments choose backends through configuration (`type` strings).
- **Negative:** operators must place the right connector JARs (and their dependencies) in
  `connectors-dir`; misconfiguration surfaces only at bootstrap.
- See [extensibility.md](../extensibility.md) and [data-architecture.md](../data-architecture.md).
