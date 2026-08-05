Created: 2026-08-06
Last updated: 2026-08-06
Version: v1

# Runtime Compatibility Checks

Applications and capabilities can declare which payos-runtime versions they're compatible with, and the platform checks that declaration at two independent points — once when `apm`/`cpm`/`ppm` install the package, and again every time `BootServer` starts up — so a mismatch is caught whether it was introduced by installing an incompatible package or by upgrading the runtime underneath already-installed packages. This page is the single consolidated reference for the whole feature; the individual CLI/config pages linked throughout stay the place-specific reference for their own tool.

## Why two enforcement points

`cpm`/`apm --install --runtime-version <v>` only catches a mismatch at the moment a package is installed. Two things it cannot catch on its own: a capability dropped into `.capabilities/` by hand (bypassing `cpm` entirely), and a runtime binary upgraded after the fact without anyone re-running install against every already-installed package. `BootServer` re-checks every declared bound against the actual running runtime version on every boot, closing both gaps. The two checks share the same comparison logic (`RuntimeCompatibility`, see below) and the same manifest fields — they're one feature expressed at two lifecycle moments, not two separate features.

## The version range

A manifest declares an optional `minRuntimeVersion` and/or `maxRuntimeVersion` — both inclusive, both independently optional, no range-expression grammar (no `>=`/`<`/caret ranges). Absence of both means "no constraint, compatible with anything." Versions are compared as plain `MAJOR.MINOR.PATCH` triples: any `-suffix` (PayOS artifacts use `-RELEASE`) is stripped before comparing, and missing trailing segments default to `0` (so `"1.10"` compares as `1.10.0`). This comparison and a human-readable range formatter (`>=1.10.0, <=2.0.0`, or `(no bounds declared)`) live in one shared class: `ma.s2m.payos.config.RuntimeCompatibility` (`payos-foundation`, package `ma.s2m.payos.config`) — both `payos-pm` (install-time) and `payos-kernel`'s `BootServer` (boot-time) call the same code, so the verdict is identical regardless of which check ran.

## Where the runtime version comes from

The payos-runtime version isn't visible anywhere at runtime by default — `payos-kernel`'s own version is just a build-time Maven coordinate. Two things had to be added:

- **Build-time stamping**: `payos-runtime/pom.xml`'s `maven-shade-plugin` configuration adds `Implementation-Version: ${project.version}` to the shaded jar's `META-INF/MANIFEST.MF` via a `manifestEntries` block on the `ManifestResourceTransformer`.
- **In-process read**: `ma.s2m.payos.config.PayOSConfig.getRuntimeVersion()` (`payos-kernel`) reads that attribute via `Package.getImplementationVersion()`. Returns `"unknown"` (never `null`) when running from unpacked classes — dev/test, or any invocation that isn't the assembled shaded jar — so callers never need a null check. `BootServer.main()` logs this at the very start of boot (`Starting payos-runtime version 1.11.0-RELEASE`).
- **Offline read (no JVM needed)**: `ma.s2m.payos.pm.RuntimeVersionReader` (`payos-pm`) reads the same `Implementation-Version` attribute directly from a jar file on disk via `java.util.jar.JarFile`, for tooling that needs to know a target runtime's version without starting it.

## Declaring compatibility

### Capabilities (`manifest.json`, read by `cpm`)

```json
{
  "id": "payment-links",
  "name": "Payment Links",
  "version": "1.2.0",
  "minRuntimeVersion": "1.10.0",
  "maxRuntimeVersion": "2.0.0"
}
```

Model: `ma.s2m.payos.pm.cpm.CapabilityManifest` (`payos-pm`). Full field table: [cli-tools/cpm.md](../cli-tools/cpm.md).

### Applications (`manifest.json`, read by `apm` — and by `ppm` for a product's constituent applications, see below)

