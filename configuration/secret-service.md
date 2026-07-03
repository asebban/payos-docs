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
| `namespace` | — | Vault namespace (Enterprise). |
| `tls-skip-verify` | `false` | Skip TLS verification (non-production only). |
| `timeout` | `10` | HTTP timeout in seconds. |

**Auth precedence:** if `role-id`/`secret-id` are present, **AppRole** is used; otherwise the
static `token`. Capabilities: GET/SET/DELETE/LIST/DESCRIBE/VERSION and **TOKENIZE** (tokens are
stored as ordinary KV v2 entries under a `tokens/` sub-path of the tenant's namespace, and are
excluded from ordinary secret listings).

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
