# Versioning & release

PayOS versions are centralized in the **parent POM** and surfaced to consumers through a
**BOM**. This page describes the versioning model and the release helper scripts.

## Where versions live

| Concern | Owner |
| --- | --- |
| Java version + dependency versions | `payos-parent` (parent POM, `1.0.0-RELEASE`) |
| Platform/kernel version pin for consumers | `payos-bom` (artifact `payos-core-bom`, `1.2.0-RELEASE`) |
| Kernel version | `payos-bom` owns `payos-kernel.version` (`1.3.0-RELEASE`) |

Consumers import the BOM (scope `import`) to align on a consistent set of PayOS module
versions without repeating each version.

## Current module versions

| Module | Version |
| --- | --- |
| payos-parent | 1.0.0-RELEASE |
| payos-bom (`payos-core-bom`) | 1.2.0-RELEASE |
| payos-kernel | 1.3.0-RELEASE |
| payos-runtime | 1.3.0-RELEASE |
| payos-server-http | 1.2.0 |
| payos-server-tcp | 1.0.6 |
| payos-server-queue | 1.0.6 |
| dynamic-database-service | 1.1.7 |
| queue-service-nats | 1.0.6 |
| webhook-service-http | 1.0.3 |
| payos-secret-api | 1.0.0 |
| secret-service-filesystem | 1.0.0 |
| secret-service-vault | 1.0.0 |
| payosv2-packer (`edc`) | 1.2.0-RELEASE |
| payos-pm (`apm`/`cpm`/`ppm`) | 1.1.2-RELEASE |
| pdoc | 1.0.0-SNAPSHOT |

(See [module-map.md](module-map.md) for what each module does and the versions embedded in
the runtime.)

## Release helper scripts

`payos-parent/scripts/update-payos-version.{ps1,sh}` updates versions consistently. It
supports several modes:

| Mode | Updates |
| --- | --- |
| `kernel` | The kernel version. |
| `module` | A single module's version. |
| `parent` | The parent POM version. |
| `new-module` | Onboards a new module. |
| `release` | A coordinated release bump. |

## Release flow (typical)

1. Bump versions with the appropriate `update-payos-version` mode.
2. Build and test the affected modules.
3. Rebuild `payos-runtime` so the fat JAR embeds the new module versions.
4. Update `payos-bom` so consumers pick up the new pins.

## Next

- [module-map.md](module-map.md)
- [build-conventions.md](build-conventions.md)
