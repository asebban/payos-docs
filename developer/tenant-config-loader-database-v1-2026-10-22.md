# tenant-config-loader-database — Setup Guide

**Created:** 2026-10-10
**Last updated:** 2026-10-13
**Version:** 4

## 1. What this module does

`tenant-config-loader-database` is an `IConfigLoader` implementation that declares `multitenancy.tenants` entries from rows in a database table, instead of (or in addition to) hand-writing them in `payos.json`/the config directory's JSON files.

It deliberately reuses `platform-database-service`'s own connection details (the same database `$PlatformDB` connects to) rather than requiring a second, dedicated connection config — the tenant registry table is expected to live in the platform database alongside other platform-owned tables.

It does not reuse `$PlatformDB`'s own runtime connection pool or Hibernate session factory — that object does not exist yet at the point in boot where this loader runs. Instead it opens its own short-lived, plain JDBC connection, reads the registry table once, and closes it.

Every mapped field — database connection, `security`, `quota` — has its **own scalar registry column**: there is no JSON blob column anywhere in this design. Each field is independently configurable and independently optional (see §4.2).

A separate, independently opt-in companion (`TenantRegistryChangePoller`, §10) can watch the table for changes and trigger a live reload without a process restart.

## 2. The IConfigLoader mechanism it plugs into

`IConfigLoader` (`ma.s2m.payos.config.IConfigLoader`, `payos-foundation` 2.3.0-RELEASE+) is a one-method SPI: `Map<String, Object> loadConfig(Map<String, Object> currentConfig) throws Exception`.

Implementations are discovered via `ServiceLoader` against `PayOSConfig.getServiceAdapterClassLoader()` — the same classloader `IDatabaseServiceFactory`/`IRuntimeIntegration` use — inside `ConfigLoader.loadServerConfig()` (`payos-kernel`, `payos` repository), right after `ServiceAdapterLoader.initialize(...)` has built that classloader and before `loadApplications(...)`/`loadMultitenancyConfig(...)` run.

Each discovered loader receives the configuration accumulated so far from JSON (bootstrap file, config directory, `custom.json`), and its return value is deep-merged on top via `ConfigMerger.deepMerge(config, contributed)` — the loader's keys win on conflict. After every loader has run, relative paths and JDBC file URLs are re-normalized against the runtime base directory (safe to re-run, this step is idempotent on values already resolved from JSON), so a loader-contributed relative H2 URL gets resolved exactly like a hand-written one.

Running before `loadMultitenancyConfig(...)` matters specifically for this module: a tenant this loader adds still gets the same default `security`/`isolationMode`/`quota` backfill that a JSON-declared tenant gets, for whichever of those blocks this loader did not itself contribute.

