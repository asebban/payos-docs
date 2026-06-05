# ADR-0004 — Structural multi-tenancy with mandatory tenant/correlation IDs

**Status:** Accepted

## Context

PayOS serves many financial institutions from shared runtimes. In a regulated environment,
tenant isolation cannot depend on each application remembering to filter by tenant, and every
action must be traceable for incident investigation (PCI DSS Req. 10).

## Decision

Make tenant isolation **structural**:

- Resolve the tenant at transport ingress and open a tenant scope **before** any business
  logic runs (`TenantPolicyService.enforceAndOpenScope`), throwing on policy violation.
- Bind the tenant to the database service for the request's lifetime
  (`setCurrentTenant` / `beginRequestScope` / `endRequestScope` / `clearCurrentTenant`).
- Treat `X-Correlation-Id` and `X-Tenant-Id` as **mandatory cross-transport metadata**:
  generate a correlation UUID if absent, propagate both unchanged through context, logs,
  responses, and async side effects, and never rewrite them downstream without a documented
  trust-boundary mapping.
- Mirror the tenant into the SLF4J MDC and emit an audit record per API execution.

## Consequences

- **Positive:** isolation does not rely on application discipline; every log line and
  webhook is tenant- and trace-scoped; incidents are investigable.
- **Positive:** per-tenant schemas, isolation modes, and quotas are configurable centrally.
- **Negative:** requests without a resolvable tenant are rejected (except via the
  development-only tenant simulator); all transports must implement scope opening.
- See [multi-tenancy.md](../multi-tenancy.md) and [security-architecture.md](../security-architecture.md).
