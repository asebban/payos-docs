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
| Environment variables & config references | [env-var-resolution.md](env-var-resolution.md)<br>[config-references.md](config-references.md)<br>[example-multi-env.md](example-multi-env.md) | Placeholder syntax (`${...}`) for environment variables, files, and config keys, plus a worked multi-environment example. |
| `servers` | [servers.md](servers.md) | Transport listeners (HTTP/HTTPS/TCP/queue), TLS, Swagger UI. |
| License validation (`PAYOS_LICENSE_DEV_MODE`, `PAYOS_LICENSE_FILE_PATH`, …) | [license-service-v2-2026-11-01.md](license-service-v2-2026-11-01.md) | Dev/prod validator selection, the fixed `*.lic` file convention, and the editor secret provider env vars — all validated first at `BootServer` startup, before any other service (and, for the real validator, before `payos.json` itself) is read. The `license-file-path` JSON key is no longer used. |
| `security` | [security-oidc.md](security-oidc.md)<br>[oidc-configuration-guide.md](oidc-configuration-guide.md) | OIDC/pac4j authentication, sessions (including the pluggable `sessionStoreType`/Redis distributed backend), CORS; the second link is the full walkthrough guide (providers, precedence, discovery URI resolution, quickstart). |
| `multitenancy` | [multi-tenancy.md](multi-tenancy.md) | Tenant policy, quotas, isolation, simulator. |
| `tenant-config-loader` | [tenant-config-loader-database.md](tenant-config-loader-database-v1-2026-10-22.md) | Optional module: declares `multitenancy.tenants` entries from a database table instead of (or on top of) hand-written JSON, with an opt-in live-reload poller. |
| `database-service` | [database-service.md](database-service.md) | JDBC/Hibernate connection and pooling. |
| `platform-database-service` | [platform-database-service.md](platform-database-service.md) | Optional second, tenant-independent database backing `$PlatformDB` — reference/master data shared by every tenant. |
| `queue-service` | [queue-service.md](queue-service.md) | MoM connector (e.g. NATS). |
| `notification-service` | [notification-service.md](notification-service.md) | Publisher-side `$Notification` connector — independent from `queue-service`. |
| `secret-service` | [secret-service.md](secret-service.md) | Secret provider (`filesystem`/`vault`). |
| `editor-secret-service` | [editor-secret-service.md](editor-secret-service.md) | Separate secret provider backing the bundle-wide `encryptionKey` (`CryptoService`/`IEditorProvider`) — deliberately distinct from `secret-service`. |
| `webhooks` / `http-webhook-service` | [webhook-service.md](webhook-service.md) | Webhook dispatcher. |
| `i18n` | [i18n.md](i18n.md) | Locale resolution. |
| `idempotency` | [idempotency.md](idempotency.md) | Idempotent-response cache for mutating API requests. |
| `audit-trail` | [audit-trail.md](audit-trail.md) | Pluggable, durable PCI-DSS audit-trail: delivery category (`buffered-store` or `database`), store backend, business-keys allowlist. Optional — a durable default activates with zero configuration. |
| `cache-service` | [cache-service.md](cache-service.md) | Optional distributed cache abstraction (`ICacheStore`), shared across instances/bundles — `memory` or `redis` backend. |
| `sliding-window-service` | [sliding-window-service.md](sliding-window-service.md) | Optional exact sliding-window event counter (`ISlidingWindowCounter`) for quota/rate-limit checks — `memory` or `redis` backend. |
| `service-adapters-dir` / `extensions-dir` | [extensions-connectors.md](extensions-connectors.md) | Plugin discovery paths and classloaders (legacy SPI-backend loader). |
| Connector framework (`connectors.json`, connector descriptor) | [connector-framework-parameters-v3-2026-08-11.md](connector-framework-parameters-v3-2026-08-11.md) | Business/payment connector plugin system — `connectors.json`, `META-INF/connector.properties`, credential references, hot-reload, tenant scoping, idempotency/dedup, retry policy, execution-state persistence, DLQ/terminal routing, diagnostics. Wired into `BootServer` since 2026-07-27. |
| `task-scheduler` *(embedded mode only; standalone TSS uses its own `tss-config.json`)* | [task-scheduling-system-v11-2026-11-15.md](task-scheduling-system-v11-2026-11-15.md) | `payos-job-scheduler` ("TSS"): its own schema connection, scheduler sizing, the per-tenant execution concurrency quota, `jobs.json`'s per-tenant policy fields and per-job `arguments`, standalone TSS running PayOS config loaders (e.g. a database tenant registry), jobs hot reload, the secured administration endpoint (port and roles), `jobs/template.json` with its `{{tenantId}}` placeholder, the embedded-mode `task-scheduler` block, TSS's own broker connection for `queue` triggers (independent of the host's own queue connection, with an optional directory of extra connector JARs), and the event-trigger refresh interval. |
| *(not a `bootstrap.json` block — its own standalone config, `license-generator-config.json`)* | [payos-license-generator-v5-2026-11-01.md](payos-license-generator-v5-2026-11-01.md) | `payos-license-generator`: the standalone license-issuing service's secret-provider SPI selection, editor key-pair identity, license output directory, HTTP port, structured JSON logging, optional persistence of generation attempts to any JDBC database (H2 by default), and an always-on Prometheus `/metrics` endpoint. |
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
