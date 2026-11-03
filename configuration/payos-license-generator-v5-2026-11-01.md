Created: 2026-10-27
Last updated: 2026-11-01
Version: 5 — supersedes v4: renames the `editor-tenant-id` config key to `editor-id` — it identifies the editor's own namespace in the secret provider, not a PayOS multitenancy tenant.

# payos-license-generator configuration

Configuration reference for `payos-license-generator` — a standalone service, not part of `payos-runtime`'s `payos.json`/`bootstrap.json` configuration tree. For the full build/run walkthrough see [operations/payos-license-generator-setup-guide-v7-2026-11-01.md](../operations/payos-license-generator-setup-guide-v7-2026-11-01.md).

## Resolution order

Every setting below resolves with the same priority, checked in this order, first match wins:

1. **System property** — the key name as-is, e.g. `-Dsecret-provider-type=vault`.
2. **Environment variable** — `PAYOS_LICENSE_<KEY>`, uppercased, with `-` and `.` replaced by `_`. For example `secret-provider-type` becomes `PAYOS_LICENSE_SECRET_PROVIDER_TYPE`.
3. **The config file** — `license-generator-config.json`, read once at startup. Entirely optional: a missing file simply means this tier contributes nothing, every setting falls through to its default.
4. **The built-in default** listed in the table below.

`secret-provider-config` is the one exception: it is a nested JSON object, not a single value, so it is sourced **only from the config file** — there is no system-property or environment-variable override for the object as a whole. Individual provider implementations may have their own environment-variable fallbacks independent of this service (for example the filesystem provider's `PAYOS_SECRET_MASTER_KEY`, documented in [secret-service.md](secret-service.md)).

## Where the config file itself lives

`license-generator-config.json` is located at `<generator-path>/license-generator-config.json`, where `<generator-path>` is:

- the directory passed via the `--generator-path <dir>` CLI argument, if supplied — **every** relative file-path-valued setting below (not just the config file's own location) resolves relative to this directory;
- otherwise, the process's working directory at launch.

## Keys

| Key | Type | Default | Purpose |
| --- | --- | --- | --- |
| `secret-provider-dir` | path | `connectors` | Directory scanned for secret-provider JARs. A `URLClassLoader` is built from every `.jar` found there, and `ServiceLoader` discovers `ISecretProviderFactory` implementations against it — the service has no compile-time dependency on any provider. |
| `secret-provider-type` | value | `filesystem` | Which discovered factory to instantiate, matched by its `ISecretProviderFactory.type()`. Startup fails with a clear error naming the missing type if no matching factory is found in `secret-provider-dir`. |
| `secret-provider-config` | object | `{}` | Passed as-is to the selected factory's `create(Map<String, Object>)`. For the filesystem provider: `root` (key storage directory) and `keyfile` (master key file path) — see [secret-service.md](secret-service.md) for the full filesystem/Vault key set. |
| `editor-id` | value | `editor` | Namespace/id under which the editor's RSA key pair is stored in the secret provider — not a PayOS multitenancy tenant (there is exactly one editor), just the SPI's generic storage-namespace parameter reused for this purpose. Must match the `--tenant` used when the key pair was created (see the setup guide's `spm keypair-create` step). |
| `editor-key-name` | value | `editor-signing-key` | Name of the editor's stored RSA key pair. Must match the `--key-name` used at key-pair creation time. Used for both `encryptWithPublicKey`/`generateAndWrapSymmetricKey` (the key envelope) and `signWithKeyPair` (the license signature). |
| `license-output-dir` | path | `licenses` | Directory the completed, signed license file is written to, named `<license_id>.json`. Written atomically (temp file, then rename) — see the architecture doc's `LicenseFileWriter` notes. |
| `http-port` | value | `8080` | Port the Undertow HTTP listener binds, on all interfaces (`0.0.0.0`). |
| `persistence-enabled` | value | `false` | When `true`, every generation attempt (success or failure) is additionally written as a row to a database over a plain JDBC connection — see Persistence below. When `false` (the default), attempts are still logged (see Logging below) but no database is touched at all. |
| `persistence-jdbc-url` | value | `jdbc:h2:file:<generator-path>/license-generator-db` | JDBC connection URL, used exactly as given — **not** resolved against `--generator-path` the way the path-typed settings above are, since it's a connection string, not necessarily a filesystem path. Only the *default* (used when this key is absent) is built from `--generator-path`; an explicit value is passed straight to the driver. Set it to any other vendor's URL (PostgreSQL, MySQL, etc.) to persist there instead. Only relevant when `persistence-enabled` is `true`. |
| `persistence-jdbc-driver` | value | `org.h2.Driver` | Fully-qualified JDBC driver class to load before connecting. The service bundles H2's driver; any other vendor's driver JAR must be placed on the classpath yourself (the kernel/service does not bundle it) — see Persistence below. |
| `persistence-jdbc-username` | value | `sa` | Database user for the JDBC connection. |
| `persistence-jdbc-password` | value | *(empty)* | Database password for the JDBC connection. |

## Example

```json
{
  "secret-provider-dir": "connectors",
  "secret-provider-type": "filesystem",
  "secret-provider-config": {
    "root": "secrets/store",
    "keyfile": "secrets/master.key"
  },
  "editor-id": "editor",
  "editor-key-name": "editor-signing-key",
  "license-output-dir": "licenses",
  "http-port": "8099"
}
```

## Logging

Every log line the service emits is a single JSON object, printed to stdout via Logback's `net.logstash.logback.encoder.LogstashEncoder` — there is no configuration key to change this format for MVP. Standard fields present on every line include `@timestamp`, `level`, `message`, `logger_name`, and `thread_name`; a `correlation_id` field is added automatically (via SLF4J's MDC) to every log line produced while handling a specific HTTP request — the same value returned to the caller in the `X-Correlation-Id` response header, so a failed request can be traced directly to its log lines.

One additional log line is emitted per generation attempt, success or failure alike, independent of whether persistence (below) is enabled — its message is always `"License generation attempt completed"`, carrying four event-specific fields: `client_id`, `license_id` (`null` unless the attempt succeeded), `machine_ids` (an array), and `outcome` (one of `SUCCESS`, `VALIDATION_FAILURE`, `INFRA_FAILURE`).

## Persistence

Setting `persistence-enabled` to `true` additionally writes that same per-attempt information as a row over a plain JDBC connection — any database, not only H2, configured entirely by `persistence-jdbc-url`/`persistence-jdbc-driver`/`persistence-jdbc-username`/`persistence-jdbc-password`. The table, `license_generation_attempts`, has columns `client_id`, `license_id` (`null` on failure), `generated_at` (timestamp), `outcome`, and `error_category` (`null` on success, otherwise the same value as `outcome`). Persistence is off by default — omitting `persistence-enabled` entirely, or setting it to `false`, means no database connection is ever opened and no row is ever written.

The table is created automatically on first use for a vendor that supports `CREATE TABLE IF NOT EXISTS` — true of H2 (the bundled default), PostgreSQL, MySQL/MariaDB, and SQLite. For a vendor that doesn't support that syntax (for example Oracle or SQL Server), create the table yourself with this schema before enabling persistence; the service detects the table already exists and simply uses it instead of failing.

```json
{
  "persistence-enabled": "true",
  "persistence-jdbc-url": "jdbc:h2:file:./db/license-generator"
}
```

The leading `./` is required, not cosmetic: H2 2.x rejects a bare relative path (`jdbc:h2:file:db/license-generator`, no `./`) outright with "a file path that is implicitly relative to the current working directory is not allowed" — and, unlike `license-output-dir` or the other path-typed settings, this value is **not** resolved against `--generator-path` first (see the Keys table above), so `./` here is relative to the process's actual working directory at launch, not the bundle directory. Prefer an absolute path if the two differ in your deployment.

To use another database instead of the bundled H2 default, point `persistence-jdbc-url`/`persistence-jdbc-driver` at it and put that vendor's JDBC driver JAR on the classpath yourself (the service only bundles H2's):

```json
{
  "persistence-enabled": "true",
  "persistence-jdbc-url": "jdbc:postgresql://localhost:5432/payos_licenses",
  "persistence-jdbc-driver": "org.postgresql.Driver",
  "persistence-jdbc-username": "payos"
}
```

For the password, prefer the system-property or environment-variable tier (`PAYOS_LICENSE_PERSISTENCE_JDBC_PASSWORD`) over the config file — this service's config file has no `${ENV_VAR}` placeholder expansion of its own (unlike `payos-runtime`'s), so a value written there is stored as plain text.

