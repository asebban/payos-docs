# `spm` — Secret Package Manager

`spm` manages secrets for the **filesystem** secret provider
(`secret-service-filesystem`). It operates directly on the provider's encrypted storage, so
operators can provision and rotate secrets without going through a running runtime. For the
provider configuration see
[configuration/secret-service.md](../configuration/secret-service.md); for operational
practice see [operations/secrets-management.md](../operations/secrets-management.md).

## Install the tool

```bash
# from the secret-service-filesystem module
./scripts/install.sh        # or scripts/install.ps1 on Windows
```

This installs `spm` (typically into `~/.payos/bin`). On Windows there are `spm.cmd` /
`spm.ps1` launchers.

## What it manages

`spm` reads and writes the same AES-256-GCM storage the runtime uses:

```
<root>/<tenant>/<name>.enc          # encrypted secret
<root>/<tenant>/<name>.meta.json    # metadata
<root>/<tenant>/tokens/<uuid>.enc   # tokenized values
```

It uses the same master key as the runtime — supplied via the provider `keyfile` or the
`PAYOS_SECRET_MASTER_KEY` environment variable.

## Typical operations

`spm` exposes a subset of the filesystem provider's capabilities as subcommands:

| Subcommand | Purpose |
| --- | --- |
| `keygen` | Generate a 32-byte AES-256 master key file. |
| `set` | Create/update a secret. |
| `get` | Read a secret. |
| `list` | List secret names for a tenant. |
| `delete` | Remove a secret. |
| `describe` | Show metadata (type, current version, timestamps). |
| `key-create` | Generate a named AES-256 symmetric key, stored internally — key material is never displayed. |
| `keypair-create` | Generate a named RSA/EC key pair (`--algorithm RSA_2048`\|`RSA_3072`\|`RSA_4096`\|`EC_P256`), stored internally — only the public key (PEM) is printed; the private key is never displayed or returned. Needed, for example, to bootstrap the editor's signing key for [payos-license-generator](../operations/payos-license-generator-setup-guide-v1-2026-10-27.md). |
| `encrypt` | Encrypt content with a named internal key (from `key-create`), print base64 ciphertext. |
| `decrypt` | Decrypt base64 ciphertext with a named internal key, write raw plaintext. |

The filesystem provider also supports version history (`IVersionedSecretProvider`) and
tokenization (`ITokenProvider`) at the Java API level, but `spm` doesn't yet expose CLI
subcommands for them (no `restore`, `destroy-version`, `tokenize`, or `detokenize`) — use the
Java API directly for those until CLI support is added.

Point `spm` at the provider's `root` and master key (matching your
[`secret-service`](../configuration/secret-service.md) config) and specify the tenant whose
secrets you are managing.

## Examples

```bash
# generate a named RSA key pair, printing only the public key
spm keypair-create --root ./secrets --tenant editor --key-name editor-signing-key --algorithm RSA_2048

# set a secret for tenant "acme"
spm set --root ./secrets --tenant acme --name psp-api-key --value "s3cr3t"

# list and describe
spm list     --root ./secrets --tenant acme
spm describe --root ./secrets --tenant acme --name psp-api-key

# delete
spm delete --root ./secrets --tenant acme --name psp-api-key
```

> Flag names follow the installed `spm` help (`spm --help`). The operations above map to the
> provider capabilities; use `--help` to confirm exact flags in your build.

## Safety

- Treat the master key as a high-value secret; back it up separately from `<root>`.
- Restrict filesystem permissions on `<root>` and the key file.
- Every operation is auditable through the provider's audit logger when run via the runtime;
  keep an operational record of out-of-band `spm` changes.

## Vault deployments

`spm` targets the **filesystem** provider. For Vault, use the Vault CLI/API and the AppRole
setup in [operations/secrets-management.md](../operations/secrets-management.md).

## Next

- [operations/secrets-management.md](../operations/secrets-management.md)
- [configuration/secret-service.md](../configuration/secret-service.md)
- [developer/secrets-usage.md](../developer/secrets-usage.md)
