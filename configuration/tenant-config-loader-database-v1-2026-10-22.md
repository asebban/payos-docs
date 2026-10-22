Created: 2026-10-22
Last updated: 2026-10-22
Version: 1

# `tenant-config-loader` — database-backed tenant registry

`tenant-config-loader-database` is a separate, optional module (`ma.s2m.payos:tenant-config-loader-database`) that declares `multitenancy.tenants` entries from rows in a database table, instead of — or on top of — hand-written entries in `payos.json`/the config directory. It implements the generic `IConfigLoader` SPI; see [architecture/tenant-config-loader-database.md](../architecture/tenant-config-loader-database-v1-2026-10-22.md) for how that SPI fits into the boot sequence, and [developer/tenant-config-loader-database.md](../developer/tenant-config-loader-database-v1-2026-10-22.md) for a worked table/script example and the full column-mapping reference.

## Enabling it

```json
{
  "tenant-config-loader": {
    "enabled": true,
    "table": "tenant_registry"
  }
}
```

`table` is the only other required key once `enabled` is `true`. Absent, or `enabled` anything other than `true`, is a fully-supported "not configured" case — the loader contributes nothing and the rest of boot proceeds exactly as if the module weren't installed.

## Prerequisite: `platform-database-service`

The loader reuses `platform-database-service.configuration`'s connection details (`url`, `username`, `password`, `driver_class`) to open a short-lived JDBC connection to the registry table — it does **not** read a dedicated connection block of its own. This is deliberate reuse, not a requirement that the table live inside `$PlatformDB`'s own runtime connection pool (that pool does not exist yet at the point in boot this loader runs — see the architecture doc). If `platform-database-service` is not configured at all, the loader throws and is skipped (logged, never fatal to boot — see [Fatal vs. non-fatal](../architecture/tenant-config-loader-database-v1-2026-10-22.md#fatal-vs-non-fatal)).

## Deployment

Ship the module's jar (`tenant-config-loader-database-<version>.jar`, built from this module's own `pom.xml`) in the bundle's `service-adapters-dir` — the same directory `payos-database-audit-trail`/`queue-service-nats` use, discovered the same way (`ServiceLoader` against `PayOSConfig.getServiceAdapterClassLoader()`). No kernel code change, no `connectors.json` entry — dropping the jar in that directory is the entire installation step.

## Full key reference

All keys live under the `tenant-config-loader` block. Every `*Column` key names a column in the registry table (`table`); a key with no default must be set explicitly or that field is left out of the contributed tenant config (not an error).

### Core

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `false` | Turns the loader on. Everything below is irrelevant while this is `false`/absent. |
| `table` | *(required)* | The registry table name, read as `SELECT * FROM <table>`. |
| `tenantIdColumn` | `tenant_id` | Column holding the tenant id — becomes the `multitenancy.tenants` map key. A row with a blank/null value here is skipped with a `WARN` log, not fatal. |
| `schemaColumn` | *(none — optional)* | Maps to `multitenancy.tenants.<id>.schema`. |
| `isolationModeColumn` | *(none — optional)* | Maps to `multitenancy.tenants.<id>.isolationMode`. |
| `dbUrlColumn` | `db_url` | Maps to `multitenancy.tenants.<id>.database-service.configuration.url`. |
| `dbUsernameColumn` | `db_username` | Maps to `...configuration.username`. |
| `dbPasswordColumn` | *(none — optional)* | Maps to `...configuration.password`. |
| `dbDriverClassColumn` | `db_driver_class` | Maps to `...configuration.driver-class`. |
| `dbDialectColumn` | *(none — optional)* | Maps to `...configuration.dialect`. |
| `dbDdlAutoColumn` | *(none — optional)* | Maps to `...configuration.ddl-auto`. |

A tenant row that supplies none of the `db*Column` fields simply contributes no `database-service` override for that tenant — it falls back to the bundle's root `database-service` default, same as a JSON-declared tenant with no override block.

### `security.*` (one column per field)

| Key | Default column |
| --- | --- |
| `securityProviderColumn` | `security_provider` |
| `securityEncryptionKeyColumn` | `security_encryption_key` |
| `securityOidcProviderBaseUrlColumn` | `security_oidc_provider_base_url` |
| `securityRealmColumn` | `security_realm` |
| `securityDiscoveryUriColumn` | `security_discovery_uri` |
| `securityScopeColumn` | `security_scope` |
| `securityClientIdColumn` | `security_client_id` |
| `securityClientSecretColumn` | `security_client_secret` |
| `securityCallbackUriColumn` | `security_callback_uri` |
| `securityPreferredJwsAlgorithmColumn` | `security_preferred_jws_algorithm` |
| `securityLogoutUrlColumn` | `security_logout_url` |
| `securityPostLogoutRedirectUriColumn` | `security_post_logout_redirect_uri` |
| `securitySessionTtlSecondsColumn` | `security_session_ttl_seconds` |
| `securitySessionMaxEntriesColumn` | `security_session_max_entries` |
| `securitySessionCookieSecureColumn` | `security_session_cookie_secure` |
| `securityAllowedOriginsColumn` | `security_allowed_origins` — comma-separated in the cell (e.g. `https://a.example.com,https://b.example.com`), split/trimmed into a list; this is the one field with no lenient single-string fallback downstream. |

Every `security.*` field is independently optional: a column that is unset or blank for a given row is simply left out of that tenant's `security` block. The whole block is only omitted if **none** of these columns produced anything for that row — same as a JSON-declared tenant with no `security` key at all.

### `quota.*` (one column per field)

| Key | Default column |
| --- | --- |
| `quotaEnabledColumn` | `quota_enabled` |
| `quotaRequestsPerMinuteColumn` | `quota_requests_per_minute` |

Same independently-optional, whole-block-omitted-if-empty semantics as `security.*` above.

### Live reload (`TenantRegistryChangePoller`)

Off by default, even when the loader itself is enabled — this starts a background thread and periodically opens connections for the life of the process, a second, independent opt-in.

| Key | Default | Purpose |
| --- | --- | --- |
| `pollEnabled` | `false` | Starts the background poller. Requires `enabled` to also be `true`. |
| `pollIntervalSeconds` | `10` | How often the poller checks for a change. |
| `changeDetectionColumn` | `updated_at` | Column whose table-wide `MAX(...)` is compared across polls to detect a change — a timestamp column maintained by whoever writes the table, or any other monotonically-increasing column (a sequence/version column works identically). Never read into the contributed tenant config itself — purely a change signal. |

See [developer/tenant-config-loader-database.md](../developer/tenant-config-loader-database-v1-2026-10-22.md#live-reload) for exactly what "reload" means here (a full application-wide config reload, not scoped to the tenant that changed) before enabling this in a deployment with many tenants or a short poll interval.

## Next

- [Architecture: tenant-config-loader-database](../architecture/tenant-config-loader-database-v1-2026-10-22.md) — the `IConfigLoader` SPI, boot-sequence positioning, deep-merge semantics, fatal/non-fatal policy.
- [Developer: tenant-config-loader-database](../developer/tenant-config-loader-database-v1-2026-10-22.md) — worked table/script example, full column reference with a target-config-field mapping, live-reload caveats.
- [Multi-tenancy](multi-tenancy.md) — the `multitenancy.tenants` shape this loader contributes into.
- [Platform database service](platform-database-service.md) — the connection block this loader reuses.
