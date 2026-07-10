# Configuration reference

This section is the authoritative reference for **every** PayOS configuration key, organized
by block. `payos.json` is the bundle entrypoint and normally contains `configDir`; the actual
runtime blocks are loaded by merging the JSON files under that directory (typically
`config/bootstrap.json`) and then exposed through `PayOSConfig.settings`. The loading
pipeline and hot-reload behavior are described in
[architecture/runtime-architecture.md](../architecture/runtime-architecture.md) and
[operations/hot-reload.md](../operations/hot-reload.md).

## How configuration is loaded

```
payos.json
  → CryptoService.decryptIfEncrypted()      (decrypt encrypted values)
  → EnvVarResolver.resolve()                (resolve ${ENV_VAR} placeholders)
  → read configDir                          (usually "config")
  → merge JSON files from configDir         (for example bootstrap.json)
  → PayOSConfig.settings (volatile Map)     (the live registry)
```

All keys below are defined as constants in `ma.s2m.payos.config.IConfigSpec`.

## Blocks

| Block | Document | Purpose |
| --- | --- | --- |
| Bootstrap (entrypoint + top level) | [bootstrap-reference.md](bootstrap-reference.md) | `payos.json`, `bootstrap.json`, top-level runtime blocks, runtime dirs. |
| Environment variables & config references | [env-var-resolution.md](env-var-resolution.md)<br>[config-references.md](config-references.md) | Placeholder syntax (`${...}`) for environment variables, files, and config keys. |
| `servers` | [servers.md](servers.md) | Transport listeners (HTTP/HTTPS/TCP/queue), TLS, Swagger UI. |
| `security` | [security-oidc.md](security-oidc.md) | OIDC/pac4j authentication, sessions, CORS. |
| `multitenancy` | [multi-tenancy.md](multi-tenancy.md) | Tenant policy, quotas, isolation, simulator. |
| `database-service` | [database-service.md](database-service.md) | JDBC/Hibernate connection and pooling. |
| `queue-service` | [queue-service.md](queue-service.md) | MoM connector (e.g. NATS). |
| `notification-service` | [notification-service.md](notification-service.md) | Publisher-side `$Notification` connector — independent from `queue-service`. |
| `secret-service` | [secret-service.md](secret-service.md) | Secret provider (`filesystem`/`vault`). |
| `webhooks` / `http-webhook-service` | [webhook-service.md](webhook-service.md) | Webhook dispatcher. |
| `i18n` | [i18n.md](i18n.md) | Locale resolution. |
| `connectors-dir` / `extensions-dir` | [extensions-connectors.md](extensions-connectors.md) | Plugin discovery paths and classloaders (legacy SPI-backend loader). |
| Connector framework (`connectors.json`, connector descriptor) | [connector-framework-parameters-v1-2026-07-10.md](connector-framework-parameters-v1-2026-07-10.md) | Business/payment connector plugin system — `connectors.json`, `META-INF/connector.properties`, credential references, hot-reload, tenant scoping. Not yet wired into `BootServer`. |
| Complete reference guide of configuration | [Json configuration reference](./json-configuration-reference.md) | |

## A complete index

For a single flat list of every key (block + key + default), see
[reference/configuration-keys.md](../reference/configuration-keys.md).

## Conventions

- **Convention over configuration:** most keys have sensible defaults; only override what you
  need.
- **Environment placeholders:** any string value may contain `${ENV_VAR}`, resolved at load.
  See [Environment Variable Resolution](env-var-resolution.md) for full syntax.
- **Config references:** any string value may reference other config keys using dot notation
  (e.g., `${database.host}` or `${config:server.port}`). This allows reusing values across
  the configuration. See [Configuration References](config-references.md) for details.
- **Encrypted values:** sensitive values can be stored encrypted and are decrypted at load
  (see [operations/bundle-encryption.md](../operations/bundle-encryption.md)).
- **Hot reload:** edits to watched config files are applied without a restart
  (see [operations/hot-reload.md](../operations/hot-reload.md)).
