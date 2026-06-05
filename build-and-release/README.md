# Build & release

How the PayOS multi-module Maven project is built, versioned, and assembled into runnable
artifacts. This section is for engineers maintaining the platform itself.

## Contents

| Topic | Document |
| --- | --- |
| Build conventions (Java 21, Maven, plugins) | [build-conventions.md](build-conventions.md) |
| Versioning & release flow | [versioning.md](versioning.md) |
| Module map (what each module is, versions, artifacts) | [module-map.md](module-map.md) |

## At a glance

- **Java 21**, **Maven** multi-module build.
- A **parent POM** (`payos-parent`) centralizes the Java version and dependency versions.
- A **BOM** (`payos-bom`, artifact `payos-core-bom`) pins the kernel/platform versions for
  consumers.
- The **runtime** (`payos-runtime`) shades the kernel, transports, and standard connectors
  into one executable JAR (`ma.s2m.payos.BootServer`).
- Service/transport backends are separate modules producing **connector JARs**.

## Next

- [module-map.md](module-map.md) — the canonical list of modules.
- [versioning.md](versioning.md) — versions and release steps.
