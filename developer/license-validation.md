# Implementing a custom licence validator

**Created:** 2026-08-19  
**Last updated:** 2026-08-19

PayOS validates the licence as the very first step of `BootServer.main`, before any service is initialised. The validation logic is behind the `ILicenseValidator` interface so it can be swapped per environment without touching the boot sequence. Configuration reference: [configuration/license-service.md](../configuration/license-service.md). Architecture overview: [architecture/license-validation-architecture.md](../architecture/license-validation-architecture.md).

## The interface

```java
package ma.s2m.payos.license;

import java.util.Map;

public interface ILicenseValidator {

    /** Called during boot with the path read from payos.json (`license-file-path`). */
    void validate(Map<String, Object> configuration, String licenseFilePath) throws Exception;

    /** Called when the licence content is available in memory (e.g. from a secrets store). */
    void validate(Map<String, Object> configuration, byte[] licenseData) throws Exception;
}
```

Throw `LicenseValidationException` (or any `Exception`) to signal a validation failure. `BootServer` catches any exception, logs it, and calls `System.exit(1)`.

## Shipped implementations

| Class | Behaviour | When to use |
| --- | --- | --- |
| `DevLicenseValidator` | No-op — always passes silently | Local development |
| `LicenseValidator` | Checks `licenseFilePath` is non-null and the file exists | Staging / production (file-presence check only) |

`BootServer` currently wires `DevLicenseValidator`. Switch to `LicenseValidator` or a custom implementation for environments where a licence must be present.

## Writing a custom validator

Implement `ILicenseValidator` in your own module and override both `validate` methods. The `configuration` map contains the full content of `payos.json`, giving access to any custom parameters you need (environment name, tenant scope, runtime version, etc.).

```java
package com.example.license;

import ma.s2m.payos.license.ILicenseValidator;
import ma.s2m.payos.license.LicenseValidationException;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Map;

public class SignedLicenseValidator implements ILicenseValidator {

    private static final PublicKey EMBEDDED_PUBLIC_KEY = loadPublicKey();

    @Override
    public void validate(Map<String, Object> configuration, String licenseFilePath)
            throws LicenseValidationException {
        if (licenseFilePath == null || licenseFilePath.isBlank()) {
            throw new LicenseValidationException("license-file-path is required");
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
        // Parse and verify cryptographic signature against EMBEDDED_PUBLIC_KEY.
        // Check expiry, permitted runtime version, tenant scope, etc.
        // Throw LicenseValidationException on any failure — never log the raw bytes.
    }

    private static PublicKey loadPublicKey() {
        // Load from a resource bundled into the JAR — never from config or disk.
        ...
    }
}
```

### Wiring the validator in `BootServer`

Replace the one-liner that instantiates the active validator:

```java
// before
ILicenseValidator licenseValidator = new DevLicenseValidator();

// after
ILicenseValidator licenseValidator = new SignedLicenseValidator();
```

## Security guidelines

- **Embed the public key in the binary**, not in `payos.json` or on disk, so substituting the config cannot bypass signature verification.
- **Do not log licence bytes or the raw file content** — the payload may contain tenant-identifying or commercially sensitive data.
- **Do not throw generic exceptions that expose internal detail** — return a short, opaque message in `LicenseValidationException`.
- **Fail closed**: if the validator cannot determine validity (I/O error, malformed file, clock skew beyond tolerance), throw `LicenseValidationException` rather than silently passing.
