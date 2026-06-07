# Bundle encryption (`edc`)

For regulated, tamper-evident delivery, PayOS bundles can be **packed and encrypted** with
the `edc` tool (the `payosv2-packer` module). This page covers operational use; the CLI
reference is also summarized in [cli-tools/edc.md](../cli-tools/edc.md).

## What `edc` does

`edc` performs AES encryption/decryption of bundle contents, writing a packed artifact with a
magic header (`P8OS`). It supports two modes — **pack** (encrypt) and **unpack** (decrypt) —
and can source its encryption key from a flag, a generated key, or a secret provider.

- Main class: `ma.s2m.Main` (module `payosv2-packer`, version `1.2.0-RELEASE`).
- Installed as the command `edc` via `install-edc.sh` / `install-edc.ps1`.

## Install

```bash
# from the payosv2-packer module
./install-edc.sh        # or install-edc.ps1 on Windows
```

This installs the `edc` command (typically into `~/.payos/bin`).

## Generate a key

```bash
edc --generatekey 16
```

`--generatekey <n>` generates an `n`-character key (16 for AES-128). Store it securely — pre­
ferably in your secret provider rather than on disk.

## Pack (encrypt) a bundle

```bash
edc --encryption pack \
    --inputdir ./bundle \
    --key "0123456789abcdef"
```

| Flag | Purpose |
| --- | --- |
| `--encryption pack` | Encrypt mode. |
| `--inputdir` | Bundle directory to pack. |
| `--key <16-char>` | Encryption key (or if --key is absent, source it from a secret provider, below). |

## Unpack (decrypt)

```bash
edc --encryption unpack \
    --inputdir ./packed-bundle \
    --key "0123456789abcdef"
```

## Sourcing the key from a secret provider

Instead of `--key`, `edc` can read the key from a configured secret provider — useful to
avoid keys on the command line or in scripts:

| Flag | Default | Purpose |
| --- | --- | --- |
| `--secret-provider` | `filesystem` | Provider type (`filesystem` / `vault`). |
| `--secret-tenant` | `default` | Tenant under which the key is stored. |
| `--secret-name` | `encryptionKey` | Secret name holding the key. |
| `--root` | — | Filesystem provider root. |
| `--keyfile` | — | Filesystem provider master key file. |
| `--secret-config` | — | Path to a secret-provider config. |
| `--connectors-dir` | — | Where to load the provider connector from. |

This mirrors the runtime's [secret service](../configuration/secret-service.md), so the same
Vault or filesystem store can hold the bundle key.

## Operational guidance

- Treat the bundle key as a high-value secret: store it in Vault, restrict access, and rotate
  it on a schedule (see [secrets-management.md](secrets-management.md)).
- Keep an auditable record of which key version packed which delivered bundle.
- The `P8OS` magic header lets tooling verify a packed artifact before unpacking.
- This guide only shows an example for filesystem secret provider. To explore all options (including vault secret provider) see the following document [CLI tools guide](./cli-tools-guide.md).

## Next

- [cli-tools/edc.md](../cli-tools/edc.md)
- [secrets-management.md](secrets-management.md)
