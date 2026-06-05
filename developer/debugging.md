# Debugging

Server-side logic in PayOS is JavaScript executed inside the
[GraalVM sandbox](../architecture/scripting-engine.md). This page collects practical
techniques for diagnosing application behavior.

## Use logging, scoped by correlation ID

Every request carries an `X-Correlation-Id` (generated if absent) and an `X-Tenant-Id`, both
mirrored into the SLF4J MDC and included in audit records. When tracing a problem, capture
the correlation ID from the response headers and grep the logs for it.

```bash
curl -i http://localhost:8080/payments/api/quote   # note X-Correlation-Id in the response
```

```javascript
function execute(request, controlData) {
    var ctx = $Request.getContextData();
    // include the correlation id in your own diagnostics
    return { correlationId: ctx.get("correlationId"), tenant: $Tenant };
}
```

See [operations/observability.md](../operations/observability.md) for the logging/MDC model.

## Common pitfalls

| Symptom | Likely cause |
| --- | --- |
| `$DB` / `$Queue` / `$Secrets` is `undefined` | The corresponding service is not configured; the binding is injected only when present. See [scripting bindings](scripting-bindings.md). |
| `Java.type("...")` throws | The class is on the denylist (e.g. `java.lang.System`) or not on the [extensions path](../configuration/extensions-connectors.md). |
| File/network access fails silently or throws | The sandbox blocks ambient I/O by design — use a binding, connector, or extension. See [scripting engine](../architecture/scripting-engine.md). |
| Resource not found | The `extends` chain or capability activation; a capability resource resolves only when active. See [application model](application-model.md). |
| Request rejected before your script runs | Tenant policy: no resolvable tenant. Enable the dev-only tenant simulator or send `X-Tenant-Id`. See [multi-tenancy](../architecture/multi-tenancy.md). |
| Old code still runs after editing | Confirm [hot reload](../operations/hot-reload.md) picked up the change, or restart. |

## Reproducing across transports

Because logic is [transport-neutral](../architecture/request-processing.md), a bug seen over
TCP or a queue can usually be reproduced over HTTP with the same body and context — the
fastest loop for iteration.

## Error pipeline

A thrown `BusinessException` becomes a structured error response; other exceptions become a
generic error but are logged with correlation/tenant context, and the `api.on-error` hook /
webhook fire. Look there first when an endpoint returns an error. See
[writing APIs: error handling](writing-apis.md#error-handling).

## Static checks before running

Use [`pdoc`](../cli-tools/pdoc.md) to validate your `@payos.openapi` annotations and the
package managers' `--status` commands ([apm](../cli-tools/apm.md), [cpm](../cli-tools/cpm.md),
[ppm](../cli-tools/ppm.md)) to confirm what is installed and active.

## Next

- [Operations: observability](../operations/observability.md)
- [Architecture: scripting engine](../architecture/scripting-engine.md)
