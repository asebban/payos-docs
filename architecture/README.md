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
14. [License validation](./license-validation-architecture.md)
15. [Connector framework architecture](./connector-framework-architecture-v1-2026-08-24.md) — the business/payment connector plugin system (`IConnector`, `$Connector(...)`, module split across `payos-connector-api`/`payos-connector-sdk`/`payos-foundation`/kernel)
16. [Connector framework class diagram](./connector-framework-class-diagram-v1-2026-09-02.md) — every production class of the connector framework across all five repositories, with inheritance/implementation/usage/instantiation relationships
17. [Connector framework class reference](./connector-framework-class-reference-v1-2026-09-07.md) — guided class-by-class map of the connector framework by repository, lifecycle phase, execution flow, and debugging symptom
18. [Observability metrics architecture](./observability-metrics-architecture-v1-2026-09-06.md) — Micrometer-backed PayOS metrics, Prometheus exposure, execution flows, class diagrams, and metric taxonomy
19. [Identity-first tenant resolution & centralized transport security](./tenant-identity-resolution-plan-v5-2026-09-20.md) — implemented: authenticated principal is the primary tenant source ahead of `X-Tenant-Id`; principal resolution, API-key validation, and tenant scope opening are centralized in `Server.processRequest` rather than per transport; interactive Keycloak login stays HTTP-only; `TenantPolicyService` resolves its own principal directly; `DynamicDataAccessService`'s ambient tenant context is now restore-safe under nested `$Api.*` calls
20. [Task Scheduling System class diagram](./job-scheduler-class-diagram-v18-2026-10-07.md) — living document, updated per story, but with a known, explicitly-flagged gap (§9): §4-8/10-11 cover Stories 1.1-1.2 (including the AC1 production-wiring gap resolution) in full, §13 (added 2026-10-06) covers Stories 2.1-2.2 (the `history/` package — `JobExecutionEvent`/`JobExecutionRepository`, real Quartz misfire detection, the per-tenant execution concurrency quota, cluster-recovery wiring, plus a same-day per-tenant-quota-override follow-up), §14 (added 2026-10-07) covers Story 2.3 (the `metrics/` package, `JobExecutionEvent.durationMillis`, `CompositeJobExecutionListener`, and a same-day follow-up adding a standalone-mode Prometheus HTTP exporter — a small cross-repo change to `payos` kernel's `MicrometerMetricsRecorder`), and §15 (added 2026-10-07) covers Story 2.4 (sensitive-data masking — `history/SensitiveDataScrubber`, a best-effort free-text scanner reusing `SensitiveFieldMasker.isSensitiveKey` as its denylist, plus defensive masking in `JobMetricsEmitter`) — but Stories 1.3-1.5 (platform-level jobs, `custom-jobs.json`, `ExclusiveScriptJob`) were never added between 2026-10-02 and 2026-10-06 and remain undocumented here despite being `done`. Also covers the `IDatabaseService`/`DynamicDataAccessService` additions made for TSS (`openDedicatedSessionFor`/`DedicatedTenantSession`), the kernel's `DatabaseServiceInitializer.resolveTenantDatabaseConnections`/`initializePlatform` (both reused unchanged by TSS), and its `IJobConfigSpec` configuration-key spec, with execution-flow walkthrough and a bytecode-verified explanation of Quartz's `Scheduler`/`JobFactory`/`JobRunShell`/`TriggerListener`/`DBConnectionManager` wiring
21. [Atomicity across `$DB`, `$Audit`, and `$Queue`](./atomicity-db-audit-queue-v10-2026-10-20.md) — `atomicityScope` (per-endpoint, `mappings.json`), the `$DB`/`$PlatformDB` read-only/absent lockdown it drives, `$Audit`'s direct transactional write to `payos_audit_event` (no buffer, no dispatcher), and `$Queue`'s Transactional Outbox pattern (`payos_queue_outbox` + `QueueOutboxDispatcher`, paired with `QueueServer`, unlimited retry, never abandons a message)
22. [tenant-config-loader-database](./tenant-config-loader-database-v1-2026-10-22.md) — the `IConfigLoader` SPI, how a database-backed tenant registry is deep-merged into `multitenancy.tenants` at boot, the one-shot loader vs. the separate opt-in `TenantRegistryChangePoller` live-reload trigger, and why a failing loader is never fatal to boot

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
