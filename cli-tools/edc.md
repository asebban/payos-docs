# `edc` — Bundle encryptor/packer

`edc` packs and encrypts (or unpacks and decrypts) PayOS bundles for regulated, tamper-evident
delivery. It is the `payosv2-packer` module (main class `ma.s2m.Main`, version
`1.2.0-RELEASE`). Operational guidance is in
[operations/bundle-encryption.md](../operations/bundle-encryption.md).

## Install

```bash
# from the payosv2-packer module
./install-edc.sh        # or install-edc.ps1 on Windows
```

Installs the `edc` command (typically into `~/.payos/bin`).

## Modes

`edc` operates in two encryption modes (AES); packed artifacts carry the magic header `P8OS`.

| Flag | Purpose |
| --- | --- |
| `--encryption pack` | Encrypt/pack a bundle. |
| `--encryption unpack` | Decrypt/unpack a bundle. |
| `--inputdir <dir>` | The input bundle directory. |
| `--generatekey <n>` | Generate an `n`-character key (e.g. 16 for AES-128). |
| `--key <16-char>` | Provide the encryption key inline. |

## Sourcing the key from a secret provider

Instead of `--key`, `edc` can pull the key from a secret provider (mirrors the runtime
[secret service](../configuration/secret-service.md)):

| Flag | Default | Purpose |
| --- | --- | --- |
| `--secret-provider` | `filesystem` | Provider type. |
| `--secret-tenant` | `default` | Tenant holding the key. |
| `--secret-name` | `encryptionKey` | Secret name. |
| `--root` | — | Filesystem provider root. |
| `--keyfile` | — | Filesystem provider master key file. |
| `--secret-config` | — | Secret provider config path. |
| `--connectors-dir` | — | Where to load the provider connector from. |

## Examples

```bash
# generate a 16-char key
edc --generatekey 16

# pack a bundle with an inline key
edc --encryption pack --inputdir ./bundle --key "0123456789abcdef"

# pack using a key stored in the filesystem secret provider
edc --encryption pack --inputdir ./bundle \
    --secret-provider filesystem --root ./secrets \
    --secret-tenant default --secret-name encryptionKey

# unpack
edc --encryption unpack --inputdir ./packed-bundle --key "0123456789abcdef"
```

## Next

- [operations/bundle-encryption.md](../operations/bundle-encryption.md)
- [operations/secrets-management.md](../operations/secrets-management.md)
