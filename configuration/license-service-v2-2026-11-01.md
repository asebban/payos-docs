# License validation configuration

**Created:** 2026-08-19
**Last updated:** 2026-11-01
**Version:** v2

> Supersedes the v0 revision of this page (2026-08-19). The `license-file-path` JSON key it documented is **no longer read anywhere** — the constant (`IConfigSpec.LICENSE_FILE_PATH`) still exists but has no consumer left in the codebase. Licence location and validator selection are both environment-variable-driven now; see below. Design: [architecture/license-validation-architecture-v5-2026-11-20.md](../architecture/license-validation-architecture-v5-2026-11-20.md).
>
> **v2 (2026-11-01)**: renames `PAYOS_EDITOR_TENANT_ID` to `PAYOS_EDITOR_ID` (§3) — it identifies the editor's own namespace in the secret provider, not a PayOS multitenancy tenant.

Licence validation is the **first** thing `BootServer` does — for the real validator, before `payos.json` is even read (see the architecture doc, §2). If validation fails the process exits immediately (`System.exit(1)`).

## 1. Choosing the active validator

| Variable / property | Default | Effect |
| --- | --- | --- |
| `payos.license.devMode` (system property) | — | Takes priority over the environment variable below if set. |
| `PAYOS_LICENSE_DEV_MODE` (environment variable) | unset | Checked if the system property is unset. |

Resolution: unset, or any value `Boolean.parseBoolean` does not resolve to `false` (i.e. anything other than exactly `false`/`FALSE`/etc.), selects `DevLicenseValidator` (no-op). **Only an explicit `false` selects the real `LicenseValidator`.** There is no build-time toggle anymore — see [build-and-release/license-validator-mode-v1-2026-10-31.md](../build-and-release/license-validator-mode-v1-2026-10-31.md) for what replaced the old Maven-profile mechanism.

```bash
# production: enable real validation
export PAYOS_LICENSE_DEV_MODE=false
```

## 2. Locating the licence file (real validator only)

No configuration key points to the licence file anymore. Instead, a fixed convention (`ma.s2m.payos.license.LicenseFileLocator`, in `payos-foundation`) applies:

| Variable | Purpose |
| --- | --- |
| `PAYOS_LICENSE_FILE_PATH` | Overrides the convention scan — points directly at the licence file. Must exist; an invalid override is always a hard failure. |
| *(no variable — convention)* | The single `*.lic` file directly under the bundle's root directory (the directory containing `payos.json`, or `--bundle-path` if it was a directory). Zero or multiple matches without an override is a hard failure. |

This same variable and convention are shared, unchanged, with `payosv2-packer` (`edc`) — see [operations/bundle-encryption.md](../operations/bundle-encryption.md).

## 3. Resolving the editor's secret provider before config is loaded (real validator only)

The real validator needs a connection to the editor's secret provider (for signature verification and RSA key-unwrap) before `payos.json` can be read — so this must come from environment variables only, never from a JSON block. `BootServer.buildEditorSecretServiceConfigFromEnvironment()` builds the equivalent of the `editor-secret-service` JSON block purely from:

| Variable | Equivalent JSON key |
| --- | --- |
| `PAYOS_EDITOR_SECRET_TYPE` | `editor-secret-service.configuration.type` |
| `PAYOS_EDITOR_SECRET_ENABLED` | `editor-secret-service.configuration.enabled` |
| `PAYOS_EDITOR_SECRET_CFG_<NAME>` | `editor-secret-service.configuration.<name>` (uppercased, `-` → `_`) |
| `PAYOS_EDITOR_ID` | Namespace/id of the editor's signing key pair in the secret provider — not a PayOS tenant (default `editor`) |
| `PAYOS_EDITOR_KEY_NAME` | Key pair name (default `editor-signing-key`) |

These are the exact same variables `EditorEncryptionKeyInitializer.tryInitializeFromEnvironment()` already reads for the vault-fallback mechanism (see [editor-secret-service.md](editor-secret-service.md)) — the two paths share the same names so they can never drift apart. If none of these variables are set at all, boot fails immediately with an explicit message rather than attempting a credential-less connection.

```bash
export PAYOS_EDITOR_SECRET_TYPE=vault
export PAYOS_EDITOR_SECRET_CFG_ADDRESS=https://editor-vault.s2m.internal:8200
export PAYOS_EDITOR_SECRET_CFG_TOKEN=s.XXXXXXXXXXXX
export PAYOS_EDITOR_SECRET_CFG_KV_MOUNT=secret
export PAYOS_EDITOR_ID=editor
export PAYOS_EDITOR_KEY_NAME=editor-signing-key
```

## 4. What happens after a successful validation

The source-decryption key unwrapped from the licence's `key_envelope.encrypted_key` is set as `CryptoService`'s active key **before** `ConfigLoader.loadServerConfig()` runs — so `payos.json` itself may be encrypted with that exact key. See the architecture doc §8 for the in-memory protection details (`LicenseContext`'s own `SplitKeyGuard`, independent from `CryptoService`'s).

## 5. Behaviour by validator

| Active validator | No `.lic` found | Licence invalid (signature/dates/machine/etc.) |
| --- | --- | --- |
| `DevLicenseValidator` (default) | Ignored — never even looked for | Ignored |
| `LicenseValidator` | Logs error + `System.exit(1)` | Logs error + `System.exit(1)` |

## 6. Notes

- Hot-reload does **not** re-run licence validation — the result holds for the lifetime of the process, same as before this feature existed.
- If `CryptoService` already has a key from a validated licence, the vault-based `editor-secret-service`/`encryptionKey` fallback (§3's variables, and the equivalent JSON block) is skipped entirely — see [editor-secret-service.md](editor-secret-service.md).
- File permissions: restrict read access to the OS user running the PayOS process (e.g. `chmod 640`). Never expose the licence file path or content in logs or API responses.

## Next

- [architecture/license-validation-architecture-v5-2026-11-20.md](../architecture/license-validation-architecture-v5-2026-11-20.md) — full design.
- [developer/license-validation-v2-2026-11-01.md](../developer/license-validation-v2-2026-11-01.md) — writing a custom validator.
- [editor-secret-service.md](editor-secret-service.md) — the vault fallback mechanism and its environment variables.
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — `payosv2-packer`'s matching licence-derived key support.
