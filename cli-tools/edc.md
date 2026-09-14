# `edc` — Bundle encryptor/packer

`edc` packs and encrypts (or unpacks and decrypts) PayOS bundles for regulated, tamper-evident
delivery. It is the `payosv2-packer` module (main class `ma.s2m.Main`, version
`1.3.0-RELEASE`). Operational guidance is in
[operations/bundle-encryption.md](../operations/bundle-encryption.md).

## Install

```bash
# from the payosv2-packer module
./install-edc.sh        # or install-edc.ps1 on Windows
```

Installs the `edc` command (typically into `~/.payos/bin`).

## Modes

`edc` operates in two encryption modes (AES). `pack` writes the `P8G2` magic header
(`AES/GCM/NoPadding`, a fresh random IV per file, 128-bit authentication tag); `unpack` also
still reads the legacy `P8OS` header (`AES/ECB/PKCS5Padding`, unauthenticated) for bundles
packed by older tool versions, but `pack` never writes it anymore.

| Flag | Purpose |
| --- | --- |
| `--encryption pack` | Encrypt/pack a bundle. |
| `--encryption unpack` | Decrypt/unpack a bundle. |
| `--inputdir <dir>` | The input bundle directory. |
| `--outputdir <dir>` | Write the result here instead of mutating `--inputdir` — see "Output location and crash safety" below. |
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
| `--service-adapters-dir` | — | Where to load the provider connector from. |

### Vault provider flags

When `--secret-provider vault` is used, these additional flags configure the Vault client
(each also has a `PAYOS_VAULT_*` environment variable fallback):

| Flag | Default | Purpose |
| --- | --- | --- |
| `--vault-address` | — | Vault server URL (e.g. `https://vault.internal:8200`). |
| `--vault-namespace` | — | Vault namespace (optional). |
| `--vault-kv-mount` | `secret` | KV v2 mount path. |
| `--vault-auth-method` | `token` | Auth method: `token` or `approle`. |
| `--vault-token` | — | Vault token (for token auth). |
| `--vault-approle-mount` | `approle` | AppRole mount path. |
| `--vault-role-id` | — | AppRole role-id (for approle auth). |
| `--vault-secret-id` | — | AppRole secret-id (for approle auth). |
| `--vault-timeout` | `10` | HTTP timeout, in seconds. |

## Unpack access control

`unpack` performs disk-level decryption, which the platform's process model reserves to the
editor's own internal verification — never an integrator or a client (see
[architecture/tenant-bundle-encryption-key-lifecycle](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md)).
When the key is resolved from a secret provider (not `--key`), `unpack` additionally requires a
second, separate secret to be readable under the same provider/tenant — only its presence is
checked, never its content:

| Flag | Env var | Default | Purpose |
| --- | --- | --- | --- |
| `--unpack-auth-secret-name` | `PAYOS_UNPACK_AUTH_SECRET_NAME` | `unpackAuthorization` | Secret whose mere readability authorizes `unpack`. |

Grant read access to this secret only to editor-internal credentials — an integrator's or
client's Vault policy, scoped only to `encryptionKey` for the runtime's on-the-fly decrypt, must
never be granted it. **This check does not apply when `--key <literal>` is used** — passing the
raw key directly on the command line bypasses every credential-based control here, the same
pre-existing limitation as always ("Key length requirement" below notes the other consequence of
that flag).

For how to actually create `unpackAuthorization` and the two Vault policies (editor-admin vs.
client/integrator) that make it unreachable outside the editor — plus the filesystem-provider
equivalent — see
[architecture/tenant-bundle-encryption-key-lifecycle §2](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md#provisioning-unpack-access-control-unpackauthorization).

## Output location and crash safety

By default (no `--outputdir`), `pack`/`unpack` mutate `--inputdir` in place. The residual risk —
a failed or interrupted run leaving a mix of processed/unprocessed files — is bounded two ways:

- **Re-running is safe.** `pack` skips any file already carrying a recognized magic header;
  `unpack` skips any file carrying none. An interrupted run can simply be re-run to completion.
- **Every write is atomic.** Each file is written to a temp file in the same directory, then
  renamed into place — a process killed mid-write can never leave a truncated, corrupt file.

Pass `--outputdir <dir>` for a stronger guarantee: `--inputdir` is never touched at all. The
result is written under `--outputdir`, mirroring `--inputdir`'s tree — files that don't need
transforming (already-encrypted files on `pack`, non-encrypted files on `unpack`) are copied
through unchanged rather than dropped, so the output is always a complete bundle. A failed run
can be discarded (delete the incomplete output directory) and retried against an input that was
never at risk.

## Key length requirement

Pack and unpack always require an **exactly 16-character** key — `PayOSPacker` rejects any
other length (`key.length() != 16`) for both `pack` and `unpack`. This is independent of
`--generatekey <n>`, which can generate a key of any length `n`; if you intend to use the
generated key with `pack`/`unpack`, generate it with `n = 16`.

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

# pack to a separate output directory — ./bundle is never modified
edc --encryption pack --inputdir ./bundle --outputdir ./bundle-packed --key "0123456789abcdef"
```

## Next

- [operations/bundle-encryption.md](../operations/bundle-encryption.md)
- [operations/secrets-management.md](../operations/secrets-management.md)