```json
{
  "id": "my-app",
  "version": "1.0.0",
  "base.path": "/opt/payos/apps/my-app",
  "minRuntimeVersion": "1.10.0",
  "maxRuntimeVersion": "2.0.0"
}
```

Model: `ma.s2m.payos.pm.apm.ApplicationManifest` (`payos-pm`). The two fields are also mirrored into `bootstrap.json`'s `applications[]` entry (`IConfigSpec.Applications.Application.MIN_RUNTIME_VERSION`/`MAX_RUNTIME_VERSION`, JSON keys `minRuntimeVersion`/`maxRuntimeVersion`) — that's what `BootServer`'s boot-time audit actually reads, since `manifest.json` itself is a transient install-time input, not something the running server re-reads. Full field table: [configuration/bootstrap-reference.md](../configuration/bootstrap-reference.md) and [cli-tools/apm.md](../cli-tools/apm.md).

### Applications installed as part of a product (`ppm install`)

A product manifest's `applications[]` entries only declare `id`/`basePath`/`version` — not `minRuntimeVersion`/`maxRuntimeVersion`. Rather than requiring every product manifest to repeat those bounds for each constituent application, `ppm install` also checks for `<appDir>/manifest.json` after the application directory is available (whether it was already present or freshly pulled from the `applicationCatalog`), and merges its optional fields — including `minRuntimeVersion`/`maxRuntimeVersion` — into the `bootstrap.json` entry, on top of the `id`/`basePath`/`version` the product manifest resolved. Absent `manifest.json`: no-op, the entry is built solely from the product manifest as before. Shared merge logic: `ma.s2m.payos.pm.apm.ApplicationManifestMerge`, used by both `ApplicationManager.install()` (`apm`) and `ProductManager.ensureApplicationPresent()` (`ppm`), so a per-application `manifest.json` is honoured identically regardless of which CLI pulled it. Details: [cli-tools/ppm.md](../cli-tools/ppm.md#per-application-manifestjson-constituent-applications).

## Enforcement at install time (`cpm`/`apm --install`)

Pass `--runtime-version <v>` to check the manifest's bounds against a target runtime version before installing:

```bash
cpm --install --id payment-links --runtime-version 1.11.0 --bundle-path /opt/payos/bundle
apm --install --app my-app --runtime-version 1.11.0 --bundle-path /opt/payos/bundle
```

Omitting `--runtime-version` skips the check entirely — this is opt-in, not a breaking change to existing scripted installs. When given and the manifest declares an incompatible bound, `--install` aborts **before any side-effect** (no file copy, no `registry.json`/`bootstrap.json` write) with a message like:

```
Capability 'payment-links' version 1.2.0 requires payos-runtime >=1.10.0, <=2.0.0, but the target runtime is 1.9.0-RELEASE
```

Add `--allow-incompatible-runtime` to downgrade that abort to a logged warning and proceed anyway — an explicit, one-shot escape hatch for a deliberate override (testing a newer package against an older sandbox runtime, for example), with no effect if `--runtime-version` wasn't given. Full flag reference: [cli-tools/cpm.md](../cli-tools/cpm.md), [cli-tools/apm.md](../cli-tools/apm.md).

## Enforcement at boot (`BootServer`)

On every startup, `ma.s2m.payos.applications.RuntimeCompatibilityAuditor.audit(config, runtimeVersion)` (`payos-kernel`) walks every entry in `bootstrap.json`'s `applications` array — this covers installed capabilities too, since both are written to the same array — and returns one message per entry whose declared bounds rule out the running runtime version (or whose bounds are themselves unparseable). `BootServer` then applies a policy read from the runtime configuration (`bootstrap.json`, or any file merged from `configDir`):

```json
{
  "runtimeCompatibility": {
    "warnOnly": true
  }
}
```

