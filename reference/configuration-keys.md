# Configuration keys index

A flat index of every PayOS configuration key, grouped by block. Defined as constants in
`ma.s2m.payos.config.IConfigSpec`. For explanations and examples, follow the link in each
block heading.

## Top level — [bootstrap-reference.md](../configuration/bootstrap-reference.md)

| Key | Default | Purpose |
| --- | --- | --- |
| `runtimeBaseDir` | directory containing `payos.json` | Effective runtime base directory; computed by the loader, not normally authored in `payos.json`. |
| `configDir` | `.` | Directory merged into the configuration, resolved relative to `runtimeBaseDir` when not absolute. |
| `config-hot-reload-enabled` | `true` | Enable configuration hot-reload (watches config directories for changes). |
| `connectors-dir` | — | Connector (SPI) JAR directory. |
| `extensions-dir` | — | Extension JAR directory. |
| `tcp-handlers-dir` | — | TCP plugin directory (also per-server). |
| `applications[]` | — | Registered applications (see below). |
| `applicationCatalog` | — | Optional app catalog (`local`/`git`, `baseUrl`/`path`). |
| `capabilityCatalog` | — | Optional capability catalog (`local`/`git`, `baseUrl`/`path`). |
| `productCatalog` | — | Optional product catalog (`local`/`git`, `baseUrl`/`path`). |

Internal/effective: `RUNTIME_CONFIG_FILE = payos.json`, `RUNTIME_BASE_DIRECTORY = runtimeBaseDir`, `CAPABILITIES_DIR = .capabilities`.

### `applications[]` entry

| Key | Default | Purpose |
| --- | --- | --- |
| `id` (required) | — | Unique id; first URI segment. |
| `name` | — | Display name. |
| `base.path` | `.apps/{id}` | Application directory, resolved relative to `configDir`. |
| `version` | — | Semantic version. |
| `category` | `application` | `application` / `capability`. |
| `extends` | — | Parent app/capability id(s). |
| `authorized-tenants` / `authorized.tenants` | — | Tenant allowlist. |
| `defaultUrl` | — | Post-login redirect. |
| `mapping-files` | — | Data-model mappings. |
| `security` | inherits global | Per-app security overrides. |
| `database-service` | inherits global | Per-app DB overrides. |

## `servers[]` — [servers.md](../configuration/servers.md)

| Key | Purpose |
| --- | --- |
| `protocol` | `http`/`https`/`tcp`/`queue`. |
| `host` | Bind address. |
| `port` | Bind port. |
| `keystore` / `keystorePassword` / `keystoreType` / `keyAlias` / `keyPassword` | TLS (https). |
| `tcp-handlers-dir` | TCP plugin dir (tcp). |
| `queueClient` | Queue connector type (queue). |
| `server-topic` | Consumer topic (queue; default `default-topic`). |

### `swaggerUI`

| Key | Default | Purpose |
| --- | --- | --- |
| `local-only` | `true` | Restrict Swagger UI to localhost. |
| `openapi-yaml` | — | OpenAPI document path. |

## `security` — [security-oidc.md](../configuration/security-oidc.md)

| Key | Purpose |
| --- | --- |
| `provider` | `nimbus` (default) / `pac4j` (legacy). |
| `oidcProviderBaseUrl` / `keycloakBaseUrl` | IdP base URL. |
| `realm` | IdP realm. |
| `discoveryUri` | OIDC discovery URL. |
| `clientId` / `clientSecret` | OIDC client. |
| `callBackUri` | Redirect URI. |
| `scope` | Requested scopes. |
| `preferredJwsAlgorithm` | Token signing alg. |
| `logoutUrl` / `postLogoutRedirectUri` | Logout. |
| `sessionTtlSeconds` / `sessionMaxEntries` / `sessionCookieSecure` | Sessions. |
| `allowedOrigins` | CORS origins. |

## `multitenancy` — [multi-tenancy.md](../configuration/multi-tenancy.md)

| Key | Default | Purpose |
| --- | --- | --- |
| `requireTenantId` | `true` | Reject requests with no tenant. |
| `default-database-schema` | — | Default schema. |
| `default-isolation-mode` | — | Default isolation. |
| `default-tenant-quotas.requestsPerMinute` | — | Default rate limit. |
| `default-tenant-quotas.enabled` | — | Enforce default quotas. |
| `tenantSimulator.enabled` | `false` | Dev-only fixed tenant. |
| `tenantSimulator.tenantId` | — | Simulated tenant id. |
| `tenants[]` | — | Per-tenant overrides (see below). |

### `tenants[]` entry

