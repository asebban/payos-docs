# Technology stack

PayOS targets **Java 21** and builds with Maven. All third-party versions are pinned
centrally in the [parent POM](../build-and-release/build-conventions.md) so that every
module agrees on a single version of each dependency.

## Runtime platform

| Concern | Technology | Version |
| --- | --- | --- |
| Language / JVM | Java | 21 (`maven.compiler.source/target=21`) |
| Build | Maven | parent-managed plugin versions |
| HTTP server | Undertow | `2.3.11.Final` |
| Scripting engine | GraalVM Polyglot / JS | `24.1.1` |
| JSON | Jackson Databind | `2.16.1` (kernel) / `2.17.2` (parent-managed) |
| Authentication | pac4j (core + OIDC) | `6.0.0` |
| Logging API | SLF4J | `2.0.12` |
| Logging backend | Logback | `1.5.13` |
| Messaging | NATS client (`jnats`) | `2.17.0` |
| ORM (data service) | Hibernate | `5.6.15.Final` |
| CLI framework (tooling) | picocli | `4.7.5` |

> Where the kernel `pom.xml` and the parent POM list slightly different Jackson versions,
> the value that actually applies is the one resolved by Maven for the module being built.
> The authoritative, per-module list is in
> [build-and-release/module-map.md](../build-and-release/module-map.md).

## Testing

| Concern | Technology | Version |
| --- | --- | --- |
| Test framework | JUnit Jupiter (JUnit 5) | `5.10.2` |
| Assertions | AssertJ | `3.25.3` |
| Test runner | `maven-surefire-plugin` | `3.2.5` |

The project conventions (JUnit 5 + AssertJ, `@TempDir` for filesystem tests, asserting both
exception type and message fragment) are described in
[build-and-release/build-conventions.md](../build-and-release/build-conventions.md).

## Packaging

| Concern | Technology | Version |
| --- | --- | --- |
| Fat-JAR / shading | `maven-shade-plugin` | `3.5.1` |
| Compiler | `maven-compiler-plugin` | `3.13.0` (parent-managed) |

The runnable distribution (`payos-runtime`) is produced by the shade plugin with the main
class set to `ma.s2m.payos.BootServer`. See
[operations/deployment.md](../operations/deployment.md).

## Why these choices

- **Undertow** — a lightweight, non-blocking HTTP core that pairs with Java 21 virtual
  threads for blocking business logic (see [architecture/request-processing.md](../architecture/request-processing.md)).
- **GraalVM Polyglot/JS** — lets business logic be authored in JavaScript while running in a
  hardened, host-controlled sandbox (see [architecture/scripting-engine.md](../architecture/scripting-engine.md)).
- **pac4j OIDC** — standards-based authentication that integrates with enterprise IdPs.
- **SLF4J + Logback + MDC** — structured logging with correlation/tenant identifiers for
  regulated traceability (see [operations/observability.md](../operations/observability.md)).
- **SPI everywhere** — database, queue, secret, and webhook providers are discovered at
  runtime so the same core supports many backends without recompilation.
