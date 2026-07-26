# Secret service configuration

The `secret-service` block configures the secret provider that backs the `$Secrets` binding. Two providers ship with PayOS: `filesystem` and `vault`. Developer usage is in
[developer/secrets-usage.md](../developer/secrets-usage.md); operational setup and rotation are in [operations/secrets-management.md](../operations/secrets-management.md).

## Common keys

All keys below live under `secret-service.configuration`.

| Key | Purpose |
| --- | --- |
| `enabled` | Enable the secret service (injects `$Secrets`). |
| `type` | Provider type: `filesystem` or `vault`. Selects the `ISecretProviderFactory`. |

The factory is discovered by `type` and the provider is constructed at bootstrap. The provider JAR must be on the [connectors path](extensions-connectors.md).

## `filesystem` provider

Local, AES-256-GCM encrypted storage. Provider:
`FileSystemSecretProvider` (factory type `filesystem`).

```json
{
  "secret-service": {
    "configuration": {
      "enabled": true,
      "type": "filesystem",
      "root": "secrets",
      "keyfile": "config/secret.key"
    }
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `root` | `secrets` | Root directory for encrypted secret storage. |
| `keyfile` | — | Path to the master key file. |

The master key may instead be supplied via the `PAYOS_SECRET_MASTER_KEY` environment
variable. Storage layout: `<root>/<tenant>/<name>.enc` + `.meta.json`; tokens at
`<root>/<tenant>/tokens/<uuid>.enc`. Capabilities: GET/SET/DELETE/LIST/DESCRIBE/VERSION and
**TOKENIZE**. The CLI for managing filesystem secrets is [`spm`](../cli-tools/spm.md).

## `vault` provider

HashiCorp Vault KV v2 over HTTP. Provider: `VaultSecretProvider` (factory type `vault`).

```json
{
  "secret-service": {
    "configuration": {
      "enabled": true,
      "type": "vault",
      "address": "https://vault.internal:8200",
      "kv-mount": "secret",
      "transit-mount": "transit",
      "pki-mount": "pki",
      "pki-role": "delivery-cert",
      "namespace": "payos",
      "approle-mount": "approle",
      "role-id": "${VAULT_ROLE_ID}",
      "secret-id": "${VAULT_SECRET_ID}",
      "tls-skip-verify": false,
      "timeout": 10
    }
  }
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `address` | — | Vault server address. |
| `token` | — | Static token auth (used if AppRole not provided). |
| `role-id` | — | AppRole role id. |
| `secret-id` | — | AppRole secret id. |
| `approle-mount` | `approle` | AppRole auth mount path. |
| `kv-mount` | `secret` | KV v2 mount path. |
| `transit-mount` | `transit` | Transit mount path — symmetric crypto and asymmetric key pairs. Only needed for `CRYPTO`/`ASYMMETRIC_CRYPTO` operations. |
| `pki-mount` | `pki` | PKI mount path — certificate issuance. Only needed for `CERTIFICATE_AUTHORITY` operations. |
| `pki-role` | — | Base Vault PKI role name; the role actually used per call is `<tenantId>_<pki-role>` (one Vault CA is shared by all tenants — see [secret-service-vault/README.md §7](../../secret-service-vault/README.md#7-certificate-issuance-pki)). Required only for `CERTIFICATE_AUTHORITY` operations. |
| `namespace` | — | Vault namespace (Enterprise). |
| `tls-skip-verify` | `false` | Skip TLS verification (non-production only). |
| `timeout` | `10` | HTTP timeout in seconds. |

**Auth precedence:** if `role-id`/`secret-id` are present, **AppRole** is used; otherwise the
static `token`. Capabilities: GET/SET/DELETE/LIST/DESCRIBE/VERSION, **TOKENIZE** (tokens are
stored as ordinary KV v2 entries under a `tokens/` sub-path of the tenant's namespace, and are
excluded from ordinary secret listings), **CRYPTO** (named symmetric keys via Transit —
encrypt/decrypt/sign/verify), **ASYMMETRIC_CRYPTO** (named key pairs via Transit —
generate/sign/verify with RSA or EC keys, plus verifying against an arbitrary externally-supplied
public key), and **CERTIFICATE_AUTHORITY** (X.509 certificate issuance/revocation via the PKI
engine). See [secret-service-vault/README.md](../../secret-service-vault/README.md) for the full
detail on each — key naming, wire formats, and the shared-CA-per-tenant-role design.

> Each secret is one KV v2 entry (`<kv-mount>/data/<tenantId>/<name>`), stored under the
> secret's own `name` as the JSON field key (not a fixed field name) and Base64-encoded (with a
> UTF-8 fallback on read). Operational setup is in
> [operations/secrets-management.md](../operations/secrets-management.md).

## Choosing a provider

| | `filesystem` | `vault` |
| --- | --- | --- |
| Best for | Local/dev, simple on-prem | Production, centralized secret management |
| Tokenization | ✅ | ✅ |
| External infra | None | Vault server |

## Next

- [developer/secrets-usage.md](../developer/secrets-usage.md)
- [operations/secrets-management.md](../operations/secrets-management.md)
- [cli-tools/spm.md](../cli-tools/spm.md)
