# Editor secret service configuration

The `editor-secret-service` block configures the secret provider that backs `ma.s2m.payos.editor.IEditorProvider` — specifically, `CryptoService.loadKey()`'s resolution of the bundle-wide `encryptionKey` used to decrypt `P8G2`/`P8OS` config payloads produced by `edc` (the `payosv2-packer` module). It is deliberately **separate from [`secret-service`](secret-service.md)**: `secret-service` backs the `$Secrets` binding and holds ordinary per-tenant application secrets, while `editor-secret-service` holds the one bundle-wide key that only the editor (the core PayOS team producing the base application) ever mints — see [integrators/tenant-bundle-encryption-key-lifecycle](../integrators/tenant-bundle-encryption-key-lifecycle-v2-2026-07-27.md) for the full lifecycle and the reasoning behind keeping this custody path distinct from ordinary tenant secrets.

## Common keys

All keys below live under `editor-secret-service.configuration` — the exact same shape as `secret-service.configuration`, just under a different top-level block so the two can point at entirely different backends/credentials.

| Key | Purpose |
| --- | --- |
| `enabled` | Enable the editor secret service. If `false` (or the block/`configuration` node is absent), any code path that needs the editor secret provider (currently `CryptoService.loadKey()`) fails with a `ResourceException`. |
| `type` | Provider type: `filesystem` or `vault`, or any provider SPI available in `connectors-dir`. Defaults to `vault` if omitted — unlike `secret-service`, which defaults to `filesystem`. |

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

## Why this is a separate block, not a `secret-service` alias

`encryptionKey` protects the editor's own proprietary bundle files — it is not an ordinary tenant secret, is not resolved per-request-tenant (`CryptoService.resolveSecretTenantId()` always uses the fixed `"default"` slot), and its access-control boundary is fundamentally different: only the editor ever writes it, and every downstream party (integrator, client) is granted narrowly-scoped *read-only, on-the-fly decrypt* access, never through the same credential a tenant's own `$Secrets` usage would go through. Giving it its own top-level configuration block means an operator can — and in the recommended model, should — point `secret-service` and `editor-secret-service` at two entirely different Vault instances/policies/tokens, so that revoking or rotating one never touches the other.

## Next

- [secret-service.md](secret-service.md) — the general-purpose `$Secrets` binding this block is deliberately kept separate from.
- [integrators/tenant-bundle-encryption-key-lifecycle](../integrators/tenant-bundle-encryption-key-lifecycle-v2-2026-07-27.md) — the full key lifecycle (generation, custody, delivery, rotation) this configuration block fits into.
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — the `edc` CLI that produces the encrypted payloads this key decrypts.
