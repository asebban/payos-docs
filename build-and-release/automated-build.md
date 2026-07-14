# PayOS build step by step and automated build

This module provides an explanation of the build process and the role of the shared Maven parent and the bom (bill of material) for PayOS artifacts.

This document is automation-first: for each lifecycle scenario, it gives an exact step-by-step procedure and the corresponding script command.

## 1. Artifact Responsibilities

| Artifact | Responsibility |
|---|---|
| `payos-parent` | Shared Java/Maven conventions, plugin versions, third-party dependency versions. |
| `payos-core-bom` (`payos-bom`) | Owns `payos-kernel.version` and kernel dependency policy. |
| `payos-kernel` (`payos`) | Core runtime API/behavior contract consumed by internal modules. |
| Internal modules | Independent release units (`payos-server-http`, `payos-server-tcp`, `database-service`, etc.). |
| `payos-runtime` | Distribution assembler with explicit embedded module versions. |

## 2. Quick Decision Matrix

Use this matrix to decide what to bump.

| Scenario | Must bump | Usually unchanged |
|---|---|---|
| Internal module source change | Changed module version and runtime version | `payos-parent`, `payos-core-bom`, `payos-kernel` |
| Kernel API/contract change | `payos-kernel` version, `payos-core-bom` version and runtime version | `payos-parent` (unless parent itself changed) |
| Parent dependency/plugin policy change | `payos-parent` version and all child `<parent><version>` references, as well as the runtime version | `payos-kernel`, `payos-core-bom` |
| New module onboarding | New module version, parent alignment, core BOM alignment, optional runtime embed and runtime version| Existing module versions |

## 2.1 Version Bump Rules (Runtime, BOM, Parent)

Use this section as the authoritative rule for when each top-level version must change.

| Artifact | Must change when | Usually does not need to change when |
|---|---|---|
| `payos-runtime` | Embedded module set/version changes (add/remove/update runtime dependencies), runtime startup/wiring/packaging behavior changes, runtime assembly/config contract changes, payos-kernel change, add new embedded module | Only source changes in a module that is not embedded yet, parent-only policy updates with no runtime packaging impact |
| `payos-core-bom` (`payos-bom`) | `payos-kernel.version` changes, dependency policy exposed to consumers changes (managed versions/imported contracts), BOM import policy updates consumed by modules | Parent plugin/tooling-only changes, runtime-only assembly changes |
| `payos-parent` | Shared build/dependency/plugin governance changes that affect children (plugin versions, compiler/test/toolchain conventions, shared parent-level dependency management) | Module business logic changes, kernel API changes without parent policy changes |

Practical interpretation:

1. If kernel changes, bump BOM with it.
2. If runtime composition or payos BOM or modules versions change, bump runtime in the same release.
3. If parent governance or dependeny version changes, bump parent and propagate `<parent><version>` in all children (without changing their respective versions).
4. If two or more categories change together, use `release` mode and bump each impacted artifact in one command.

## 3. Automation Scripts

Scripts are located in `payos-parent/scripts`.

- PowerShell: `update-payos-version.ps1`
- Bash: `update-payos-version.sh`

Supported modes:

- `kernel` (when kernel version changes)
- `module` (when a payos module version changes)
- `parent` (when third party dependencies change)
- `new-module` (when a new payos module is added)
- `release` (combined orchestration)

Global options:

- `--skip-build`: update POM files only
- `--skip-tests`: pass `-DskipTests` to Maven build steps

## 4. Step-by-Step: Internal Module Changes

Use this when a module implementation changes and you need a new module release.

### 4.1 What the script updates

1. `<module>/pom.xml` project version
2. Matching dependency version in `payos-runtime/pom.xml` (if present)
3. `payos-runtime/pom.xml` project version (if `--runtime-version` is provided)

### 4.2 PowerShell

```powershell
.\scripts\update-payos-version.ps1 module `
  --module payos-server-http `
  --module-version 1.1.1-RELEASE `
  --runtime-version 1.2.8-RELEASE
```

### 4.3 Bash

```bash
./scripts/update-payos-version.sh module \
  --module payos-server-http \
  --module-version 1.1.1-RELEASE \
  --runtime-version 1.2.8-RELEASE
```

## 5. Step-by-Step: Kernel Changes

Use this when kernel contract/API changes and consumers must adopt a new kernel policy.

### 5.1 What the script updates

1. `payos/pom.xml` project version (`payos-kernel`)
2. `payos-bom/pom.xml` project version (`payos-core-bom`)
3. `payos-bom/pom.xml` property `payos-kernel.version`
4. All internal module imports of `ma.s2m.payos:payos-core-bom`
5. Optional `payos-runtime/pom.xml` project version (`--runtime-version`)

### 5.2 PowerShell

```powershell
.\scripts\update-payos-version.ps1 kernel `
  --kernel-version 1.2.6-RELEASE `
  --core-bom-version 1.0.2-RELEASE `
  --runtime-version 1.2.8-RELEASE
```

### 5.3 Bash

```bash
./scripts/update-payos-version.sh kernel \
  --kernel-version 1.2.6-RELEASE \
  --core-bom-version 1.0.2-RELEASE \
  --runtime-version 1.2.8-RELEASE
```

## 6. Step-by-Step: Parent Changes

Use this when shared Java/plugins/dependency policy changes in `payos-parent`.

### 6.1 What the script updates

1. `payos-parent/pom.xml` project version
2. Every child POM that inherits `ma.s2m.payos:payos-parent` (`<parent><version>...`)

### 6.2 PowerShell

```powershell
.\scripts\update-payos-version.ps1 parent `
  --parent-version 1.0.1-RELEASE
