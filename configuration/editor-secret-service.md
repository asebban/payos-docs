# Editor secret service configuration

The `editor-secret-service` block configures the secret provider that backs `ma.s2m.payos.editor.IEditorProvider` — specifically, `EditorEncryptionKeyInitializer`'s resolution of the bundle-wide `encryptionKey` used to decrypt `P8G2`/`P8OS` config payloads produced by `edc` (the `payosv2-packer` module). It is deliberately **separate from [`secret-service`](secret-service.md)**: `secret-service` backs the `$Secrets` binding and holds ordinary per-tenant application secrets, while `editor-secret-service` holds the one bundle-wide key that only the editor (the core PayOS team producing the base application) ever mints — see [architecture/tenant-bundle-encryption-key-lifecycle](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md) for the full lifecycle and the reasoning behind keeping this custody path distinct from ordinary tenant secrets.

This block is **required by default** in every `bootstrap.json`, not just bundles that ship encrypted config files: `EditorEncryptionKeyInitializer` runs unconditionally during `ConfigLoader.loadServerConfig()`, and a missing or malformed `editor-secret-service.configuration` block fails boot outright with a `ResourceException` (surfaced as `ConfigNotFoundException`), rather than deferring the failure to whenever an encrypted file is first loaded. Bundles that never load encrypted config files can opt out explicitly with `"enabled": false` — see the key table below.

## Common keys

All keys below live under `editor-secret-service.configuration` — the exact same shape as `secret-service.configuration`, just under a different top-level block so the two can point at entirely different backends/credentials.

| Key | Purpose |
| --- | --- |
| `enabled` | Defaults to `true` if omitted — the opposite default from `secret-service.configuration`, since the editor's own bundle-wide key is required by default rather than opt-in. Set to `false` to explicitly skip provider resolution and let boot continue with `CryptoService` left keyless (only safe if this bundle never loads encrypted config files); any other value (including omitted) requires successful resolution or boot fails with a `ResourceException`. |
| `type` | Provider type: `filesystem` or `vault`, or any provider SPI available in `service-adapters-dir`. Defaults to `vault` if omitted — unlike `secret-service`, which defaults to `filesystem`. |

The factory is discovered by `type` via the same `ISecretProviderFactory`/`ServiceLoader` mechanism as `secret-service` (`ma.s2m.payos.secret.SecretProviders`), so the provider JAR must be on the [connectors path](extensions-connectors.md) — typically the same `secret-service-vault`/`secret-service-filesystem` JAR already used for `secret-service`, just instantiated a second time with independent configuration.

## `vault` provider (recommended)

