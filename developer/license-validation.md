# Implementing a custom licence validator (obsolete — see v2)

**Last updated:** 2026-11-01

This page described `DevLicenseValidator` as hard-wired in `BootServer`, and `LicenseValidator` as a file-existence check only. Since 2026-10, dev/prod selection is runtime-driven (no build profile), and `LicenseValidator` performs full cryptographic validation.

**See instead: [license-validation-v2-2026-11-01.md](license-validation-v2-2026-11-01.md).**
