# Bootstrap reference (`payos.json` and `bootstrap.json`)

`payos.json` is the root of a [bundle](../overview/glossary.md). It is located by
`BootServer` via `--bundle-path` (a bundle directory or a direct path to `payos.json`) and
loaded by `ConfigLoader.loadServerConfig()`. In normal bundles, `payos.json` is intentionally
small: it mainly points to `configDir`, where the runtime configuration files live. The main
runtime configuration is typically `config/bootstrap.json` and contains blocks such as
`servers`, `applications`, `security`, `database-service`, and `multitenancy`.

This page documents the bundle entrypoint, the effective top-level runtime structure, and
runtime directory keys; each sub-block has its own page (see [configuration/README.md](README.md)).

## Bundle entrypoint (`payos.json`)

```json
{
  "configDir": "config"
}
```

`configDir` is resolved relative to the directory that contains `payos.json` when it is not
absolute. `ConfigLoader` then loads and merges the JSON files found in that directory.

## Runtime configuration (`config/bootstrap.json`)

A typical `bootstrap.json` contains the actual runtime blocks:

```json
{
  "connectors-dir": "connectors",
  "extensions-dir": "extensions",

  "servers":        [ /* see servers.md */ ],
  "applications":   [ /* see below */ ],
  "multitenancy":   { /* see multi-tenancy.md */ },
  "security":       { /* see security-oidc.md */ },
  "database-service": { /* see database-service.md */ },
  "queue-service":  { /* see queue-service.md */ },
  "secret-service": { /* see secret-service.md */ },
  "webhooks":       { /* see webhook-service.md */ },
  "http-webhook-service": { /* see webhook-service.md */ },
  "i18n":           { /* see i18n.md */ },
  "swaggerUI":      { /* see servers.md */ },
  "runtimeCompatibility": { /* see below */ }
}
```

## Runtime directory keys

From `IConfigSpec`. `runtimeBaseDir` is shown here because it exists in the effective
configuration map, but it is **computed by the loader from the directory that contains
`payos.json`**. Do not set it manually in `payos.json`; use `--bundle-path` to choose the
bundle root, and use `configDir` to choose where merged configuration files live.

| Key | Constant | Default | Purpose |
| --- | --- | --- | --- |
| `runtimeBaseDir` | `RUNTIME_BASE_DIRECTORY` | directory containing `payos.json` | Effective runtime base directory; computed and set on `PayOSConfig`. |
| `configDir` | `CONFIG_DIRECTORY` | `.` | Directory whose files are merged into the configuration, resolved relative to `runtimeBaseDir` when not absolute. |
| `config-hot-reload-enabled` | `CONFIG_HOT_RELOAD_ENABLED` | `true` | Enable configuration hot-reload (watches config directories for changes). Set to `false` to require manual restarts for config changes. |
| `connectors-dir` | `CONNECTORS_DIR` | — | Directory scanned for connector (SPI) JARs. |
| `extensions-dir` | `EXTENSIONS_DIR` | — | Directory scanned for extension JARs. |
| (internal) | `CAPABILITIES_DIR` | `.capabilities` | Capability state under `configDir`. |
| `tcp-handlers-dir` | `TCP_HANDLERS_DIR` | — | Directory scanned for TCP codec/handler plugins. |

The runtime entrypoint file name itself is `payos.json` (`RUNTIME_CONFIG_FILE`).

> Directory resolution for connectors/extensions/tcp-handlers also honors system properties and environment variables — see [extensions-connectors.md](extensions-connectors.md).

## `applications`

An array registering each application in the runtime, normally declared in
`config/bootstrap.json` or another JSON file under `configDir`. Model: `Application`; keys
from `IConfigSpec.Applications.Application`.

```json
{
  "applications": [
    {
      "id": "payments",
      "name": "Payments",
      "base.path": "../apps/payments",
      "version": "1.0.0",
      "minRuntimeVersion": "1.10.0",
      "maxRuntimeVersion": "2.0.0",
      "category": "application",
      "extends": ["audit-capability"],
      "authorized-tenants": ["acme", "globex"],
      "defaultUrl": "/payments/page/dashboard",
      "mapping-files": ["mappings/account.json"],
      "security": { /* per-app overrides, see security-oidc.md */ },
      "database-service": { /* per-app overrides, see database-service.md */ }
    }
  ]
}
```

| Key | Required | Default | Purpose |
| --- | --- | --- | --- |
| `id` | ✅ | — | Unique id; first URI path segment. |
| `name` | | — | Display name. |
| `base.path` | | `.apps/{id}` | Application directory, resolved relative to `configDir`. |
| `version` | | — | Semantic version. |
| `minRuntimeVersion` | | — | Lower bound (inclusive) of the compatible payos-runtime version. See [runtime compatibility checks](../developer/runtime-compatibility-checks-v1-2026-08-06.md). |
| `maxRuntimeVersion` | | — | Upper bound (inclusive) of the compatible payos-runtime version. Same reference as above. |
| `category` | | `application` | `application` or `capability`. |
| `extends` | | — | Parent app/capability id(s) for resource inheritance. |
| `authorized-tenants` / `authorized.tenants` | | — | Tenant allowlist. |
| `defaultUrl` | | — | Post-login redirect. |
| `mapping-files` | | — | Data-model mapping file paths. |
| `security` | | inherits global | Per-app security/OIDC overrides. |
| `database-service` | | inherits global | Per-app database overrides. |

The application model is explained in
[developer/application-model.md](../developer/application-model.md).

## `applicationCatalog` (optional)

Resolve applications from a catalog rather than local paths:

| Field | Purpose |
| --- | --- |
| source type | `local` or `git`. |
| `baseUrl` | Git base URL (for `git`). |
| `path` | Local repo path (for `local`). |

## `runtimeCompatibility` (optional)

Controls what `BootServer` does when an installed application/capability's `minRuntimeVersion`/`maxRuntimeVersion` (above) rules out the running payos-runtime version:

```json
{
  "runtimeCompatibility": {
    "warnOnly": true
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `warnOnly` | `false` | `false`: log every incompatibility at `ERROR` and abort boot. `true`: log at `WARN` and keep starting (dev/sandbox use). |

Full reference: [developer/runtime-compatibility-checks-v1-2026-08-06.md](../developer/runtime-compatibility-checks-v1-2026-08-06.md).

## Environment placeholders & encryption

Any string value may use `${ENV_VAR}` (resolved by `EnvVarResolver`) and may be stored encrypted (decrypted by `CryptoService` at load). See [operations/bundle-encryption.md](../operations/bundle-encryption.md).

## Next

- [servers.md](servers.md) — transport listeners.
- [reference/configuration-keys.md](../reference/configuration-keys.md) — the flat index.
