# Observability

PayOS is built for regulated operation: every request is **traceable** and **auditable** by
design. This page describes correlation/tenant propagation, logging (MDC), audit records, and
health signals. The standard is mandated by the project's tracing rules and implemented
across all transports.

## Correlation & tenant IDs

Two identifiers are treated as **mandatory cross-transport metadata**:

| Header | Context key | Behavior |
| --- | --- | --- |
| `X-Correlation-Id` | `correlationId` (`CONTEXT_CORRELATION_ID`) | If absent at ingress, a UUID is generated. Propagated unchanged through context, logs, responses, and async side effects. |
| `X-Tenant-Id` | `tenantId` (`CONTEXT_TENANT_ID`) | Resolved at ingress; preserved alongside the correlation id for tenant-scoped traceability. |

Rules:

- Never rewrite correlation IDs downstream unless explicitly crossing a trust boundary with a
  documented mapping.
- Include both identifiers in **error responses** and **async processing logs**.
- All transports (HTTP, TCP, queue) follow the same contract.

## Logging and the MDC

Logging is via SLF4J/Logback. The current tenant is mirrored into the SLF4J **MDC**
(`TenantScope.MDC_TENANT_ID`), so you can include it in your log pattern:

```xml
<pattern>%d %-5level [%X{tenantId}] [%X{correlationId}] %logger - %msg%n</pattern>
```

This makes every log line attributable to a tenant and a request.

> Prefer SLF4J logging over `System.out`; avoid `printStackTrace()`. (Project convention.)

## Audit trail

`AuditLogger.logApiExecution(userId, tenantId, correlationId, path, status)` runs in the
`finally` stage of the API pipeline — so an audit record is produced **whether the request
succeeded or failed**. Secret operations additionally write to the `payos.secret.audit`
logger (see [secrets-management.md](secrets-management.md)).

For PCI DSS evidence, route audit loggers to a tamper-resistant sink / SIEM.

## Diagnostics

`Diagnostics` (`ma.s2m.payos.diagnostics`) is a separate event category from the audit trail —
short-retention, WARN-level incident-triage data, not regulator-mandated evidence. `Diagnostics`
and `IDiagnosticsRecorder` are nature-agnostic (a single `logEvent(DiagnosticEvent)` entry
point); every diagnostic event carries a mandatory `nature` field identifying what kind of
diagnostic it is. The only nature implemented so far is `"connector"` (connector
retry/terminal-routing decisions), produced by a separate helper,
`ma.s2m.payos.connector.diagnostics.ConnectorDiagnosticsHelper`, and called by
`ConnectorScriptHandle` — see
[configuration/connector-framework-parameters-v3-2026-08-11.md](../configuration/connector-framework-parameters-v3-2026-08-11.md)
§11. The default implementation logs one JSON line per event to the SLF4J **`DIAGNOSTICS`**
logger category — route/filter it independently in logback configuration, the same way `AUDIT`
is routed separately.

## Health and lifecycle

| Endpoint | Purpose |
| --- | --- |
| `/health` | Liveness/readiness probe. |
| `/stop` | Graceful shutdown (localhost only). |

See [reference/http-endpoints.md](../reference/http-endpoints.md).

## Tracing async side effects

Webhook deliveries and queue messages carry the originating request's correlation and tenant
IDs, so asynchronous effects remain linkable to the request that triggered them. When
publishing to a queue, include these in the payload — see
[developer/queue-messaging.md](../developer/queue-messaging.md).

## What to monitor

- Error rates per `api.on-error` event / status code.
- Per-tenant request volume vs. configured [quotas](../configuration/multi-tenancy.md).
- Connector health (DB pool, queue connectivity, secret provider reachability).
- Configuration reloads (the watcher logs reload events — see [hot-reload.md](hot-reload.md)).

## Next

- [hot-reload.md](hot-reload.md)
- [reference/http-endpoints.md](../reference/http-endpoints.md)
