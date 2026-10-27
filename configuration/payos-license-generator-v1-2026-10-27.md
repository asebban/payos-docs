Created: 2026-10-27
Last updated: 2026-10-27
Version: 1

# payos-license-generator configuration

Configuration reference for `payos-license-generator` — a standalone service, not part of `payos-runtime`'s `payos.json`/`bootstrap.json` configuration tree. For the full build/run walkthrough see [operations/payos-license-generator-setup-guide-v1-2026-10-27.md](../operations/payos-license-generator-setup-guide-v1-2026-10-27.md).

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
| `editor-tenant-id` | value | `editor` | Tenant ID under which the editor's RSA key pair is stored in the secret provider. Must match the `--tenant` used when the key pair was created (see the setup guide's `spm keypair-create` step). |
| `editor-key-name` | value | `editor-signing-key` | Name of the editor's stored RSA key pair. Must match the `--key-name` used at key-pair creation time. Used for both `encryptWithPublicKey`/`generateAndWrapSymmetricKey` (the key envelope) and `signWithKeyPair` (the license signature). |
| `license-output-dir` | path | `licenses` | Directory the completed, signed license file is written to, named `<license_id>.json`. Written atomically (temp file, then rename) — see the architecture doc's `LicenseFileWriter` notes. |
| `http-port` | value | `8080` | Port the Undertow HTTP listener binds, on all interfaces (`0.0.0.0`). |

## Example

```json
{
  "secret-provider-dir": "connectors",
  "secret-provider-type": "filesystem",
  "secret-provider-config": {
    "root": "secrets/store",
    "keyfile": "secrets/master.key"
  },
  "editor-tenant-id": "editor",
  "editor-key-name": "editor-signing-key",
  "license-output-dir": "licenses",
  "http-port": "8099"
}
```

## Vault instead of filesystem

Set `secret-provider-type` to `vault`, put a packaged `secret-service-vault` JAR in `secret-provider-dir` instead of (or alongside) the filesystem provider's, and populate `secret-provider-config` with Vault's own connection settings (address, token, mount points) — see [secret-service.md](secret-service.md) for the exact key names the Vault factory expects. The editor's RSA key pair must still be created once against Vault Transit before first use; `spm` only targets the filesystem provider (see [cli-tools/spm.md](../cli-tools/spm.md)), so use the Vault CLI/API directly for that step, matching [operations/secrets-management.md](../operations/secrets-management.md)'s Vault guidance.

## Not yet configurable (MVP)

Authentication is unconditionally absent — there is no config key to enable it yet; the HTTP layer has an auth seam but nothing wired to it (Epic 5 of the source PRD). Payload validation, the caller-fault/infra-fault error taxonomy, request persistence, and metrics exposition are likewise not yet implemented (Epics 2–4), so there are no configuration keys for them yet either.
