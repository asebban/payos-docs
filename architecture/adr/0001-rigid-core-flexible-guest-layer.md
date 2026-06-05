# ADR-0001 — Rigid core with a flexible JavaScript guest layer

**Status:** Accepted

## Context

PayOS must serve a fast-moving, multi-product financial ecosystem while remaining stable,
performant, and compliant. Business requirements change frequently; the platform's core
contracts must not. Letting every feature modify the core would erode performance,
security, and maintainability.

## Decision

Split the system into a **rigid core** (the Java kernel and transports) and a **flexible
guest layer** (JavaScript applications, capabilities, and products, plus connector and
extension JARs). The core is never modified to add a business feature. New behavior is
delivered by:

- writing JavaScript resources executed in a sandbox,
- plugging in service connectors through SPIs,
- adding Java extensions callable via `Java.type()`, and
- adding transport providers for new protocols.

The core exposes a stable set of script bindings (`$Request`, `$Response`, `$DB`, …) and SPI
interfaces.

## Consequences

- **Positive:** the hot path stays fast and stable; application teams iterate without core
  releases; one runtime hosts many tenants and products.
- **Positive:** clear ownership boundary between platform and application teams.
- **Negative:** business logic in JavaScript is constrained by the sandbox (no ambient I/O,
  threads, or processes); some integrations require a connector or extension JAR.
- See [platform-architecture.md](../platform-architecture.md) and
  [scripting-engine.md](../scripting-engine.md).
