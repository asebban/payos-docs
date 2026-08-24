# Build conventions

The PayOS platform is a Java 21 / Maven multi-module codebase. This page captures the build
conventions that apply across modules.

## Toolchain

| Aspect | Value |
| --- | --- |
| Language level | Java 21 (`maven.compiler.source` / `target = 21`) |
| Build tool | Maven 3.9+ |
| Compiler plugin | `maven-compiler-plugin:3.13.0` |
| Test plugin | `maven-surefire-plugin:3.2.5` |
| Shade plugin | `maven-shade-plugin:3.5.1` (runtime/packer fat JARs) |

## Source layout

```
src/main/java/ma/s2m/payos/...      production code
src/test/java/ma/s2m/payos/...      tests
```

- Package root: `ma.s2m.payos`.
- Interfaces are prefixed with `I` (e.g. `IServer`, `IQueueClient`, `ISecretProvider`).

## Dependencies

Core stack versions (managed via the [parent POM](versioning.md) and the
[BOM](module-map.md)):

| Library | Version |
| --- | --- |
| Undertow | 2.3.18.Final |
| GraalVM Polyglot/JS | 24.1.1 |
| Jackson Databind | 2.16.1 / 2.17.2 |
| pac4j (core/oidc) | 6.0.0 |
| SLF4J | 2.0.12 |
| Logback | 1.5.13 |
| NATS (jnats) | 2.17.0 |
| Hibernate | 5.6.15.Final |
| picocli | 4.7.5 |
| JUnit Jupiter | 5.10.2 |
| AssertJ | 3.25.3 |

> Always pin explicit versions in `pom.xml`; do not introduce unversioned dependencies.

## Testing conventions

- JUnit 5 + AssertJ.
- Prefer focused unit tests (config/security/utility behavior).
- For exception tests, assert both the exception type and a key message fragment.
- Use `@TempDir` for filesystem tests rather than hardcoded paths.

## Packaging the runtime

The runnable server is produced by `payos-runtime` via the shade plugin, with the manifest
main class `ma.s2m.payos.BootServer`. The packer (`payosv2-packer`) likewise shades to a
runnable `edc`. Do not break the `BootServer` entrypoint contract.

## Logging conventions

- Use SLF4J (`logger.info/warn/error`); avoid `System.out`.
- Avoid `printStackTrace()` in new code.

## Building locally

```bash
# build a single module
mvn -q -DskipTests -f <module>/pom.xml package

# build the runnable server (and what it shades)
mvn -q -DskipTests -f payos-runtime/pom.xml package
```

## Next

- [versioning.md](versioning.md)
- [module-map.md](module-map.md)
- [kernel-build-profiles.md](kernel-build-profiles.md) — dev vs production build profiles for payos-kernel
