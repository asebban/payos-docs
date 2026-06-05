# Secrets usage (`$Secrets`)

When a [secret service](../configuration/secret-service.md) is configured, scripts receive
the `$Secrets` binding — an `ISecretProvider`. Use it to read and manage secrets (API keys,
credentials, tokens) without embedding them in code or configuration. This page covers usage
from JavaScript; provider configuration is in
[configuration/secret-service.md](../configuration/secret-service.md) and operational setup
(Vault, rotation, the `spm` CLI) is in
[operations/secrets-management.md](../operations/secrets-management.md).

## Tenant scoping is automatic

Secrets are stored and retrieved **per tenant**. The provider uses the request's current
tenant, so you never pass a tenant explicitly. The base provider validates the tenant and
writes an audit entry (`payos.secret.audit`) for every operation.

## Reading a secret

```javascript
function execute(request, controlData) {
    var key = $Secrets.getSecret("psp-api-key");   // returns a SecretValue
    try {
        // use key.asString() / key bytes to call the downstream system
        return callPsp(key.asString(), controlData.amount);
    } finally {
        key.close();   // SecretValue is auto-closeable; zeroes its memory
    }
}
```

> `SecretValue` is auto-closeable and zeroes its backing memory on close. Read it, use it,
> and close it promptly; do not store it in long-lived variables.

## Writing and deleting

```javascript
$Secrets.setSecret("psp-api-key", newValue);
$Secrets.deleteSecret("obsolete-key");
```

## Listing and describing

```javascript
var names = $Secrets.listSecrets();
var meta  = $Secrets.describe("psp-api-key");   // SecretMetadata: name, type, version, timestamps
```

## Capabilities differ by provider

Not every provider supports every operation. Query support with `capabilities()`:

| Capability | `filesystem` | `vault` |
| --- | --- | --- |
| `GET` / `SET` / `DELETE` / `LIST` / `DESCRIBE` / `VERSION` | ✅ | ✅ |
| `TOKENIZE` | ✅ | ❌ |

```javascript
var caps = $Secrets.capabilities();   // Set<SecretCapability>
```

Optional extended interfaces (when the provider implements them): `ITokenProvider`
(tokenization), `IVersionedSecretProvider`, `IWatchableSecretProvider`, `ICryptoSecretProvider`,
`ICertificateSecretProvider`. See [architecture/extensibility.md](../architecture/extensibility.md).

## Error handling

Provider operations may throw:

| Exception | Meaning |
| --- | --- |
| `SecretNotFoundException` | The named secret does not exist for the tenant. |
| `SecretAccessDeniedException` | The tenant is not allowed to access the secret. |
| `TokenNotFoundException` | A detokenize lookup failed (tokenizing providers). |
| `SecretProviderException` | Generic provider failure. |

```javascript
try {
    var s = $Secrets.getSecret("psp-api-key");
    // ...
} catch (e) {
    throw new BusinessException("PSP key unavailable");
}
```

## If `$Secrets` is missing

`$Secrets` is injected only when `secret-service.enabled` is `true` and a provider `type`
(`filesystem` or `vault`) is configured with a valid connector JAR. See
[configuration/secret-service.md](../configuration/secret-service.md).

## Next

- [Configuration: secret service](../configuration/secret-service.md)
- [Operations: secrets management](../operations/secrets-management.md)
- [CLI: spm](../cli-tools/spm.md)
