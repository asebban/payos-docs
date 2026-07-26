# payos-secret-api

**Module:** `payos-secret-api`  
**Version:** `1.0.0-RELEASE`  
**Group:** `ma.s2m.payos`

## Overview

`payos-secret-api` defines the secret-management contracts used by PayOS and secret-provider connectors.

It contains:

- provider interfaces (`ISecretProvider`, `ISecretProviderFactory`)
- optional capability interfaces (`IVersionedSecretProvider`, `ICryptoSecretProvider`, `IAsymmetricCryptoSecretProvider`, `ICertificateSecretProvider`, `IWatchableSecretProvider`, `ITokenProvider`)
- common models (`SecretValue`, `SecretMetadata`, `SecretCapability`, `KeyAlgorithm`)
- shared exception hierarchy
- audit models
- an abstract base implementation (`AbstractSecretProvider`) used by providers

This module is API-only and does not ship a concrete secret backend.

## Capability Interfaces

| Interface | Capability flag | Purpose |
|---|---|---|
| `IVersionedSecretProvider` | `VERSION` | Secret versioning and rollback |
| `ICryptoSecretProvider` | `CRYPTO` | Generate a named internal **symmetric** key, then encrypt/decrypt/sign/verify data with it by name |
| `IAsymmetricCryptoSecretProvider` | `ASYMMETRIC_CRYPTO` | Generate a named internal **key pair**, sign/verify with it by name, or verify against an arbitrary externally-supplied public key. Not implemented by either shipped provider yet — planned: Vault (Transit `rsa`/`ecdsa` key types) and filesystem (Bouncy Castle). See [architecture/secret-provider-architecture.md](../payos-docs/architecture/secret-provider-architecture.md). |
| `ICertificateSecretProvider` | `CERTIFICATE_AUTHORITY` | Per-tenant CA: issue X.509 certificates (from a CSR, or for an internally-held key pair), fetch the CA certificate, revoke. Not implemented by either shipped provider yet. |
| `IWatchableSecretProvider` | — | Live reload on secret changes |
| `ITokenProvider` | `TOKENIZE` | Store sensitive values behind opaque tokens (UUID v4) |

Providers declare supported capabilities through `ISecretProvider.capabilities()` returning a `Set<SecretCapability>`.
Available values: `GET`, `SET`, `DELETE`, `LIST`, `DESCRIBE`, `VERSION`, `TOKENIZE`, `CRYPTO`, `ASYMMETRIC_CRYPTO`, `CERTIFICATE_AUTHORITY`.

