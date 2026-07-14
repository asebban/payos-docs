# CLI tools

PayOS ships a family of command-line tools for managing applications, capabilities,
products, secrets, bundle encryption, and API documentation. Each is a standalone executable
installed into your `PATH` (typically `~/.payos/bin`).

## The tools

| Command | Module | Purpose | Doc |
| --- | --- | --- | --- |
| `apm` | payos-pm | **A**pplication **P**ackage **M**anager — install/uninstall applications. | [apm.md](apm.md) |
| `cpm` | payos-pm | **C**apability **P**ackage **M**anager — install/activate capabilities. | [cpm.md](cpm.md) |
| `ppm` | payos-pm | **P**roduct **P**ackage **M**anager — install/uninstall products. | [ppm.md](ppm.md) |
| `spm` | secret-service-filesystem | **S**ecret **P**ackage **M**anager — manage filesystem secrets. | [spm.md](spm.md) |
| `edc` | payosv2-packer | Encrypt/pack and decrypt/unpack bundles. | [edc.md](edc.md) |
| `pdoc` | pdoc | Generate OpenAPI 3.1 docs statically from annotations. | [pdoc.md](pdoc.md) |

## Common conventions

- The package managers (`apm`, `cpm`, `ppm`) are built with **picocli** and share global
  options: `--bundle-path` (default `.`), `-h`/`--help`, `-v`/`--version`.
- Exit codes: `0` success, `1` error, `2` usage error.
- Each tool installs via the module's `install-*.sh` / `install-*.ps1` script.

## Which tool for which unit

| You want to manage a… | Use |
| --- | --- |
| Standalone application | [`apm`](apm.md) |
| Reusable capability (extends target, activatable) | [`cpm`](cpm.md) |
| Product (a set of apps + server config) | [`ppm`](ppm.md) |
| Secret (filesystem provider) | [`spm`](spm.md) |

See [developer/application-model.md](../developer/application-model.md) for the distinction
between applications, capabilities, and products.

## Related

- [Operations: deployment](../operations/deployment.md)
- [Operations: bundle encryption](../operations/bundle-encryption.md)
- [Operations: secrets management](../operations/secrets-management.md)
- [Developer: API documentation](../developer/api-documentation.md)
