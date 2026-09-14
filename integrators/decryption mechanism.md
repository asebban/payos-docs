
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
   256-bit recommended for PCI-DSS Req 3.6.1, 128/192 accepted for backward compatibility).
   `CryptoService` itself never resolves a secret provider or reads bootstrap configuration — it
   only performs cryptographic operations against whatever key was set. A missing or malformed
   `editor-secret-service` configuration block is fatal: `EditorEncryptionKeyInitializer` throws,
   which fails the boot rather than leaving `CryptoService` keyless — see
   [configuration/editor-secret-service.md](../configuration/editor-secret-service.md).
3. Whenever a bundle file needs decrypting, `CryptoService.decryptIfEncrypted(data)` detects
   the format from a 4-byte magic header and decrypts in memory using that already-resolved key:
   - **`P8G2`** — `AES/GCM/NoPadding` (12-byte IV, 128-bit authentication tag). The PCI-DSS-compliant
     format, and, as of `payosv2-packer` v1.3.0-RELEASE / this doc's 2026-09-13 update, what
     `edc pack` actually writes by default.
   - **`P8OS`** — `AES/ECB/PKCS5Padding`, unauthenticated (no integrity/tamper detection).
     `edc pack` no longer produces this — `edc unpack` still decodes it read-only, so bundles
     packed by older tool versions remain usable.
4. **The resolved key is never held whole for longer than a single decrypt call.** Before
   2026-08-11, the key bytes were re-fetched from the secret provider on every decrypt call and
   held only as a short-lived local variable, zeroed by `SecretValue.close()` afterward. Between
   2026-08-11 and 2026-09-13, `setKey` instead resolved the key once at bootstrap and held it as a
   single `SecretKeySpec` field for the process lifetime — fewer round-trips to the secret
   provider, at the cost of a longer-lived, contiguous key residency in memory. **Since
   2026-09-13, `CryptoService` splits the key instead of holding it whole**: `setKey` XORs the
   incoming key against a `SecureRandom` mask, zeroes the caller's array immediately, and retains
   `mask`/`share` for the process lifetime, each inside its own object; a background task
   re-randomizes the mask periodically (every 5 minutes, currently hardcoded), recomputing
   `share` against the unchanged real key. Every decrypt call reconstructs the whole key into a
   fresh buffer just long enough to initialize the `Cipher`, then zeroes that buffer in a
   `finally` block — see
   [architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md#7-runtime-startup-and-decrypt-at-load-with-in-memory-key-protection)
   for the full mechanism and its honestly-stated limits (it defeats a naive memory-byte scan, not
   an attacker who can capture a full heap/object-graph snapshot at one instant).

There is no re-derivation from an external source (Vault, HSM) per operation, and no envelope
encryption — the split/reconstruct scheme above works entirely from the one key value resolved
once at bootstrap. If you need protection against an attacker who can actively instrument the
running JVM (a debugger, a native memory-read primitive) or capture a full structured heap dump,
that is **not** a property this mechanism provides — flag it as a requirement rather than
assuming it's already covered.

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
- [architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md](../architecture/tenant-bundle-encryption-key-lifecycle-v11-2026-09-13.md) — the full key lifecycle this page's decrypt step fits into (generation, custody, delivery, rotation).