The two connectors shipped with PayOS both implement `ITokenProvider` and `ICryptoSecretProvider`,
but their `VERSION` support and crypto backends differ: `FileSystemSecretProvider` implements `IVersionedSecretProvider` (real version history via archived envelopes) and backs `ICryptoSecretProvider` with named AES-256 keys generated and kept internally, never exposed (see `secret-service-filesystem`'s README for the CLI). `VaultSecretProvider` only reports Vault KV v2's `current_version` through `describeSecret` (no `IVersionedSecretProvider`), and backs `ICryptoSecretProvider` with the Vault **Transit** secrets engine — named keys live entirely in Vault, encrypt/decrypt use Transit's `encrypt`/`decrypt` endpoints, and sign/verify use Transit's `hmac`/`verify` endpoints since the key type (`aes256-gcm96`) is symmetric (see `secret-service-vault`'s README, section 5).

### ICryptoSecretProvider

```java
public interface ICryptoSecretProvider {
    void    generateKey(String tenantId, String keyName)                       throws SecretProviderException;
    boolean keyExists(String tenantId, String keyName)                         throws SecretProviderException;
    byte[]  encrypt(String tenantId, String keyName, byte[] plaintext)         throws SecretProviderException;
    byte[]  decrypt(String tenantId, String keyName, byte[] ciphertext)        throws SecretProviderException;
    byte[]  sign(String tenantId, String keyName, byte[] data)                 throws SecretProviderException;
    boolean verify(String tenantId, String keyName, byte[] data, byte[] sig)   throws SecretProviderException;
}
```

`keyName` identifies a key that lives entirely inside the provider — callers generate it once via
`generateKey` and thereafter only ever reference it by name; the raw key material is never
returned by any method on this interface.

### IAsymmetricCryptoSecretProvider

Not implemented by either shipped provider yet. Same "identify by name, raw key material never
returned" contract as `ICryptoSecretProvider`, but `keyName` identifies a **key pair** instead of a
symmetric key — `generateKeyPair`/`getPublicKey` only ever return the public half.

```java
public interface IAsymmetricCryptoSecretProvider {
    byte[]  generateKeyPair(String tenantId, String keyName, KeyAlgorithm algorithm)   throws SecretProviderException;
    byte[]  getPublicKey(String tenantId, String keyName)                              throws SecretProviderException;
    boolean keyPairExists(String tenantId, String keyName)                             throws SecretProviderException;
    byte[]  signWithKeyPair(String tenantId, String keyName, byte[] data)               throws SecretProviderException;
    boolean verifyWithKeyPair(String tenantId, String keyName, byte[] data, byte[] sig) throws SecretProviderException;
    boolean verifyWithPublicKey(byte[] publicKey, byte[] data, byte[] signature)        throws SecretProviderException;
}
```

Public keys are PEM-encoded X.509 SubjectPublicKeyInfo. `signWithKeyPair`/`verifyWithKeyPair` are
named distinctly from `ICryptoSecretProvider#sign`/`verify` (rather than overloading those names)
because a class implementing both interfaces — a Vault-backed provider, for instance — would
otherwise have one method body servicing two same-signature abstract methods with different
semantics (symmetric HMAC vs. asymmetric signature). `verifyWithKeyPair` checks against the key
pair's own stored public key (round-trip case: you signed it, you're checking it);
`verifyWithPublicKey` is stateless and takes no `tenantId`/`keyName` — it's for authenticating
something signed by a third party you only know by public key, not one this provider generated.
`KeyAlgorithm` values: `RSA_2048`, `RSA_3072`, `RSA_4096`, `EC_P256`.

### ICertificateSecretProvider

Not implemented by either shipped provider yet. Per-tenant certificate authority — every issuance
path is designed so the subject's private key never reaches the provider:

```java
public interface ICertificateSecretProvider {
    X509Certificate getCACertificate(String tenantId)                                          throws SecretProviderException;
    X509Certificate issueCertificateFromCsr(String tenantId, String subjectName, byte[] csrPem) throws SecretProviderException;
    X509Certificate issueCertificateForKey(String tenantId, String subjectName, String keyName) throws SecretProviderException;
    X509Certificate getCertificate(String tenantId, String name)                                throws SecretProviderException;
    List<String>    listCertificates(String tenantId)                                           throws SecretProviderException;
    void            revokeCertificate(String tenantId, String serialNumber)                     throws SecretProviderException;
}
```

Two issuance paths, both keeping the subject's private key out of the provider's hands:
- `issueCertificateFromCsr` — for an external client: the client generates its own key pair and
  sends only a PEM-encoded PKCS#10 CSR; the provider never sees the private key. This is the path
  for authenticating an external client against the provider (e.g. mTLS).
- `issueCertificateForKey` — for a certificate the provider itself will use (e.g. its own service
  identity): binds a cert to a key pair already generated via
  `IAsymmetricCryptoSecretProvider#generateKeyPair`, whose private key already never left the
  provider and stays that way; future proof-of-possession goes through that interface's
  `signWithKeyPair`.

### ITokenProvider

`ITokenProvider` separates token vault operations from secret vault operations.
A provider that implements it must advertise `SecretCapability.TOKENIZE`.

```java
public interface ITokenProvider {
    String       tokenize(String tenantId, byte[] sensitiveValue) throws SecretProviderException;
    byte[]       detokenize(String tenantId, String token)        throws TokenNotFoundException, SecretProviderException;
    boolean      tokenExists(String tenantId, String token)       throws SecretProviderException;
    void         revokeToken(String tenantId, String token)       throws TokenNotFoundException, SecretProviderException;
    List<String> listTokens(String tenantId)                      throws SecretProviderException;
}
```

### Exception hierarchy

All four exceptions extend `RuntimeException` directly (none extends `SecretProviderException`
— they're siblings, not a subtype hierarchy):

| Exception | Extends | Thrown when |
|---|---|---|
| `SecretProviderException` | `RuntimeException` | Generic provider failure (I/O, encryption, config) |
| `SecretNotFoundException` | `RuntimeException` | Requested secret (or version) does not exist |
| `SecretAccessDeniedException` | `RuntimeException` | Caller lacks permission, or tenantId is invalid |
| `TokenNotFoundException` | `RuntimeException` | Token does not exist or has been revoked |

## SPI Contract

A provider implementation module must:

1. Implement `ISecretProviderFactory`
2. Return a unique `type()` value (for example `filesystem`)
3. Register the factory in `META-INF/services/ma.s2m.payos.secret.api.ISecretProviderFactory`

## Module Structure

```text
payos-secret-api/
├── pom.xml
└── src/main/java/ma/s2m/payos/secret/
    ├── api/
    ├── model/
    ├── exception/
    ├── audit/
    └── spi/
```

## Build

```bash
mvn -q -DskipTests compile
mvn -q test
mvn -q -DskipTests package
```
