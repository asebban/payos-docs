# Architecture

Internal design of PayOS for **architects** and anyone who needs to understand how the runtime works. These documents describe *why* the system is shaped the way it is and *how* the pieces interact. They are grounded in the kernel and transport source code.

## Reading order

0. [Architecture style](./architecture-style.md) — Description of the architecture style of PayOS
1. [Platform architecture](platform-architecture.md) — the kernel / guest-layer split and the layering.
2. [Runtime architecture](runtime-architecture.md) — bootstrap, configuration loading, hot-reload, global registry.
3. [Request processing](request-processing.md) — the full path from transport ingress to response.
4. [Scripting engine](scripting-engine.md) — the GraalVM Polyglot sandbox and its security model.
5. [Multi-tenancy](multi-tenancy.md) — tenant identity, scopes, isolation, and quotas.
6. [Security architecture](security-architecture.md) — authentication, sessions, principal, and traceability.
7. [Data architecture](data-architecture.md) — the database service and multi-tenant data access.
8. [Extensibility](extensibility.md) — connectors, extensions, transport providers, and capabilities.
9. [Eventing & webhooks](eventing-webhooks.md) — hooks, system events, and webhook dispatch.
10. [Architecture Decision Records](adr/README.md) — significant decisions captured over time.
11. [Server side i18n localization service](./server-side-i18n-architecture.md) — Architecture of the backend localization system
12. [Ecosystem surface](./ecosystem-surface.md) — a description of the surface that enables access to the PayOS platform
13. [Queue server](./queue-architecture.md) — Queue server architecture

## Cross-cutting principles

Every document here is an application of the [PayOS design principles](../overview/what-is-payos.md#design-principles):

- **Rigid core + flexible guest layer** → the kernel is stable; behavior is added through
  scripts, connectors, and extensions, never by editing the core.
- **Externalized runtime** → applications are data loaded *into* the process, decoupled
  from the transport servers.
- **Native multi-tenancy** → isolation is structural; the tenant scope is opened before any
  business logic runs.
- **Secure & compliant by design** → the sandbox, the security pipeline, and the
  correlation/tenant propagation are built into the request path.
- **Modular by design** → transports and services are independent modules behind SPIs.