| `runtimeCompatibility.warnOnly` | Behaviour |
| --- | --- |
| `false` (default, absent = `false`) | Every issue is logged at `ERROR`, then boot aborts (`System.exit(1)`) — safe default for production. |
| `true` | Every issue is logged at `WARN`, boot continues — intended for dev/sandbox, mirrors `multitenancy.tenantSimulator`'s explicit off-by-default opt-in for other dev-only shortcuts. |

Config key constants: `IConfigSpec.RuntimeCompatibility` (`payos-foundation`). Policy-reading helper: `BootServer.isRuntimeCompatibilityWarnOnly(Map)` (package-private, unit-tested directly since `BootServer.main()` itself calls `System.exit()` and isn't testable end-to-end).

## Reporting: auditing what's installed before a runtime upgrade

`--status --compat` on `cpm`/`apm` prints the declared range for each installed application/capability, plus an `OK`/`INCOMPATIBLE` verdict when `--runtime-version` is also given — useful to check everything installed in a bundle before bumping the runtime version:

```bash
cpm --status --compat --runtime-version 2.0.0 --bundle-path /opt/payos/bundle
apm --status --app my-app --compat --runtime-version 2.0.0 --bundle-path /opt/payos/bundle
```

```
id:           payment-links
version:      1.2.0
status:       INSTALLED
compat-range: >=1.10.0, <=2.0.0
compat-check: OK (runtime=2.0.0)
```

## What's deliberately out of scope

Capability-to-capability dependency version constraints (the `dependencies[]` graph `cpm` already resolves at install time — "capability A needs capability B, optionally at some version") are a separate concern from runtime compatibility and are not touched by any of the above; that graph's own version resolution continues to work exactly as it did before this feature.

## File map

| Repo | File | What |
| --- | --- | --- |
| `payos-foundation` | `ma/s2m/payos/config/IConfigSpec.java` | `Applications.Application.MIN_RUNTIME_VERSION`/`MAX_RUNTIME_VERSION`; `RuntimeCompatibility` config block (`warnOnly`). |
| `payos-foundation` | `ma/s2m/payos/config/RuntimeCompatibility.java` | Shared version-range comparison (`isCompatible`, `describeRange`, `parse`). |
| `payos-kernel` (`payos`) | `ma/s2m/payos/config/PayOSConfig.java` | `getRuntimeVersion()`. |
| `payos-kernel` (`payos`) | `ma/s2m/payos/applications/RuntimeCompatibilityAuditor.java` | Boot-time audit over `bootstrap.json`'s `applications[]`. |
| `payos-kernel` (`payos`) | `ma/s2m/payos/BootServer.java` | Startup version log; audit call; hard-fail/warn-only policy. |
| `payos-runtime` | `pom.xml` | `maven-shade-plugin` manifest stamping (`Implementation-Version`). |
| `payos-pm` | `ma/s2m/payos/pm/RuntimeVersionReader.java` | Offline jar manifest read (no JVM). |
| `payos-pm` | `ma/s2m/payos/pm/cpm/CapabilityManifest.java`, `CapabilityManager.java`, `Cpm.java` | Schema fields; install-time check + `--allow-incompatible-runtime`; `--status --compat`; `--runtime-version` CLI option. |
| `payos-pm` | `ma/s2m/payos/pm/apm/ApplicationManifest.java`, `ApplicationManifestMerge.java`, `ApplicationManager.java`, `Apm.java` | Schema fields; shared manifest-merge helper; install-time check + `--allow-incompatible-runtime`; `--status --compat`; `--runtime-version` CLI option. |
| `payos-pm` | `ma/s2m/payos/pm/ppm/ProductManager.java` | Manifest-first merge for a product's constituent applications. |

## Next

- [cli-tools/cpm.md](../cli-tools/cpm.md) · [cli-tools/apm.md](../cli-tools/apm.md) · [cli-tools/ppm.md](../cli-tools/ppm.md)
- [developer/application-model.md](application-model.md)
- [configuration/bootstrap-reference.md](../configuration/bootstrap-reference.md)