Same configuration keys as [`secret-service`'s `vault` provider](secret-service.md#vault-provider) (`address`, `token` / `role-id`+`secret-id`, `approle-mount`, `kv-mount`, `namespace`, `tls-skip-verify`, `timeout`) — see that page for the full key table. Only the two keys below are specific to this block:

```json
{
  "editor-secret-service": {
    "configuration": {
      "enabled": true,
      "type": "vault",
      "address": "https://editor-vault.s2m.internal:8200",
      "token": "s.XXXXXXXXXXXX",
      "kv-mount": "secret"
    }
  }
}
```

In the recommended key-blind delivery model, this block points at a Vault instance/policy scoped to exactly `secret/data/default/encryptionKey` (see the lifecycle doc linked above) — typically a centrally-operated Vault distinct from whatever `secret-service` a given deployment's own tenants might separately configure for their own business secrets.

If one central Vault instance serves multiple clients rather than a dedicated Vault per client, `kv-mount` (or `namespace` on Vault Enterprise) is what must differ per client — `address` alone is not enough, and neither is a distinct token/policy on its own, since the tenant id this block resolves against is always the fixed `"default"` slot, never a client identifier. See [integrators/tenant-bundle-encryption-key-lifecycle](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md#multi-client-isolation-on-a-single-central-vault-instance) for the full pattern (one KV mount per client, scoped policy per mount).

## `filesystem` provider

Same configuration keys as [`secret-service`'s `filesystem` provider](secret-service.md#filesystem-provider) (`root`, `keyfile`, or the `PAYOS_SECRET_MASTER_KEY` environment variable):

```json
{
  "editor-secret-service": {
    "configuration": {
      "enabled": true,
      "type": "filesystem",
      "root": "/opt/payos/editor-secrets",
      "keyfile": "/opt/payos/editor-secrets/.keyfile"
    }
  }
}
```

Simple, single-instance deployments may use this instead of Vault — but note that the filesystem provider's own master key then becomes the thing that must be delivered out-of-band to whichever party needs decrypt access, exactly as described for `secret-service`'s filesystem provider.

## Configuring via environment variables instead (required if `payos.json` is fully encrypted)

Everything above assumes `editor-secret-service` lives as plaintext JSON somewhere `ConfigLoader` can read before any decryption happens. If `payos.json` itself is encrypted in its entirety — no exclusion carved out for this block — that assumption breaks: nothing can read a JSON block sitting inside a file that itself needs the key from that very block to be decrypted first. For exactly this case, `EditorEncryptionKeyInitializer.tryInitializeFromEnvironment()` resolves the same configuration from environment variables instead, checked before `payos.json` is read off disk at all:

| Variable | Equivalent JSON key |
| --- | --- |
| `PAYOS_EDITOR_SECRET_TYPE` | `editor-secret-service.configuration.type` |
| `PAYOS_EDITOR_SECRET_ENABLED` | `editor-secret-service.configuration.enabled` |
| `PAYOS_EDITOR_SECRET_CFG_<NAME>` | `editor-secret-service.configuration.<name>` — `<NAME>` is the config key, uppercased with `-` replaced by `_` (e.g. `PAYOS_EDITOR_SECRET_CFG_KV_MOUNT` → `kv-mount`) |

Equivalent to the `vault` example above:

```
PAYOS_EDITOR_SECRET_TYPE=vault
PAYOS_EDITOR_SECRET_CFG_ADDRESS=https://editor-vault.s2m.internal:8200
PAYOS_EDITOR_SECRET_CFG_TOKEN=s.XXXXXXXXXXXX
PAYOS_EDITOR_SECRET_CFG_KV_MOUNT=secret
```

These variables take full precedence over the JSON block: if any `PAYOS_EDITOR_SECRET_*` variable is set, resolution happens from the environment alone and `payos.json`'s `editor-secret-service` block (if any) is never even looked for — mixing sources (some fields from JSON, some from environment) is not supported. If none of these variables are set, behavior is exactly as documented above (the JSON block is read from `payos.json`, which must then be plaintext or otherwise already-decryptable).

This only covers `editor-secret-service` — the classical `secret-service` block (`$Secrets`, ordinary tenant secrets) has no equivalent environment-variable path yet; see [architecture/tenant-bundle-encryption-key-lifecycle](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md#3-editor-scopes-what-actually-gets-encrypted) for that open gap.

## Why this is a separate block, not a `secret-service` alias

`encryptionKey` protects the editor's own proprietary bundle files — it is not an ordinary tenant secret, is not resolved per-request-tenant (`EditorEncryptionKeyInitializer` always uses the fixed `"default"` slot, once, at bootstrap), and its access-control boundary is fundamentally different: only the editor ever writes it, and every downstream party (integrator, client) is granted narrowly-scoped *read-only, on-the-fly decrypt* access, never through the same credential a tenant's own `$Secrets` usage would go through. Giving it its own top-level configuration block means an operator can — and in the recommended model, should — point `secret-service` and `editor-secret-service` at two entirely different Vault instances/policies/tokens, so that revoking or rotating one never touches the other.

## Next

- [secret-service.md](secret-service.md) — the general-purpose `$Secrets` binding this block is deliberately kept separate from.
- [architecture/tenant-bundle-encryption-key-lifecycle](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md) — the full key lifecycle (generation, custody, delivery, rotation) this configuration block fits into.
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — the `edc` CLI that produces the encrypted payloads this key decrypts.
