# Servers configuration

The `servers` array declares the transport listeners PayOS starts. Each entry is started via
`Servers.start(host, port, protocol[, config])`, which selects a `ServerProvider` by
protocol through `ServiceLoader`. The transport architecture is in
[architecture/request-processing.md](../architecture/request-processing.md) and
[architecture/extensibility.md](../architecture/extensibility.md).

## Shape

```json
{
  "servers": [
    { "protocol": "https", "host": "0.0.0.0", "port": 8443,
      "keystore": "config/keystore.p12", "keystorePassword": "${KS_PASS}",
      "keystoreType": "PKCS12", "keyAlias": "payos", "keyPassword": "${KEY_PASS}" },

    { "protocol": "tcp", "host": "0.0.0.0", "port": 9000,
      "tcp-handlers-dir": "tcp-handlers" },

    { "protocol": "queue", "host": "localhost", "port": 4222,
      "type": "nats", "consumer-topic": "payos.requests" }
  ]
}
```

## Common keys (all protocols)

From `IConfigSpec.Servers.Server`:

| Key | Purpose |
| --- | --- |
| `protocol` | `http`, `https`, `tcp`, or `queue`. Selects the `ServerProvider`. |
| `host` | Bind address. |
| `port` | Bind port. |

## TLS keys (`https`)

| Key | Purpose |
| --- | --- |
| `keystore` | Path to the keystore. |
| `keystorePassword` | Keystore password. |
| `keystoreType` | e.g. `PKCS12`, `JKS`. |
| `keyAlias` | Alias of the server key. |
| `keyPassword` | Key password. |

HTTPS adds HSTS and serves cookies with `Secure`. See
[security-architecture](../architecture/security-architecture.md).

## TCP keys (`tcp`)

| Key | Purpose |
| --- | --- |
| `tcp-handlers-dir` | Directory scanned for `TcpMessageDecoder` / `TcpMessageEncoder` / `TcpMessageHandler` plugin JARs. |

Resolution order for the handlers dir: **system property → `TCP_HANDLERS_DIR` env var →
this server key**. The first concrete plugin implementation wins; otherwise UTF-8 defaults
apply. See [architecture/extensibility.md](../architecture/extensibility.md).

## Queue keys (`queue`)

| Key | Purpose |
| --- | --- |
| `type` | Queue connector type (e.g. `nats`). |
| `consumer-topic` | Topic the server consumes requests from (consumer topic; default `default-topic`). |

The queue transport consumes request envelopes and replies on the reply topic. See
[queue-service.md](queue-service.md) for the connector itself.

## HTTP endpoints & Swagger UI

The HTTP/HTTPS transport exposes built-in endpoints (`/health`, `/me`, `/callback`,
`/logout`, `/stop`, `/openapi.yaml`, `/swagger/**`) in addition to application routes
`/{appId}/[api|page|component|menu]/**`. The full list is in
[reference/http-endpoints.md](../reference/http-endpoints.md).

Swagger UI is configured with the `swaggerUI` block:

| Key | Default | Purpose |
| --- | --- | --- |
| `local-only` | `true` | Restrict Swagger UI to localhost. |
| `openapi-yaml` | — | Path to the OpenAPI document served at `/openapi.yaml`. |

Generate the OpenAPI document with [`pdoc`](../cli-tools/pdoc.md) — see
[developer/api-documentation.md](../developer/api-documentation.md).

## Security headers (HTTP)

The HTTP transport sets hardened headers on responses: `X-Content-Type-Options: nosniff`,
`X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store...`,
`Pragma: no-cache`, and a strict CSP (`default-src 'none'; frame-ancestors 'none';
base-uri 'none'`), plus HSTS on HTTPS. CORS is resolved app → tenant → global (see
[security-oidc.md](security-oidc.md)).

## Next

- [security-oidc.md](security-oidc.md) — authentication & CORS.
- [reference/http-endpoints.md](../reference/http-endpoints.md) — endpoint catalog.