The embedded-H2 case can still be inspected directly with H2's bundled shell tool, pointed at the same URL (the service must be stopped first — H2 locks the file exclusively against a second connection from another process):

```bash
java -cp h2-2.3.232.jar org.h2.tools.Shell \
  -url "jdbc:h2:file:/absolute/path/to/db/license-generator" -user sa -password "" \
  -sql "SELECT * FROM license_generation_attempts;"
```

For any other vendor, inspect the table with that vendor's own client against the same `persistence-jdbc-url`/credentials.

## Metrics

`GET /metrics` is always on — there is no configuration key to disable it, and no dependency on `persistence-enabled`. It serves a Prometheus text-exposition scrape (Micrometer's `PrometheusMeterRegistry`, bundled) of two counters, updated on every generation attempt:

| Metric | Labels | Meaning |
| --- | --- | --- |
| `license_generator_licenses_generated_total` | *(none)* | Total licenses successfully generated. |
| `license_generator_generation_errors_total` | `category="validation"` \| `category="infra"` | Total failed attempts, broken down by whether the caller's payload was invalid or an infrastructure fault (secret provider unreachable, etc.) occurred. |

Both are plain counters — they only ever increase for the life of the process; a restart resets them to zero, same as any in-memory Micrometer registry with no remote push.

## Vault instead of filesystem

Set `secret-provider-type` to `vault`, put a packaged `secret-service-vault` JAR in `secret-provider-dir` instead of (or alongside) the filesystem provider's, and populate `secret-provider-config` with Vault's own connection settings (address, token, mount points) — see [secret-service.md](secret-service.md) for the exact key names the Vault factory expects. The editor's RSA key pair must still be created once against Vault Transit before first use; `spm` only targets the filesystem provider (see [cli-tools/spm.md](../cli-tools/spm.md)), so use the Vault CLI/API directly for that step, matching [operations/secrets-management.md](../operations/secrets-management.md)'s Vault guidance.

## Not yet configurable (MVP)

Authentication is unconditionally absent — there is no config key to enable it yet; the HTTP layer has an auth seam but nothing wired to it (Epic 5 of the source PRD), and this applies to `/metrics` too, which is just as unauthenticated as `/v1/generate_license`.
