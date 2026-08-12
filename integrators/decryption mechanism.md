
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
2. **Since 2026-08-11, resolution happens exactly once, at bootstrap, not per decrypt call.**
   `EditorEncryptionKeyInitializer.initialize(...)` is called from
   `ConfigLoader.loadServerConfig()` immediately after `PayOSConfig.settings` is populated (and
   before any per-application config — which may itself be encrypted — is loaded). It builds the
   editor secret provider from `editor-secret-service` configuration, calls
   `secretProvider.getSecret(tenantId, "encryptionKey")` once, and hands the resulting key bytes
   to `CryptoService.setKey(byte[])`, which validates the size (16/24/32 bytes — AES-128/192/256;
   256-bit recommended for PCI-DSS Req 3.6.1, 128/192 accepted for backward compatibility) and
   wraps it in a `SecretKeySpec` held for the lifetime of the process. `CryptoService` itself
   never resolves a secret provider or reads bootstrap configuration — it only performs
   cryptographic operations against whatever key was set. A missing or malformed
   `editor-secret-service` configuration block is fatal: `EditorEncryptionKeyInitializer` throws,
   which fails the boot rather than leaving `CryptoService` keyless — see
   [configuration/editor-secret-service.md](../configuration/editor-secret-service.md).
3. Whenever a bundle file needs decrypting, `CryptoService.decryptIfEncrypted(data)` detects
   the format from a 4-byte magic header and decrypts in memory using that already-resolved key:
   - **`P8G2`** — `AES/GCM/NoPadding` (12-byte IV, 128-bit authentication tag). The PCI-DSS-compliant
     format `CryptoService` is ready to decode — but as of this writing, `edc`/`payosv2-packer`'s
     `pack` command does not actually produce it yet; see
     [architecture/tenant-bundle-encryption-key-lifecycle-v4-2026-08-12.md](../architecture/tenant-bundle-encryption-key-lifecycle-v4-2026-08-12.md#gaps-to-close-before-this-is-production-ready).
   - **`P8OS`** — `AES/ECB/PKCS5Padding`. What `edc` actually produces today. Unauthenticated
     (no integrity/tamper detection) — treat as the current, not the target, format.
4. **The resolved key now lives for the process lifetime, not just per-operation.** Before
   2026-08-11, the key bytes were re-fetched from the secret provider on every decrypt call and
   held only as a short-lived local variable; `SecretValue.close()` (via try-with-resources)
   zeroed that fetch's byte array once the `Cipher` operation was done. The bootstrap-time
   resolution above trades that "fetch fresh, hold briefly, zero after each use" property for
   "fetch once, hold as a `CryptoService` field for as long as the process runs" — fewer
   round-trips to the secret provider (and resilience if it becomes briefly unavailable
   mid-run), at the cost of a longer-lived key residency in process memory. There is still no
   split-key/XOR/non-contiguous-memory scheme in either model.

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

- `payos/src/main/java/ma/s2m/payos/security/CryptoService.java` — `decryptIfEncrypted(...)`, `setKey(byte[])`.
- `payos/src/main/java/ma/s2m/payos/security/EditorEncryptionKeyInitializer.java` — resolves the editor secret provider and the key, once, at bootstrap.
- `payos/src/main/java/ma/s2m/payos/config/ConfigLoader.java` — calls `EditorEncryptionKeyInitializer.initialize(...)`.
- `payos-secret-api/src/main/java/ma/s2m/payos/secret/model/SecretValue.java` — the zero-on-close behavior.
- [integrators/assembling and encrypting a bundle.md](assembling%20and%20encrypting%20a%20bundle.md) — the encryption side (`edc`).
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — operational guidance.
- [architecture/tenant-bundle-encryption-key-lifecycle-v4-2026-08-12.md](../architecture/tenant-bundle-encryption-key-lifecycle-v4-2026-08-12.md) — the full key lifecycle this page's decrypt step fits into (generation, custody, delivery, rotation).
