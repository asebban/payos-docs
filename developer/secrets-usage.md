# Secrets usage (`$Secrets`)

When a [secret service](../configuration/secret-service.md) is configured, scripts receive the `$Secrets` binding — a `SecretsBinding` that wraps the tenant's configured `ISecretProvider` (`filesystem` or `vault`). It exposes **reading**, **writing/deleting**, **listing**, **tokenizing**, **encrypt/decrypt/sign/verify with a named key**, **named-key generation** (symmetric and asymmetric), and a stateless **hash**; it still does not expose metadata reads or capability queries (see [What scripts cannot do](#what-scripts-cannot-do)). This page covers usage from JavaScript; provider configuration is in [configuration/secret-service.md](../configuration/secret-service.md) and operational setup (Vault, rotation, the `spm` CLI) is in [operations/secrets-management.md](../operations/secrets-management.md).

## Tenant scoping is automatic

`SecretsBinding` captures the current request's tenant once and passes it on every call, so scripts never pass a tenant explicitly. (The underlying `ISecretProvider` SPI takes `tenantId` explicitly on every method — that only matters if you're implementing a custom provider.) The base provider validates the tenant and writes an audit entry (`payos.secret.audit`) for every operation.

## Reading a secret

```javascript
function execute(request, controlData) {
    var apiKey = $Secrets.get("psp-api-key");   // returns a String, already UTF-8 decoded
    return callPsp(apiKey, controlData.amount);
}
```

> The provider's in-memory secret representation (`SecretValue`) is zeroed on the Java side before the plain `String` reaches script code — there's nothing to close from JavaScript.
> Still, don't log the value or stash it in a long-lived/global variable.

## Writing and deleting secrets

```javascript
$Secrets.set("psp-api-key", "sk_live_xxxx");   // creates, or replaces the current value
$Secrets.delete("psp-api-key");
```

`set` always replaces the entire stored entry — there's no partial update. Because there's no `metadata` parameter, the stored `SecretMetadata.type` defaults to `"string"`; scripts that need a specific type (or a TTL) must go through provider admin tooling instead. `delete` throws `SecretNotFoundException` if the name doesn't exist.

## Listing secrets

```javascript
var names = $Secrets.list();   // List<String> of secret names for the current tenant
```

## Tokenization

```javascript
var token    = $Secrets.tokenize(cardNumber);   // opaque, non-reversible token (UUID v4)
var original = $Secrets.detokenize(token);
```

Both shipped providers implement `ITokenProvider` (`filesystem` natively; `vault` by storing
tokens as ordinary KV v2 entries under a `tokens/` sub-path, kept out of `list()`/`listSecrets`
results). Against a custom provider that doesn't implement `ITokenProvider`, both calls throw
`UnsupportedOperationException`.

## Crypto operations (encrypt/decrypt/sign/verify)

```javascript
var ciphertext = $Secrets.encrypt("invoice-pdf", documentText);   // Base64 string
var original   = $Secrets.decrypt("invoice-pdf", ciphertext);

var signature  = $Secrets.sign("invoice-pdf", documentText);      // Base64 string
var isValid    = $Secrets.verify("invoice-pdf", documentText, signature);   // boolean
```

`encrypt`/`decrypt`/`sign`/`verify` operate on a **named symmetric key** that must already exist —
generate one with `$Secrets.generateKey` (below), or provision it out-of-band via the `spm
key-create` CLI (filesystem) or the Vault Transit API directly (vault) — see
[`secret-service-filesystem`](../../secret-service-filesystem/README.md) and
[`secret-service-vault`](../../secret-service-vault/README.md), section 5.

