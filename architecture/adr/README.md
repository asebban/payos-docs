# Architecture Decision Records (ADRs)

An **Architecture Decision Record** captures a significant architectural decision together
with its context and consequences. ADRs are immutable once accepted: to change a decision,
add a new ADR that supersedes the old one.

This index lists the decisions that are already embedded in the PayOS codebase. Each ADR
below is written from the code as it exists today; they document *why* the platform is
shaped the way the [architecture](../README.md) section describes.

## Format

Each ADR follows a short template:

- **Status** — Accepted / Superseded.
- **Context** — the forces and constraints.
- **Decision** — what was decided.
- **Consequences** — the resulting trade-offs.

## Index

| ADR | Title | Status |
| --- | --- | --- |
| [ADR-0001](0001-rigid-core-flexible-guest-layer.md) | Rigid core with a flexible JavaScript guest layer | Accepted |
| [ADR-0002](0002-spi-connectors-for-services.md) | Pluggable services via SPI connectors | Accepted |
| [ADR-0003](0003-transport-agnostic-request-model.md) | Transport-agnostic request/response model | Accepted |
| [ADR-0004](0004-structural-multi-tenancy.md) | Structural multi-tenancy with mandatory tenant/correlation IDs | Accepted |
| [ADR-0005](0005-sandboxed-graalvm-scripting.md) | Sandboxed GraalVM scripting, deny-by-default | Accepted |
| [ADR-0006](0006-distributed-cache-over-sticky-sessions.md) | Distributed shared cache (Redis) over sticky sessions for cross-instance state | Accepted |
| [ADR-0007](0007-distributed-cache-middleware-selection.md) | Distributed cache middleware selection: Valkey over Redis, Dragonfly, Hazelcast/Ignite, NATS JetStream KV, Memcached | Proposed |

> To add a new decision, copy the format above into `NNNN-title.md`, increment the number,
> and add a row here.
