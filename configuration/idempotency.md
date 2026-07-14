# Idempotency configuration

The `idempotency` block controls `IdempotencyService`, which caches API responses by a client-supplied idempotency key so mutating requests (e.g. payment operations) are safe to retry. The service and pipeline integration are described in [architecture/security-architecture.md](../architecture/security-architecture.md#idempotency); request-flow placement is in [architecture/request-processing.md](../architecture/request-processing.md).

## Shape

```json
{
  "idempotency": {
    "enabled": true,
    "ttlSeconds": 86400,
    "headerName": "X-Idempotency-Key",
    "failOnAbsenceOfIdempotencyKey": true
  }
}
```

## Keys

From `IConfigSpec.Idempotency`:

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `true` | Enable the idempotency service. When `false`, `checkIdempotency` is a no-op and `storeResponse` never caches. |
| `ttlSeconds` | `86400` (24h) | How long a cached response stays valid for replay. |
| `headerName` | `X-Idempotency-Key` | Request header carrying the idempotency key. |
| `failOnAbsenceOfIdempotencyKey` | `true` | When `true`, a request with a missing/blank idempotency key is rejected (`400 Bad Request`) before script execution. When `false`, such a request simply proceeds without an idempotency check (no rejection, no caching). |

## Resolution order

Each key is resolved in this order, first match wins:

```
idempotency.<key> in bootstrap.json
  → system property (payos.idempotency.<key>)
  → environment variable (PAYOS_IDEMPOTENCY_<KEY>)
  → built-in default
```

| Key | System property | Environment variable |
| --- | --- | --- |
| `enabled` | `payos.idempotency.enabled` | `PAYOS_IDEMPOTENCY_ENABLED` |
| `ttlSeconds` | `payos.idempotency.ttlSeconds` | `PAYOS_IDEMPOTENCY_TTL_SECONDS` |
| `headerName` | `payos.idempotency.headerName` | `PAYOS_IDEMPOTENCY_HEADER_NAME` |
| `failOnAbsenceOfIdempotencyKey` | `payos.idempotency.failOnAbsenceOfIdempotencyKey` | `PAYOS_IDEMPOTENCY_FAIL_ON_ABSENCE_OF_IDEMPOTENCY_KEY` |

The system property / environment variable fallback predates the `bootstrap.json` block and is
kept for backward compatibility; prefer configuring `idempotency` in `bootstrap.json` for new
deployments.

## Wiring

`BootServer` constructs `IdempotencyConfig(PayOSConfig.settings)` and an `InMemoryIdempotencyStore`,
then registers the resulting `IdempotencyService` via `PayOSConfig.setIdempotencyService(...)`.
`ApiResourceHandler` calls `checkIdempotency(request)` before script execution and
`storeResponse(request, response)` after a successful run.

## Next

- [Architecture: security](../architecture/security-architecture.md#idempotency)
- [Architecture: request processing](../architecture/request-processing.md)
