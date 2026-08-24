# License service configuration

**Created:** 2026-08-19  
**Last updated:** 2026-08-19

The `license-file-path` key tells PayOS where to find the licence file on disk. Licence validation is the **first** thing `BootServer` performs after loading `payos.json` — before any service (database, queue, secrets, notifications) is initialised. If validation fails the process exits immediately.

The active validator is an `ILicenseValidator` implementation wired inside `BootServer`. The shipped implementations and how to supply a custom one are covered in [developer/license-validation.md](../developer/license-validation.md); the design is documented in [architecture/license-validation-architecture.md](../architecture/license-validation-architecture.md).

## Shape

```json
{
  "license-file-path": "/opt/payos/license/payos.lic"
}
```

The key lives at the top level of `payos.json` (not nested under a block).

## Keys

From `IConfigSpec`:

| Key | Default | Purpose |
| --- | --- | --- |
| `license-file-path` | — | Absolute (or relative) path to the licence file. Passed as-is to `ILicenseValidator.validate(config, licenseFilePath)`. |

## Behaviour by validator

| Active validator | `license-file-path` absent/null | File missing on disk |
| --- | --- | --- |
| `DevLicenseValidator` (default) | Ignored — no-op | Ignored — no-op |
| `LicenseValidator` | Logs error + `System.exit(1)` | Logs error + `System.exit(1)` |
| Custom implementation | Implementation-defined | Implementation-defined |

## Notes

- The path is resolved with `Paths.get(...).toAbsolutePath().normalize()` by `LicenseValidator`, so relative paths are resolved against the JVM working directory.
- The licence file content is not read or validated by `LicenseValidator` — only existence is checked. Cryptographic validation (signature, expiry, tenant scope) must be added in a custom `ILicenseValidator` implementation.
- Hot-reload does **not** re-run licence validation. The validation result holds for the lifetime of the process.
- File permissions: restrict read access to the OS user running the PayOS process (e.g. `chmod 640`). Do not expose the licence file path in logs or API responses.
