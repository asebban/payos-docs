# Secrets usage (`$Secrets`)

When a [secret service](../configuration/secret-service.md) is configured, scripts receive the `$Secrets` binding — a `SecretsBinding` that wraps the tenant's configured `ISecretProvider` (`filesystem` or `vault`). It exposes a narrow, script-safe surface for **reading**, **listing**, **tokenizing**, and **encrypt/decrypt/sign/verify with a pre-provisioned named key**; it does not expose writes, deletes, metadata, capability queries, or named-key creation (see [What scripts cannot do](#what-scripts-cannot-do)). This page covers usage from JavaScript; provider configuration is in [configuration/secret-service.md](../configuration/secret-service.md) and operational setup (Vault, rotation, the `spm` CLI) is in [operations/secrets-management.md](../operations/secrets-management.md).

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

`encrypt`/`decrypt`/`sign`/`verify` operate on a **named key** that must already exist — scripts
cannot create named keys (see [What scripts cannot do](#what-scripts-cannot-do)). The key is
provisioned once, out-of-band, via the `spm key-create` CLI (filesystem) or the Vault Transit API
directly (vault) — see [`secret-service-filesystem`](../../secret-service-filesystem/README.md)
and [`secret-service-vault`](../../secret-service-vault/README.md), section 5.

All four methods take/return plain strings: `encrypt`/`sign` return Base64 text (so the result is
always a safe plain string regardless of the provider's native ciphertext format — raw AES-GCM
envelope bytes for `filesystem`, Vault's `vault:v1:...` wire format for `vault`); `decrypt`/`verify`
accept that same Base64 text back. Both shipped providers implement `ICryptoSecretProvider`.
Against a custom provider that doesn't, all four calls throw `UnsupportedOperationException`.

> `sign`/`verify` use HMAC-SHA256 under both shipped providers, not an asymmetric signature
> scheme — the named key is symmetric (AES-256), the same key used for `encrypt`/`decrypt`.

## What scripts cannot do

Writing (`setSecret`), deleting (`deleteSecret`), reading metadata (`describeSecret`), querying
supported operations (`capabilities()`), and creating named crypto keys (`generateKey`) are part
of the Java `ISecretProvider`/`ICryptoSecretProvider` SPI used by provider implementations and
platform tooling (e.g. the [`spm`](../cli-tools/spm.md) CLI for the filesystem provider) —
`$Secrets` does not expose them to scripts. A script can *use* a named key it was given, but it
can never mint one — mirroring how scripts can read/tokenize existing secrets but never write or
delete them.

For reference, the capabilities the two shipped providers implement at the SPI level:

| Capability | `filesystem` | `vault` |
| --- | --- | --- |
| `GET` / `SET` / `DELETE` / `LIST` / `DESCRIBE` | ✅ | ✅ |
| `VERSION` | ✅ | ✅ |
| `TOKENIZE` | ✅ | ✅ |
| `CRYPTO` | ✅ | ✅ |

For `vault`, `VERSION` means `describeSecret` reports Vault KV v2's real `current_version` —
Vault keeps prior versions internally, but the provider doesn't yet expose them through
`IVersionedSecretProvider`. `filesystem` does implement `IVersionedSecretProvider`
(`getSecretVersion`, `listVersions`, `restoreVersion`, `destroyVersion`): every `setSecret`
archives the overwritten envelope, so older versions can be read back or restored (as a new
version) via the SPI. None of this is reachable from `$Secrets` — these are SPI-level
operations for provider implementers and admin tooling.

For `CRYPTO`, `filesystem` backs named keys with AES-256 keys generated via `SecureRandom` and
wrapped under the provider's master key; `vault` backs them with the Vault Transit secrets engine
(`aes256-gcm96` keys, tenant-scoped by composing the Transit key name as `<tenantId>_<keyName>`).
`generateKey`/`keyExists` are SPI-level only, not reachable from `$Secrets`.

Custom providers may additionally implement `IWatchableSecretProvider` or
`ICertificateSecretProvider` — see [architecture/extensibility.md](../architecture/extensibility.md)
for the SPI pattern. Neither shipped provider implements these today.

## Error handling

`$Secrets.get`, `list`, `tokenize`, `detokenize`, `encrypt`, `decrypt`, `sign`, and `verify` may throw:

| Exception | Meaning |
| --- | --- |
| `SecretNotFoundException` | The named secret does not exist for the tenant (`get`). |
| `SecretAccessDeniedException` | The tenant is not allowed to access the secret. |
| `TokenNotFoundException` | The token doesn't exist or was revoked (`detokenize`). |
| `SecretProviderException` | Generic provider failure — including a named key that was never generated (`encrypt`/`decrypt`/`sign`/`verify`). |
| `UnsupportedOperationException` | `tokenize`/`detokenize` called against a provider without `ITokenProvider` support, or `encrypt`/`decrypt`/`sign`/`verify` called against a provider without `ICryptoSecretProvider` support. |

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
