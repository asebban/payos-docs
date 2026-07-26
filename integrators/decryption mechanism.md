
# Decryption mechanism

> This page previously described an RSA-envelope + Vault-decrypt + split-key-with-XOR memory
> protection scheme. That design was never implemented — the actual code (verified against
> `payos/src/main/java/ma/s2m/payos/security/CryptoService.java`) is materially simpler. This
> page has been rewritten to match the shipped mechanism.

## How it actually works

1. The AES symmetric key (`encryptionKey`) is stored as a plain secret in the configured
   [secret provider](../operations/secrets-management.md) (`filesystem` or `vault`) — there is
   **no RSA key pair, no envelope encryption, and no "ask Vault to decrypt with a private key"
   step**. The secret provider is asked for the raw key bytes directly.
2. At runtime, `CryptoService.loadKey()` calls
   `secretProvider.getSecret(tenantId, "encryptionKey")` and reads the key bytes straight out
   of the returned `SecretValue`. Accepted key sizes are 16, 24, or 32 bytes (AES-128/192/256);
   256-bit is recommended for PCI-DSS Req 3.6.1, but 128/192 are still accepted for backward
   compatibility.
3. Whenever a bundle file needs decrypting, `CryptoService.decryptIfEncrypted(data)` detects
   the format from a 4-byte magic header and decrypts in memory:
   - **`P8G2`** — `AES/GCM/NoPadding` (12-byte IV, 128-bit authentication tag). The PCI-DSS-compliant
     format `CryptoService` is ready to decode — but as of this writing, `edc`/`payosv2-packer`'s
     `pack` command does not actually produce it yet; see
     [tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md](tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md#gaps-to-close-before-this-is-production-ready).
   - **`P8OS`** — `AES/ECB/PKCS5Padding`. What `edc` actually produces today. Unauthenticated
     (no integrity/tamper detection) — treat as the current, not the target, format.
4. The decrypted `SecretKeySpec`/key bytes are held only as long as the `SecretValue` /
   `Cipher` operation needs them. `SecretValue.close()` (called via try-with-resources) zeroes
   its internal byte array (`Arrays.fill(bytes, (byte) 0)`) once the caller is done with it —
   there is no split-key/XOR/non-contiguous-memory scheme; the protection is "hold the key for
   as short a time as possible and zero it afterward," not memory-layout obfuscation.

There is no re-derivation step, no two-random-number XOR reconstruction, and no split storage
across non-contiguous memory regions. If you need genuinely hardened in-memory key protection
(e.g. to defend against memory-dump attacks), that is **not** a property this mechanism
currently provides — flag it as a requirement rather than assuming it's already covered.

## Where the key actually comes from

The key is provisioned exactly like any other secret in the platform — via `spm`/`edc`'s
`--secret-provider` flags, or directly through the filesystem/Vault secret provider — see
[operations/secrets-management.md](../operations/secrets-management.md) and
[cli-tools/edc.md](../cli-tools/edc.md). There is no separate "vault instance delivered by the
editor containing an RSA key pair" — the secret provider **is** the vault, and it stores the
AES key directly.

## References

- `payos/src/main/java/ma/s2m/payos/security/CryptoService.java` — `decryptIfEncrypted(...)`, `loadKey()`.
- `payos-secret-api/src/main/java/ma/s2m/payos/secret/model/SecretValue.java` — the zero-on-close behavior.
- [integrators/assembling and encrypting a bundle.md](assembling%20and%20encrypting%20a%20bundle.md) — the encryption side (`edc`).
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — operational guidance.
- [tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md](tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md) — the full key lifecycle this page's decrypt step fits into (generation, custody, delivery, rotation).
