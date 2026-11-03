# Implementing a custom licence validator

**Created:** 2026-08-19
**Last updated:** 2026-11-01
**Version:** v2

> Supersedes the v0 revision of this page (2026-08-19), which still described `DevLicenseValidator` as hard-wired in `BootServer` and `LicenseValidator` as a file-existence check only. See [architecture/license-validation-architecture-v3-2026-11-01.md](../architecture/license-validation-architecture-v3-2026-11-01.md) for the full design this page is a practical companion to.
>
> **v2 (2026-11-01)**: `editor-tenant-id` renamed to `editor-id` in the security guidelines below — not a PayOS tenant.

PayOS validates the licence as one of the very first things `BootServer.main` does — for the real `LicenseValidator`, even before `payos.json` is read off disk, since the licence's own `key_envelope` is what makes an encrypted `payos.json` decryptable in the first place (see the architecture doc, §2 and §7). The validation logic is behind the `ILicenseValidator` interface so it can be swapped per environment without touching the boot sequence. Configuration reference: [configuration/license-service-v2-2026-11-01.md](../configuration/license-service-v2-2026-11-01.md).

## The interface

```java
package ma.s2m.payos.license;

import java.util.Map;

public interface ILicenseValidator {

    /** Called with the path to the licence file (resolved by LicenseFileLocator for the real validator). */
    void validate(Map<String, Object> configuration, String licenseFilePath) throws Exception;

    /** Called when the licence content is already available in memory. */
    void validate(Map<String, Object> configuration, byte[] licenseData) throws Exception;
}
```

Throw `LicenseValidationException` (or any `Exception`) to signal a validation failure. `BootServer` catches any exception, logs it, and calls `System.exit(1)`.

## Shipped implementations

| Class | Behaviour | When active |
| --- | --- | --- |
| `DevLicenseValidator` | No-op — always passes silently, never touches `LicenseContext` | **Default** — active whenever `payos.license.devMode`/`PAYOS_LICENSE_DEV_MODE` is unset or not explicitly `false` |
| `LicenseValidator` | Full cryptographic validation: RSA signature, validity window, `issued_at`, `licensee.client_id`/`key_envelope.key_id` consistency, machine fingerprint, RSA-unwraps the source-decryption key into `LicenseContext` | Only when the dev-mode toggle is explicitly set to `false` |

`LicenseValidatorProvider.create()` — called once by `BootServer` — picks between the two; see [configuration/license-service-v2-2026-11-01.md](../configuration/license-service-v2-2026-11-01.md) for the exact resolution rules. There is no longer a build-time switch (no Maven profile, no alternate source directory) — the same compiled JAR contains both implementations and the choice is made purely at runtime.

## Testing against a real licence locally

`LicenseValidator`'s own test suite (`payos/src/test/java/ma/s2m/payos/license/LicenseValidatorTest.java`) builds a genuinely signed, enveloped licence using `payos-license-generator`'s real classes (test-scope dependency) against a real `FileSystemSecretProvider` — useful as a template if you need to exercise the real validator end to end rather than mocking it. `payosv2-packer`'s `MainLicenseKeyTest` does the same for the packer side, including a full pack/unpack round trip with the key unwrapped from that licence.

## Writing a custom validator

Implement `ILicenseValidator` in your own module and override both `validate` methods. The `configuration` map is whatever the caller passes — for the real validator's pre-config boot-time call, this is an environment-variable-derived map (see the architecture doc §7), not `payos.json` itself.

```java
package com.example.license;

import ma.s2m.payos.license.ILicenseValidator;
import ma.s2m.payos.license.LicenseValidationException;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Map;

public class CustomLicenseValidator implements ILicenseValidator {

    @Override
    public void validate(Map<String, Object> configuration, String licenseFilePath)
            throws LicenseValidationException {
        if (licenseFilePath == null || licenseFilePath.isBlank()) {
            throw new LicenseValidationException("No licence file path provided");
        }
        Path path = Paths.get(licenseFilePath).toAbsolutePath().normalize();
        if (!path.toFile().exists()) {
            throw new LicenseValidationException("Licence file not found: " + path);
        }
        try {
            validate(configuration, Files.readAllBytes(path));
        } catch (LicenseValidationException e) {
            throw e;
        } catch (Exception e) {
            throw new LicenseValidationException("Failed to read licence file: " + e.getMessage(), e);
        }
    }

    @Override
    public void validate(Map<String, Object> configuration, byte[] licenseData)
            throws LicenseValidationException {
        // Parse, verify signature/expiry/whatever this environment needs.
        // Throw LicenseValidationException on any failure — never log the raw bytes.
    }
}
```

### Wiring the validator

There is no single wiring point to edit anymore — `BootServer` calls `LicenseValidatorProvider.create()`, which only knows about `DevLicenseValidator` and `LicenseValidator`. To ship a third implementation, either fork `LicenseValidatorProvider`'s selection logic in your own build, or — if the custom validator should also get the "runs before `payos.json` is read" treatment — extend `BootServer`'s `realLicenseValidator` branch (see the architecture doc §2) to recognize it. There is currently no extension point for this beyond editing the kernel directly; raise this with the team if you need a pluggable third implementation.

## Security guidelines

- **Never trust a signer identity declared inside the document being verified** — `LicenseValidator` always resolves the verification key pair from the *verifier's own* configuration (`editor-id`/`editor-key-name`), never from a field inside the licence. Follow the same rule in a custom validator.
- **Do not log licence bytes or the raw file content** — the payload may contain tenant-identifying or commercially sensitive data.
- **Do not throw generic exceptions that expose internal detail** — return a short, opaque message in `LicenseValidationException`.
- **Fail closed**: if the validator cannot determine validity (I/O error, malformed file, clock skew beyond tolerance), throw `LicenseValidationException` rather than silently passing.
- **Zero any raw key material** you reconstruct as soon as you're done with it — see `SplitKeyGuard`/`CryptoService.setKey(byte[])` in `payos-foundation` for the pattern the shipped validator follows.

## Next

- [architecture/license-validation-architecture-v3-2026-11-01.md](../architecture/license-validation-architecture-v3-2026-11-01.md) — full design.
- [configuration/license-service-v2-2026-11-01.md](../configuration/license-service-v2-2026-11-01.md) — configuration/environment-variable reference.