All four methods take/return plain strings: `encrypt`/`sign` return Base64 text (so the result is
always a safe plain string regardless of the provider's native ciphertext format — raw AES-GCM
envelope bytes for `filesystem`, Vault's `vault:v1:...` wire format for `vault`); `decrypt`/`verify`
accept that same Base64 text back. Both shipped providers implement `ICryptoSecretProvider`.
Against a custom provider that doesn't, all four calls throw `UnsupportedOperationException`.

> `sign`/`verify` use HMAC-SHA256 under both shipped providers, not an asymmetric signature
> scheme — the named key is symmetric (AES-256), the same key used for `encrypt`/`decrypt`.

## Generating keys

```javascript
$Secrets.generateKey("invoice-key");   // symmetric key, for encrypt/decrypt/sign/verify above

var publicKeyPem = $Secrets.generateKeyPair("signing-key", "RSA_2048");
// PEM text (X.509 SubjectPublicKeyInfo) — the private key stays with the provider, never returned
```

`generateKey` creates a new named symmetric key (`ICryptoSecretProvider`); it throws `UnsupportedOperationException` against a provider that doesn't implement that interface. `generateKeyPair` creates a new named asymmetric key pair (`IAsymmetricCryptoSecretProvider`, implemented by both `filesystem` — Bouncy Castle — and `vault` — the Transit engine) and returns only the public key — the private key stays with the provider. `algorithm` must be one of `RSA_2048`, `RSA_3072`, `RSA_4096`, `EC_P256` (it throws `IllegalArgumentException` for anything else, and `UnsupportedOperationException` against a custom provider without asymmetric support). `$Secrets` has no method to sign/verify with, or fetch the public key of, a generated key pair — the rest of `IAsymmetricCryptoSecretProvider` (`signWithKeyPair`, `verifyWithKeyPair`, `getPublicKey`, `verifyWithPublicKey`) is SPI-level only for now (see [What scripts cannot do](#what-scripts-cannot-do)). Both shipped providers refuse to regenerate an existing name — re-calling `generateKey`/`generateKeyPair` with a `keyName` that already has a key throws `SecretProviderException`; there's no in-place rotation via `$Secrets`, only creation of a new name.

## Hashing

```javascript
var digest = $Secrets.hash("stripe-api-key");   // lowercase hex SHA-256, e.g. "2cf24d...b9824"
```

`hash` is a plain SHA-256 digest, not scoped to a tenant and not backed by any provider-held key — the same input always produces the same output, on either shipped provider, and the result is reproducible with any standard SHA-256 implementation outside PayOS. Unlike every other `$Secrets` method it is always available, even against a custom provider that implements only the base `ISecretProvider` contract, since it's a `default` method on the interface itself rather than a capability a provider can opt out of. It throws `IllegalArgumentException` if `value` is `null`.

Because it's unkeyed, don't use it where the result must not be guessable from a candidate input (a session token, a password) — use `sign` (above) for that instead, which is keyed by a provider-held key. `hash` fits deterministic use cases instead: deduplication, cache keys, or comparing two values without putting either one in a log.

## What scripts cannot do

Reading metadata (`describeSecret`) and querying supported operations (`capabilities()`) are part of the Java `ISecretProvider` SPI used by provider implementations and platform tooling — `$Secrets` does not expose them to scripts. Nor does it expose the rest of `IAsymmetricCryptoSecretProvider` beyond `generateKeyPair`: `getPublicKey`, `signWithKeyPair`, `verifyWithKeyPair`, and `verifyWithPublicKey` are SPI-level only for now — a script can mint a key pair but has no way to sign, verify, or re-fetch its public key through `$Secrets` today. `keyExists`/`keyPairExists` (check whether a name is already taken before generating) are likewise SPI-only; a script finds out the same thing by calling `generateKey`/`generateKeyPair` and catching `SecretProviderException`. `IVersionedSecretProvider` (version history/rollback) and `ICertificateSecretProvider` (per-tenant CA) are also SPI-level only, reachable through provider admin tooling (e.g. the [`spm`](../cli-tools/spm.md) CLI for the filesystem provider) but not through `$Secrets`.

For reference, the capabilities the two shipped providers implement at the SPI level:

| Capability | `filesystem` | `vault` |
| --- | --- | --- |
| `GET` / `SET` / `DELETE` / `LIST` / `DESCRIBE` | ✅ | ✅ |
| `VERSION` | ✅ | ✅ |
| `TOKENIZE` | ✅ | ✅ |
| `CRYPTO` | ✅ | ✅ |
| `ASYMMETRIC_CRYPTO` | ✅ | ✅ |

For `vault`, `VERSION` means `describeSecret` reports Vault KV v2's real `current_version` —
Vault keeps prior versions internally, but the provider doesn't yet expose them through
`IVersionedSecretProvider`. `filesystem` does implement `IVersionedSecretProvider`
(`getSecretVersion`, `listVersions`, `restoreVersion`, `destroyVersion`): every `setSecret`
archives the overwritten envelope, so older versions can be read back or restored (as a new
version) via the SPI. None of this is reachable from `$Secrets` — these are SPI-level
operations for provider implementers and admin tooling.

For `CRYPTO`, `filesystem` backs named keys with AES-256 keys generated via `SecureRandom` and wrapped under the provider's master key; `vault` backs them with the Vault Transit secrets engine (`aes256-gcm96` keys, tenant-scoped by composing the Transit key name as `<tenantId>_<keyName>`). `$Secrets.generateKey` reaches this; `keyExists` remains SPI-level only.

For `ASYMMETRIC_CRYPTO`, `filesystem` backs named key pairs with Bouncy Castle, `vault` with the Transit engine's asymmetric key types. `$Secrets.generateKeyPair` reaches this; `keyPairExists`, `getPublicKey`, `signWithKeyPair`, `verifyWithKeyPair`, and `verifyWithPublicKey` remain SPI-level only (see [What scripts cannot do](#what-scripts-cannot-do)).

Both shipped providers also implement `ICertificateSecretProvider` (per-tenant CA), but it isn't reachable from `$Secrets` at all — see [architecture/extensibility.md](../architecture/extensibility.md) for the SPI pattern. `IWatchableSecretProvider` (live reload on secret changes) isn't implemented by either shipped provider today.

## Error handling

`$Secrets.get`, `set`, `delete`, `list`, `tokenize`, `detokenize`, `encrypt`, `decrypt`, `sign`, `verify`, `generateKey`, `generateKeyPair`, and `hash` may throw:

| Exception | Meaning |
| --- | --- |
| `SecretNotFoundException` | The named secret does not exist for the tenant (`get`, `delete`). |
| `SecretAccessDeniedException` | The tenant is not allowed to access the secret. |
| `TokenNotFoundException` | The token doesn't exist or was revoked (`detokenize`). |
| `SecretProviderException` | Generic provider failure — including a named key that was never generated (`encrypt`/`decrypt`/`sign`/`verify`), or a `generateKey`/`generateKeyPair` call naming a key that already exists. |
| `UnsupportedOperationException` | `tokenize`/`detokenize` called against a provider without `ITokenProvider` support; `encrypt`/`decrypt`/`sign`/`verify`/`generateKey` called against a provider without `ICryptoSecretProvider` support; or `generateKeyPair` called against a provider without `IAsymmetricCryptoSecretProvider` support. |
| `IllegalArgumentException` | `generateKeyPair` called with an `algorithm` that isn't a `KeyAlgorithm` constant, or `hash` called with a `null` value. |

```javascript
try {
    var apiKey = $Secrets.get("psp-api-key");
    // ...
} catch (e) {
    throw new BusinessException("PSP key unavailable");
}
```

## If `$Secrets` is missing

`$Secrets` is injected only when `secret-service.configuration.enabled` is `true` and a
provider `configuration.type` (`filesystem` or `vault`) is configured with a matching provider
JAR on the connectors path. See [configuration/secret-service.md](../configuration/secret-service.md).

## See how to configure and use vault secret provider

[Vault Secret Provider](./vault-secret-id-secure-injection.md)

## Next

- [Configuration: secret service](../configuration/secret-service.md)
- [Operations: secrets management](../operations/secrets-management.md)
- [CLI: spm](../cli-tools/spm.md)