```

### 6.3 Bash

```bash
./scripts/update-payos-version.sh parent \
  --parent-version 1.0.1-RELEASE
```

## 7. Step-by-Step: New Module Onboarding

Use this when a new module is added and must be aligned with PayOS versioning rules.

### 7.1 What the script updates

1. `<module>/pom.xml` parent version aligned to current `payos-parent`
2. `<module>/pom.xml` `payos-core-bom` import aligned to current core BOM
3. Inserts `dependencyManagement` + core BOM import if missing
4. `<module>/pom.xml` module project version
5. Inserts/updates module dependency in `payos-runtime/pom.xml` (unless `--no-runtime-embed`)
6. Optional `payos-runtime/pom.xml` project version (`--runtime-version`)

### 7.2 PowerShell

```powershell
.\scripts\update-payos-version.ps1 new-module `
  --module secret-service-filesystem `
  --module-version 1.0.0-RELEASE `
  --runtime-version 1.2.8-RELEASE
```

If the module should NOT be embedded in runtime yet:

```powershell
.\scripts\update-payos-version.ps1 new-module `
  --module secret-service-filesystem `
  --module-version 1.0.0-RELEASE `
  --no-runtime-embed
```

### 7.3 Bash

```bash
./scripts/update-payos-version.sh new-module \
  --module secret-service-filesystem \
  --module-version 1.0.0-RELEASE \
  --runtime-version 1.2.8-RELEASE
```

Without runtime embed:

```bash
./scripts/update-payos-version.sh new-module \
  --module secret-service-filesystem \
  --module-version 1.0.0-RELEASE \
  --no-runtime-embed
```

## 8. Build Sequences Executed by Scripts

### 8.1 `kernel` mode

1. `payos-parent` install
2. `payos-core-bom` install
3. API dependency modules install: `payos-secret-api`, `payos-notification-api`, `payos-connector-sdk`
4. `payos-kernel` install
5. Internal modules install (`payos-server-http`, `payos-server-tcp`, `payos-server-queue`,
   `database-service`, `queue-service-nats`, `webhook-service-http`, `payos-pm`,
   `payos-service-notification`, `secret-service-filesystem`, `secret-service-vault`,
   `payos-notification-connector`)
6. `payos-runtime` package

### 8.2 `module` and `new-module` modes

1. `payos-parent` install
2. `payos-core-bom` install
3. API dependency modules install: `payos-secret-api`, `payos-notification-api`, `payos-connector-sdk`
4. `payos-kernel` install
5. Changed module install
6. `payos-runtime` package

### 8.3 `parent` mode

1. `payos-parent` install
2. `payos-core-bom` install
3. API dependency modules install: `payos-secret-api`, `payos-notification-api`, `payos-connector-sdk`
4. `payos-kernel` install
5. Internal modules install (`payos-server-http`, `payos-server-tcp`, `payos-server-queue`,
   `database-service`, `queue-service-nats`, `webhook-service-http`, `payos-pm`,
   `payos-service-notification`, `secret-service-filesystem`, `secret-service-vault`,
   `payos-notification-connector`)
6. `payos-runtime` package

### 8.4 `release` mode

1. Apply parent changes (if `--parent-version` provided)
2. Apply kernel/BOM changes (if `--kernel-version` + `--core-bom-version` provided)
3. Apply existing module updates from `--module-updates`
4. Apply new module onboarding from `--new-module-updates`
5. Apply runtime version update (if `--runtime-version` provided)
6. Run one consolidated build sequence:
   - `payos-parent` install
   - `payos-core-bom` install
   - API dependency modules install: `payos-secret-api`, `payos-notification-api`, `payos-connector-sdk`
   - `payos-kernel` install
   - standard internal modules install (including `payos-service-notification` and
     `payos-notification-connector`)
   - additionally changed/new modules install
   - `payos-runtime` package

## 9. Combined Scenario: Everything Changed

When parent policy, kernel, existing modules, and new modules all change in the same release, use `release` mode.

### 9.1 PowerShell

```powershell
.\scripts\update-payos-version.ps1 release `
  --parent-version 1.0.1-RELEASE `
  --kernel-version 1.2.6-RELEASE `
  --core-bom-version 1.0.2-RELEASE `
  --module-updates "payos-server-http=1.1.1-RELEASE,payos-server-tcp=1.0.7-RELEASE" `
  --new-module-updates "secret-service-filesystem=1.0.0-RELEASE" `
  --runtime-version 1.2.9-RELEASE
```

### 9.2 Bash

```bash
./scripts/update-payos-version.sh release \
  --parent-version 1.0.1-RELEASE \
  --kernel-version 1.2.6-RELEASE \
  --core-bom-version 1.0.2-RELEASE \
  --module-updates "payos-server-http=1.1.1-RELEASE,payos-server-tcp=1.0.7-RELEASE" \
  --new-module-updates "secret-service-filesystem=1.0.0-RELEASE" \
  --runtime-version 1.2.9-RELEASE
```

Notes:

1. `--module-updates` and `--new-module-updates` accept comma-separated `DIR=VERSION` entries.
2. Add `--no-runtime-embed` if new modules should be onboarded but not yet embedded in runtime.
3. Add `--skip-build` to only update POM files.

## 10. Safe Dry-Run Style

To update all POMs first, without building:

```powershell
.\scripts\update-payos-version.ps1 <mode> ... --skip-build
```

Then review and build manually.

## 11. Practical Rule of Thumb

1. Module code changed: use `module`
2. Kernel changed: use `kernel`
3. Shared parent policy changed: use `parent`
4. New module onboarding/alignment: use `new-module`
5. Multiple categories changed in one release: use `release`

If runtime composition changes, pass `--runtime-version` so runtime release reflects embedded set changes.
