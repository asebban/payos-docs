# payos-kernel build profiles

**Created:** 2026-08-19  
**Last updated:** 2026-08-19

`payos-kernel` exposes two Maven build profiles that control which `ILicenseValidator` implementation is compiled into the JAR: a no-op validator for development and a file-checking validator for production. The design is explained in [architecture/license-validation-architecture.md](../architecture/license-validation-architecture.md); the configuration key is documented in [configuration/license-service.md](../configuration/license-service.md).

## How it works

The validator selection uses two extra source directories alongside the standard `src/main/java`:

| Directory | Always on classpath | Content |
| --- | --- | --- |
| `src/main/java` | Yes | All shared kernel sources |
| `src/main/java-dev` | Yes (added by default build via `build-helper-maven-plugin`) | `LicenseValidatorProvider` → `DevLicenseValidator` |
| `src/main/java-prod` | Only with `-Pprod` | `LicenseValidatorProvider` → `LicenseValidator` |

When `-Pprod` is active, `maven-compiler-plugin` excludes `src/main/java-dev/…/LicenseValidatorProvider.java` so the prod variant wins.

`BootServer` always calls `LicenseValidatorProvider.create()` — the same call site regardless of profile.

## Build commands

### Development build (default)

```bash
mvn package -DskipTests -f payos/pom.xml
```

The resulting JAR contains `DevLicenseValidator` as the active validator. No `license-file-path` is required at runtime.

### Production build

```bash
mvn package -DskipTests -Pprod -f payos/pom.xml
```

The resulting JAR contains `LicenseValidator` as the active validator. `license-file-path` must be set in `payos.json` and the file must exist on disk, otherwise the runtime exits immediately at boot.

### Runnable fat JAR (payos-runtime)

The profiles are inherited by the shade build. Run the same commands from `payos-runtime/pom.xml`:

```bash
# dev runtime
mvn package -DskipTests -f payos-runtime/pom.xml

# prod runtime
mvn package -DskipTests -Pprod -f payos-runtime/pom.xml
```

`payos-kernel` must already be installed in the local Maven repository before building `payos-runtime`:

```bash
mvn install -DskipTests -f payos/pom.xml           # dev
mvn install -DskipTests -Pprod -f payos/pom.xml    # prod
```

## Verifying which profile was used

Inspect the compiled class inside the JAR to confirm which validator is embedded:

```bash
# extract and decompile (requires javap on PATH)
jar xf payos-kernel-*.jar ma/s2m/payos/license/LicenseValidatorProvider.class
javap ma/s2m/payos/license/LicenseValidatorProvider.class
```

The output will reference either `DevLicenseValidator` or `LicenseValidator`.

## IDE setup

`src/main/java-dev` is registered as a source root in the default `<build>` section (not inside a profile), so the IDE resolves `LicenseValidatorProvider` without any special profile activation.

After pulling changes that modify the profiles or the `pom.xml`, reload the Maven project:

- **VS Code**: run **Java: Clean Language Server Workspace**, then **Maven: Reload project** from the Maven side panel.
- **IntelliJ IDEA**: click the **Reload All Maven Projects** button in the Maven tool window.

## Adding a custom validator for a specific environment

See [developer/license-validation.md](../developer/license-validation.md). Once the custom class is implemented, add a third source directory (`src/main/java-custom`) and a corresponding Maven profile following the same pattern as `prod`.
