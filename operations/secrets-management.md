# Secrets management

Operational guidance for running PayOS secrets in production: choosing and provisioning a
provider, setting up Vault, rotating secrets, and using the [`spm`](../cli-tools/spm.md) CLI.
Configuration keys are in [configuration/secret-service.md](../configuration/secret-service.md);
script usage is in [developer/secrets-usage.md](../developer/secrets-usage.md).

## Provider choice in production

| | `filesystem` | `vault` |
| --- | --- | --- |
| Recommended for | Edge/on-prem with no Vault | Centralized, production secret management |
| Master key | Key file or `PAYOS_SECRET_MASTER_KEY` | Vault auth (AppRole/token) |
| Tokenization | ✅ | ✅ |

Whichever you pick, ensure the provider connector JAR is on the
[connectors path](../configuration/extensions-connectors.md).

## Filesystem provider operations

Storage layout (AES-256-GCM):

```
<root>/<tenant>/<name>.enc          # encrypted secret
<root>/<tenant>/<name>.meta.json    # metadata
<root>/<tenant>/tokens/<uuid>.enc   # tokenized values
```

- Provide the master key via `keyfile` or the `PAYOS_SECRET_MASTER_KEY` environment variable
  (env var is preferable so the key is not on disk).
- Back up `<root>` and the master key **separately**; losing the key makes secrets
  unrecoverable.
- Restrict filesystem permissions on `<root>` and the key file.

Manage filesystem secrets with [`spm`](../cli-tools/spm.md) (the Secret Package Manager).

## Vault provider setup

1. **Enable KV v2** at a mount (default `secret`):

   ```bash
   vault secrets enable -path=secret kv-v2
   ```

2. **Create an AppRole** for PayOS and capture `role-id` / `secret-id`:

   ```bash
   vault auth enable approle
   vault write auth/approle/role/payos token_policies="payos-policy"
   vault read  auth/approle/role/payos/role-id
   vault write -f auth/approle/role/payos/role-id/secret-id  # generates secret-id
   ```

3. **Configure PayOS** ([secret-service.md](../configuration/secret-service.md)):

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
         "timeout": 10
       }
     }
   }
   ```

   AppRole credentials take precedence over a static `token`. Avoid `tls-skip-verify` outside
   of development.

> Each secret is stored as its own KV v2 entry (`<kvMount>/data/<tenantId>/<name>`), with the
> secret's own `name` as the JSON field key inside `data` (not a fixed field name) and the value
> Base64-encoded (falling back to raw UTF-8 on read if a value isn't valid Base64, for secrets
> written by other tooling). Tokens (`ITokenProvider`) are stored the same way, namespaced under
> `<tenantId>/tokens/<uuid>`.

## Rotation

- **Application secrets:** rotate by writing the new value via `spm` (filesystem) or the Vault
  CLI/API — not from scripts, since `$Secrets` doesn't expose writes (see
  [developer/secrets-usage.md](../developer/secrets-usage.md)). Both providers track versions
  (`describe` reports the current version); `filesystem` additionally keeps a readable version
  history via `IVersionedSecretProvider` (Java API only, not yet exposed by `spm`).
- **Vault AppRole secret-id:** re-issue the `secret-id` periodically and update
  `${VAULT_SECRET_ID}`; PayOS picks it up on [reload](hot-reload.md) or restart.
- **Filesystem master key / bundle key:** plan key rotation with re-encryption; coordinate
  with [bundle encryption](bundle-encryption.md).

## Auditing

Every secret operation writes an audit entry through the `payos.secret.audit` logger,
including tenant context. Ship these logs to your SIEM for PCI DSS evidence — see
[observability.md](observability.md).

## Least privilege & tenancy

Secrets are tenant-scoped automatically. Grant each deployment only the Vault policy it needs;
do not share AppRoles across unrelated tenants/environments.

## Next

- [configuration/secret-service.md](../configuration/secret-service.md)
- [cli-tools/spm.md](../cli-tools/spm.md)
- [developer/secrets-usage.md](../developer/secrets-usage.md)