A loader that throws is logged (`logger.error`, naming the loader's class) and skipped — never fatal to PayOS boot. A misconfigured `tenant-config-loader` block (see §4) therefore does not prevent PayOS from starting; it just means no tenants are contributed from the database for that boot, visible only via that one error log line.

**Every one of this mechanism's calls happens again, in full, on any config reload** — this is exactly what §10's live-reload polling relies on.

## 3. What this module reuses vs. what it needs of its own

Two separate things are read from two separate places — this is the part most likely to cause confusion, so it is spelled out explicitly here.

**The registry connection** (which database to run `SELECT * FROM <table>` against) comes entirely from the already-configured `platform-database-service` block — no dedicated connection config for this module exists or is needed.

**Each tenant's own data** (its application database connection, its `security`/`quota` settings) is necessarily different per tenant, and therefore cannot come from `platform-database-service` either — it has to be data, read from the registry table's own columns, one row per tenant.

This module's own config block (`tenant-config-loader`, see §4) only ever configures **column name mappings** — which column in your registry table holds which field — never connection details for the registry itself, and never the actual field values.

## 4. Required configuration

### 4.1 `platform-database-service` (must already be configured for this loader to do anything)

```json
{
  "platform-database-service": {
    "configuration": {
      "url": "jdbc:postgresql://platform-host:5432/payos_platform",
      "username": "payos_platform",
      "password": "...",
      "driver_class": "org.postgresql.Driver"
    },
    "mapping-files": ["..."]
  }
}
```

Only `url` and `username` are strictly required by this loader (`password`/`driver_class` are optional — see `TenantConfigLoader.loadConfig`, which reads them via `IConfigSpec.DatabaseService.Configuration.URL`/`USERNAME`/`PASSWORD`/`DRIVER_CLASS_UNDERSCORE`). `driver_class`, if present, is loaded via `Class.forName(...)` before opening the registry connection, to guarantee the driver is registered with `DriverManager` even if it was only just added to the classpath via `service-adapters-dir` moments earlier in the same boot.

If `platform-database-service` is absent while `tenant-config-loader.enabled` is `true`, `loadConfig` throws `IllegalStateException` — caught and logged by `ConfigLoader` as described in §2, not fatal to boot.

### 4.2 `tenant-config-loader` (this module's own block — column names only)

Only `enabled` and `table` are required; every `*Column` key is optional, and most have a default column name so the loader works out of the box against a table that follows the naming below. Overriding any of them only requires naming that one field — every other field keeps its own default.

```json
{
  "tenant-config-loader": {
    "enabled": true,
    "table": "tenant_registry",
    "tenantIdColumn": "tenant_id",

    "schemaColumn": "db_schema",
    "isolationModeColumn": "isolation_mode",

    "dbUrlColumn": "db_url",
    "dbUsernameColumn": "db_username",
    "dbPasswordColumn": "db_password",
    "dbDriverClassColumn": "db_driver_class",
    "dbDialectColumn": "db_dialect",
    "dbDdlAutoColumn": "db_ddl_auto",

    "securityProviderColumn": "security_provider",
    "securityEncryptionKeyColumn": "security_encryption_key",
    "securityOidcProviderBaseUrlColumn": "security_oidc_provider_base_url",
    "securityRealmColumn": "security_realm",
    "securityDiscoveryUriColumn": "security_discovery_uri",
    "securityScopeColumn": "security_scope",
    "securityClientIdColumn": "security_client_id",
    "securityClientSecretColumn": "security_client_secret",
    "securityCallbackUriColumn": "security_callback_uri",
    "securityPreferredJwsAlgorithmColumn": "security_preferred_jws_algorithm",
    "securityLogoutUrlColumn": "security_logout_url",
    "securityPostLogoutRedirectUriColumn": "security_post_logout_redirect_uri",
    "securitySessionTtlSecondsColumn": "security_session_ttl_seconds",
    "securitySessionMaxEntriesColumn": "security_session_max_entries",
    "securitySessionCookieSecureColumn": "security_session_cookie_secure",
    "securityAllowedOriginsColumn": "security_allowed_origins",

    "quotaEnabledColumn": "quota_enabled",
    "quotaRequestsPerMinuteColumn": "quota_requests_per_minute",

    "pollEnabled": false,
    "pollIntervalSeconds": 10,
    "changeDetectionColumn": "updated_at"
  }
}
```

Field reference (constants live in `ITenantConfigLoaderSpec.TenantConfigLoader`, `tenant-config-loader-database` module):

| Field | Required | Default column if omitted |
|---|---|---|
| `enabled` | yes | — (absent or not exactly `true` means the loader contributes nothing) |
| `table` | yes | — no default, `loadConfig` throws `IllegalStateException` if missing |
| `tenantIdColumn` | no | `tenant_id` |
| `schemaColumn` | no | none — omit to leave `schema` unset |
| `isolationModeColumn` | no | none — omit to leave `isolationMode` unset |
| `dbUrlColumn` | no | `db_url` |
| `dbUsernameColumn` | no | `db_username` |
| `dbPasswordColumn` | no | none |
| `dbDriverClassColumn` | no | `db_driver_class` |
| `dbDialectColumn` | no | none |
| `dbDdlAutoColumn` | no | none |
| `securityProviderColumn` | no | `security_provider` |
| `securityEncryptionKeyColumn` | no | `security_encryption_key` |
| `securityOidcProviderBaseUrlColumn` | no | `security_oidc_provider_base_url` |
| `securityRealmColumn` | no | `security_realm` |
| `securityDiscoveryUriColumn` | no | `security_discovery_uri` |
| `securityScopeColumn` | no | `security_scope` |
| `securityClientIdColumn` | no | `security_client_id` |
| `securityClientSecretColumn` | no | `security_client_secret` |
| `securityCallbackUriColumn` | no | `security_callback_uri` |
| `securityPreferredJwsAlgorithmColumn` | no | `security_preferred_jws_algorithm` |
| `securityLogoutUrlColumn` | no | `security_logout_url` |
| `securityPostLogoutRedirectUriColumn` | no | `security_post_logout_redirect_uri` |
| `securitySessionTtlSecondsColumn` | no | `security_session_ttl_seconds` |
| `securitySessionMaxEntriesColumn` | no | `security_session_max_entries` |
| `securitySessionCookieSecureColumn` | no | `security_session_cookie_secure` |
| `securityAllowedOriginsColumn` | no | `security_allowed_origins` (comma-separated, see §4.3) |
| `quotaEnabledColumn` | no | `quota_enabled` |
| `quotaRequestsPerMinuteColumn` | no | `quota_requests_per_minute` |
| `pollEnabled` | no | `false` — see §10 |
| `pollIntervalSeconds` | no | `10` — see §10 |
| `changeDetectionColumn` | no | `updated_at` — see §10 |

Every field is read independently: a row with only `db_url`/`db_username` filled in and every `security_*`/`quota_*` column blank contributes a tenant with no `security`/`quota` block at all (not an empty one) — `loadMultitenancyConfig` then applies the bootstrap-level default for that whole block, exactly as for a JSON-declared tenant that never set it either.

If the table doesn't have a matching column for one of the optional fields, simply omit that field's key from this block — `TenantConfigLoader` never fails on a missing optional column, it just leaves that field out of the contributed tenant config.

### 4.3 `securityAllowedOriginsColumn` is a comma-separated list, not a scalar

Unlike every other field, `allowedOrigins` must be a `List<String>` in the final config — its only consumer, `HttpServer#isOriginAllowedBySecurityConfig`, requires an actual `Collection` and will silently reject a plain string. The loader therefore splits this one column's value on commas, trims each entry, and stores a `List<String>`: `"https://a.example.com, https://b.example.com"` becomes `["https://a.example.com", "https://b.example.com"]`.

### 4.4 Values are read as plain strings — downstream parsing is lenient

Every column (including numeric/boolean fields like `sessionTtlSeconds`, `sessionMaxEntries`, `sessionCookieSecure`, `quota.enabled`, `quota.requestsPerMinute`) is read via a plain JDBC `getString(...)` and stored as a `String`. This is safe: every consumer of these fields (`PayOSSessionStore#getSessionTtlSeconds`/`getSessionMaxEntries`/`resolveSessionCookieSecure`, `DatabaseServiceInitializer#booleanOrDefault`/`integerOrDefault`) already parses them leniently, accepting either a native JSON-typed value (`Integer`/`Boolean`, from a hand-written `payos.json`) or its string representation — a column holding `"true"`/`"1200"` works exactly the same as a hand-written `true`/`1200`.

## 5. Expected table shape and a worked example

The loader runs a plain `SELECT * FROM <table>` — no filtering, every row becomes one tenant entry. Worked example matching every default column name above, plus the `updated_at` column §10's poller needs:

```sql
CREATE TABLE tenant_registry (
    tenant_id                         VARCHAR(64)  NOT NULL,
    db_schema                         VARCHAR(64),
    isolation_mode                    VARCHAR(32),
    db_url                            VARCHAR(256) NOT NULL,
    db_username                       VARCHAR(64)  NOT NULL,
    db_password                       VARCHAR(64),
    db_driver_class                   VARCHAR(128),
    db_dialect                        VARCHAR(128),
    db_ddl_auto                       VARCHAR(32),
    security_provider                 VARCHAR(64),
    security_encryption_key           VARCHAR(64),
    security_oidc_provider_base_url   VARCHAR(256),
    security_realm                    VARCHAR(64),
    security_discovery_uri            VARCHAR(256),
    security_scope                    VARCHAR(128),
    security_client_id                VARCHAR(64),
    security_client_secret            VARCHAR(64),
    security_callback_uri             VARCHAR(256),
    security_preferred_jws_algorithm  VARCHAR(16),
    security_logout_url               VARCHAR(256),
    security_post_logout_redirect_uri VARCHAR(256),
    security_session_ttl_seconds      VARCHAR(16),
    security_session_max_entries      VARCHAR(16),
    security_session_cookie_secure    VARCHAR(16),
    security_allowed_origins          VARCHAR(512),
    quota_enabled                     VARCHAR(16),
    quota_requests_per_minute         VARCHAR(16),
    updated_at                        TIMESTAMP
);
```

Only `tenant_id`/`db_url`/`db_username` need `NOT NULL` in practice — every other column may be left `NULL`/blank per tenant, and that field (or, if every `security_*`/`quota_*` column is blank, the whole block) is simply omitted for that tenant.

A row with a blank/null `tenant_id` (or whatever column `tenantIdColumn` names) is skipped with a warning log, not an error — it never aborts the whole load. If the table has zero matching rows, the loader contributes nothing (same as if it were disabled).

## 6. What ends up in the final PayOS config

For a `tenant3` row with `db_schema: "public"`, `isolation_mode: "shared-schema"`, every `security_*` column filled in, `security_logout_url` blank, `security_allowed_origins: "https://a.example.com, https://b.example.com"`, `quota_enabled: "true"`, `quota_requests_per_minute: "1200"`, and the `db_*` columns set, this loader contributes:

```json
{
  "multitenancy": {
    "tenants": {
      "tenant3": {
        "schema": "public",
        "isolationMode": "shared-schema",
        "security": {
          "encryptionKey": "rb50a5pXrgw6a4qx",
          "oidcProviderBaseUrl": "http://127.0.0.1:8080",
          "realm": "master",
          "discoveryUri": "http://127.0.0.1:8080/realms/master/.well-known/openid-configuration",
          "clientId": "test_client",
          "clientSecret": "TX3otV9o2yhZvA8FYDRooQz6p60DzLVA",
          "callBackUri": "http://127.0.0.1:8081/callback",
          "preferredJwsAlgorithm": "RS256",
          "postLogoutRedirectUri": "http://127.0.0.1:3000/",
          "allowedOrigins": ["https://a.example.com", "https://b.example.com"]
        },
        "quota": {
          "enabled": "true",
          "requestsPerMinute": "1200"
        },
        "database-service": {
          "configuration": {
            "url": "...",
            "username": "...",
            "driver-class": "..."
          }
        }
      }
    }
  }
}
```

Note `logoutUrl` is absent (the column was blank for this row) and `allowedOrigins` is a real JSON array, not a string — see §4.3.

This whole tenant entry is deep-merged on top of whatever `multitenancy` the JSON config already declared (see §2) — a JSON-declared tenant with the same ID would be overridden field-by-field by what this loader contributes, since loader keys win on conflict.

## 7. Deploying this module

1. Build the jar: `mvn clean package` (or `mvn install` to also publish it to the local `.m2` repository) from this module's root.
2. Copy `target/tenant-config-loader-database-1.0.0-RELEASE.jar` into the `service-adapters-dir` configured for the PayOS instance (the same directory used for `IDatabaseServiceFactory`/other service-adapter jars — see `service-adapters-dir` in `payos.json`/bootstrap config).
3. No further registration step is needed — the module already ships `META-INF/services/ma.s2m.payos.config.IConfigLoader` (naming `TenantConfigLoader`) and `META-INF/services/ma.s2m.payos.config.IRuntimeIntegration` (naming `TenantRegistryChangePoller`, §10), both picked up automatically by `ServiceLoader` at boot.

## 8. Version dependencies

This module requires `payos-foundation` 2.3.0-RELEASE or later (for `IConfigLoader`) and a `payos-kernel` build that wires the `IConfigLoader` SPI into `ConfigLoader.loadServerConfig()` (2.6.0-RELEASE or later). No other runtime dependency beyond what is already `provided` by `payos-kernel` (`slf4j-api`) — there is no JSON parsing in this module, every field is read as a plain JDBC string.

## 9. Test coverage

`TenantConfigLoaderTest` (H2 in-memory, 7 tests) covers: no contribution when the `tenant-config-loader` block is absent, no contribution when `enabled` is not `true`, a full worked row mapping every database-connection field (with `security`/`quota` correctly absent when their columns are blank), a full worked row mapping every `security`/`quota` field from its own column — including a blank cell being omitted and the comma-separated `allowedOrigins` column becoming a real list, rows with a blank tenant ID being skipped without aborting the rest, no contribution when the table is empty, and `IllegalStateException` when `platform-database-service` itself isn't configured.

`TenantRegistryChangePollerTest` (H2 in-memory, 4 tests) covers: a real registry update being detected and triggering the sentinel-file write within one poll interval, no trigger when nothing changes across several polls, and the poller not starting at all when either `pollEnabled` or the loader's own `enabled` is missing/false.

## 10. Live reload — `TenantRegistryChangePoller`

### 10.1 What it does and why it's a separate component

`TenantConfigLoader` itself only runs when something else asks PayOS to (re)load its configuration — it has no notion of "watching" anything on its own. `TenantRegistryChangePoller` is a second, independent `IRuntimeIntegration` (the same SPI `payos-job-scheduler`'s embedded-mode hook uses) that runs a daemon background thread for the life of the process, so a change made directly in the database — not through any PayOS API — still takes effect without a restart.

It is deliberately **not** a new reload mechanism. PayOS's kernel already hot-reloads its entire configuration whenever a `.json` file changes inside the bundle's config directory (`BootServer`'s `ConfigWatcher`/`reloadConfiguration()`), and that reload already re-invokes every `IConfigLoader` — this module's `TenantConfigLoader` included, with zero extra wiring. The poller's only job is to notice a database change and rewrite a small, dedicated sentinel file (`tenant-config-loader-poll-trigger.json`, content always `{}`) inside that same watched config directory — which is all it takes to make `ConfigWatcher` fire and the whole config, including this tenant's row, reload.

### 10.2 Enabling it

Opt-in, independently of `enabled` (§4.2) — a poller thread running for the life of the process is a bigger commitment than a one-shot read at boot, so it is off by default even when the loader itself is on:

```json
{
  "tenant-config-loader": {
    "enabled": true,
    "table": "tenant_registry",
    "pollEnabled": true,
    "pollIntervalSeconds": 10,
    "changeDetectionColumn": "updated_at"
  }
}
```

`changeDetectionColumn` must be a column whose value increases whenever any row in the table changes — a `TIMESTAMP`/`updated_at` column maintained by whoever writes the table (application code, a trigger, or manual SQL — the poller doesn't care how it gets set, only that it changed) is the natural choice; a `version`/sequence column works identically. The poller runs `SELECT MAX(changeDetectionColumn) FROM table` every `pollIntervalSeconds` and compares it to what it saw last time — not a row-by-row diff, so this stays cheap regardless of table size.

### 10.3 What "immediately" actually means here

Reload latency is bounded by `pollIntervalSeconds`, not instantaneous — a true push-based trigger (e.g. PostgreSQL's `LISTEN`/`NOTIFY`) would need a database-specific mechanism, which conflicts with this module's deliberately database-agnostic, plain-JDBC design (see §1). A 10-second poll interval means a change can take up to ~10 seconds (plus however long the subsequent full reload itself takes) to take effect — in practice, lower `pollIntervalSeconds` to a few seconds for near-real-time behavior, at the cost of more frequent `SELECT MAX(...)` queries.

### 10.4 This is a full config reload, not a per-tenant one

`reloadConfiguration()` reinitializes PayOS's entire runtime — every tenant's database pool, the queue/secret/notification clients, webhook dispatcher, connector framework, etc. — not just the one tenant whose row changed, because that is the only reload granularity PayOS's kernel currently exposes. For a handful of tenants this is inexpensive; for a very large registry or a very short `pollIntervalSeconds`, the reload cost itself (not just the poll query) is worth measuring before tuning the interval down aggressively.

A more surgical, single-tenant-only reload (touching just that tenant's own database connection, nothing else) was explored and implemented, but reverted as too complex for the value it added — see this module's own commit history if that tradeoff needs revisiting later.

### 10.5 The first poll never triggers a reload

The poller records a baseline on its very first tick and only compares against that afterwards — otherwise, every process start would immediately trigger a redundant reload of configuration that was already current at boot.
