# Choosing the license validator (dev vs. production)

**Created:** 2026-10-31
**Last updated:** 2026-10-31
**Version:** v1

> Replaces `kernel-build-profiles.md`, which described a two-source-directory/Maven-profile mechanism (`src/main/java-dev`, `src/main/java-prod`, `mvn package -Pprod`) that has been removed entirely as too complex. There is no longer a build-time switch of any kind — read on for what replaced it.

## What changed

`payos-kernel` used to compile a different `LicenseValidatorProvider` depending on whether `-Pprod` was active, pulling in `DevLicenseValidator` or `LicenseValidator` from one of two parallel source directories. This has been simplified away: **there is now exactly one compiled `LicenseValidatorProvider`, in exactly one source directory**, and it picks the active validator at runtime, not at build time.

```java
public final class LicenseValidatorProvider {
    public static ILicenseValidator create() {
        return isDevModeEnabled() ? new DevLicenseValidator() : new LicenseValidator();
    }
    // isDevModeEnabled(): payos.license.devMode system property, else PAYOS_LICENSE_DEV_MODE env var;
    // unset or anything but an explicit "false" means dev mode.
}
```

The same compiled JAR — whether built from `payos/pom.xml` or the shaded `payos-runtime` — contains both implementations. There is nothing to choose, verify, or reload in the IDE at build time anymore.

## Building

```bash
# single module
mvn -q -DskipTests -f payos/pom.xml package

# runnable fat JAR (payos-runtime)
mvn -q -DskipTests -f payos-runtime/pom.xml package
```

No `-Pprod` flag exists; passing one has no effect (there is no profile by that name left in `payos/pom.xml` or `payos-runtime/pom.xml`).

## Choosing dev vs. production at runtime

Set this when **launching** the JAR, not when building it:

```bash
# development (default — no need to set anything)
java -jar payos-runtime.jar

# production
java -Dpayos.license.devMode=false -jar payos-runtime.jar
# or
PAYOS_LICENSE_DEV_MODE=false java -jar payos-runtime.jar
```

Full configuration reference, including everything the real validator needs once it's active (licence file location, editor secret provider): [configuration/license-service-v2-2026-11-01.md](../configuration/license-service-v2-2026-11-01.md). Design rationale: [architecture/license-validation-architecture-v4-2026-11-16.md](../architecture/license-validation-architecture-v4-2026-11-16.md).

## Verifying which validator is active at runtime

There is no compiled artifact to inspect anymore (both implementations are always present). Check the running process's effective property/environment variable, or look for the `"License validation successful."` log line immediately after startup — it is only ever logged, for the real validator, before the rest of configuration loads; `DevLicenseValidator` logs nothing (it is a silent no-op, run after config load, exactly like before).

## Adding a custom validator for a specific environment

See [developer/license-validation-v2-2026-11-01.md](../developer/license-validation-v2-2026-11-01.md). There is currently no pluggable third option at the `LicenseValidatorProvider` level — a third implementation requires editing the kernel directly (no source-directory/profile pattern to extend anymore, by design).