| Key | Default | Purpose |
| --- | --- | --- |
| `id` (required) | — | Unique tenant identifier. |
| `database-schema` / `schema` | inherits `default-database-schema` | Database schema for this tenant. |
| `isolation-mode` / `isolationMode` | inherits `default-isolation-mode` | Isolation mode. |
| `tcp.handlers.dir` | — | Per-tenant TCP plugin directory. |
| `quotas.requestsPerMinute` | inherits default | Requests per minute limit. |
| `quotas.enabled` | inherits default | Enforce quota. |
| `security` | inherits global | Per-tenant OIDC overrides (all keys from global `security`). |
| `database-service` | inherits global | Per-tenant DB config (all keys from global `database-service`). |

## `database-service` — [database-service.md](../configuration/database-service.md)

| Key | Purpose |
| --- | --- |
| `dialect` | Hibernate dialect. |
| `driver_class` | JDBC driver. |
| `url` | JDBC URL. |
| `username` / `password` | Credentials. |
| `schema` | Default schema. |
| `max-pool-size` / `minimum-idle` | Pool sizing. |
| `ddl-auto` | Schema management. |
| `retired-session-factory-close-delay-seconds` | Graceful retire delay on reload. |

## `queue-service` — [queue-service.md](../configuration/queue-service.md)

| Key | Default | Purpose |
| --- | --- | --- |
| `name` | — | Logical name. |
| `type` | — | Connector type (e.g. `nats`). |
| `host` | `localhost` | Broker host. |
| `port` | `4222` | Broker port. |
| `publisher-topic` | — | Default publish topic. |
| `consumer-topic` | `default-topic` | Topic consumed by the queue transport. |

## `notification-service` — [notification-service.md](../configuration/notification-service.md)

| Key | Default | Purpose |
| --- | --- | --- |
| `type` | connector-derived (`nats`) | Connector type; selects the `IQueueClientFactory` used by the notification connector's own connection. |
| `host` | `localhost` | Notification broker host. |
| `port` | `4222` | Notification broker port. |
| `topic` | `payos.notifications` | Topic used for the connector's own connection. |

Independent of `queue-service` — the notification connector owns its own broker connection
rather than reusing `$Queue`'s client.

## `secret-service` — [secret-service.md](../configuration/secret-service.md)

Keys below live under `secret-service.configuration.*`.

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | — | Enable `$Secrets`. |
| `type` | — | `filesystem` / `vault`. |
| **filesystem** `root` | `secrets` | Storage root. |
| **filesystem** `keyfile` | — | Master key file (or `PAYOS_SECRET_MASTER_KEY`). |
| **vault** `address` | — | Vault server. |
| **vault** `token` | — | Static token. |
| **vault** `role-id` / `secret-id` | — | AppRole (precedence over token). |
| **vault** `approle-mount` | `approle` | AppRole mount. |
| **vault** `kv-mount` | `secret` | KV v2 mount. |
| **vault** `namespace` | — | Vault namespace. |
| **vault** `tls-skip-verify` | `false` | Skip TLS verify. |
| **vault** `timeout` | `10` | HTTP timeout (s). |

## `webhooks` / `http-webhook-service` — [webhook-service.md](../configuration/webhook-service.md)

| Key | Default | Purpose |
| --- | --- | --- |
| `webhooks.dispatcher` | `http` | Dispatcher type. |
| `http-webhook-service.enabled` | `true` | Enable HTTP dispatcher. |
| `http-webhook-service.connectTimeoutMs` | `5000` | Connect timeout. |
| `http-webhook-service.requestTimeoutMs` | `10000` | Request timeout. |

Per-app subscription fields (`config/webhooks.json`): `id`, `event`, `native`, `url`,
`secret`, `method`, `headers`, `disabled`, `filter.path`, `filter.method`,
`filter.statusCodes`.

## `i18n` — [i18n.md](../configuration/i18n.md)

| Key | Purpose |
| --- | --- |
| `headerName` | Locale header. |
| `defaultLocale` | Default locale. |
| `fallbackLocale` | Fallback locale. |
| `supportedLocales` | Allowed locales. |

## Plugin discovery env vars / system props — [extensions-connectors.md](../configuration/extensions-connectors.md)

| Bootstrap key | Env var |
| --- | --- |
| `connectors-dir` | `PAYOS_CONNECTORS_DIR` |
| `extensions-dir` | `PAYOS_EXTENSIONS_DIR` |
| `tcp-handlers-dir` | `TCP_HANDLERS_DIR` |
| (filesystem secrets master key) | `PAYOS_SECRET_MASTER_KEY` |

Resolution order for each: system property → env var → bootstrap/server key.
