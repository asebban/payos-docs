# `cpm` — Capability Package Manager

`cpm` installs, uninstalls, activates, and deactivates **capabilities** — reusable extensions
that applications consume through `extends`. It is part of the `payos-pm` module (picocli;
main class `ma.s2m.payos.pm.cpm.Cpm`). Capabilities differ from applications in that they can
be installed and activated/deactivated independently, per app or per tenant. See
[architecture/extensibility.md](../architecture/extensibility.md) and
[developer/application-model.md](../developer/application-model.md).

## Usage

```bash
cpm [global-options] <action> [options]
```

### Actions

| Action | Purpose |
| --- | --- |
| `--install` | Install a capability into the bundle. |
| `--uninstall` | Remove a capability (or all installed capabilities with `--all`). |
| `--activate` | Activate a capability (optionally scoped to an app/tenant). |
| `--deactivate` | Deactivate a capability. |
| `--status` | Show installation/activation status of a capability, or all of them without `--id`. |

### Options

| Option | Purpose |
| --- | --- |
| `--id <id>` | Capability id. Required for every action except `--status` (omit to show all) and `--uninstall --all`. |
| `--version <version>` | Capability version (install only; resolves latest if omitted). |
| `--from-git[=<baseUrl>]` | Use a git catalog for this install, optionally overriding its configured `baseUrl`. |
| `--from-local[=<dir>]` | Use a local catalog for this install, optionally overriding its configured `path`. `<dir>` is the **root** of the catalog — it must contain one subdirectory per capability id (`<dir>/<id>/manifest.json`, optionally versioned as `<dir>/<id>/<version>/manifest.json`) — not the directory of the single capability being installed. |
| `--all` | Uninstall every installed capability instead of a single `--id` (`--uninstall` only; mutually exclusive with `--id`). Or when `--status` is specified shows the status of all capabilities — omit `--id` there to see all. |
| `--app <appId>` | Scope activation/deactivation to an application. |
| `--tenant <tenantId>` | Scope activation/deactivation to a tenant. |
| `--cascade` | Cascade the operation (e.g. dependent capabilities). |
| `--drop-schema` | On uninstall, drop the capability's database schema. |
| `--runtime-version <version>` | Target payos-runtime version to check the manifest's `minRuntimeVersion`/`maxRuntimeVersion` bounds against (`--install` only). Omitted: no compatibility check. Given and incompatible: `--install` aborts before any side-effect. See [runtime compatibility checks](../developer/runtime-compatibility-checks-v1-2026-08-06.md). |
| `--allow-incompatible-runtime` | Downgrades an incompatible `--runtime-version` check from an install-aborting error to a warning, and proceeds anyway. No effect without `--runtime-version`. |
| `--compat` | With `--status`: also prints `compat-range` (declared bounds) and, when `--runtime-version` is given, a `compat-check: OK`/`INCOMPATIBLE` verdict. |

> `--install` resolves the capability through the configured **catalog** (`capabilityCatalog`
> in `payos.json` or `*.json` found in `configDir`) — there is no longer a plain `--path` flag for installing from an arbitrary (replaced by `--from-local`)
> local directory outside the catalog. `--from-git`/`--from-local` are mutually exclusive
> one-off overrides of that configured catalog's type/location; when neither is given, `cpm` uses `capabilityCatalog` as configured.

### Global options

| Option | Default | Purpose |
| --- | --- | --- |
| `--bundle-path <path>` | `.` | The target bundle. |
| `-h`, `--help` | | Show help. |
| `-v` | | Show version. Unlike `apm`/`ppm`, `cpm` defines only the bare `-v` short flag for this — there is no `--version` long form. |

## State files

Capability state is kept under `{configDir}/.capabilities/`:

| File | Purpose |
| --- | --- |
| `registry.json` | Installed capabilities. |
| `activation.json` | Active scopes (app/tenant). |
| `events.ndjson` | Append-only event log of capability operations. |

The runtime consults `IActivationStore.isActive(...)` during resource resolution, so a
capability's resources are only visible while it is active. See
[developer/application-model.md](../developer/application-model.md).

## Examples

```bash
# install a capability, overriding the configured catalog for this run only
# (./build/capabilities is the catalog ROOT: it must contain ./build/capabilities/audit-capability/manifest.json)
cpm --install --id audit-capability --from-local=./build/capabilities --bundle-path /opt/payos/bundle

# install from the configured capabilityCatalog as-is
cpm --install --id audit-capability --bundle-path /opt/payos/bundle

# activate it for one application and tenant
cpm --activate --id audit-capability --app payments --tenant acme \
    --bundle-path /opt/payos/bundle

# deactivate
cpm --deactivate --id audit-capability --app payments --tenant acme \
    --bundle-path /opt/payos/bundle

# show status of one capability, or all of them (omit --id to show all)
cpm --status --id audit-capability --bundle-path /opt/payos/bundle
cpm --status --bundle-path /opt/payos/bundle

# install, checking compatibility with a target runtime version first
cpm --install --id audit-capability --runtime-version 1.11.0 --bundle-path /opt/payos/bundle

# audit every installed capability against a runtime version before upgrading it
cpm --status --compat --runtime-version 2.0.0 --bundle-path /opt/payos/bundle

# uninstall and drop its schema
cpm --uninstall --id audit-capability --drop-schema --bundle-path /opt/payos/bundle

# uninstall every installed capability
cpm --uninstall --all --bundle-path /opt/payos/bundle
```

## Exit codes

`0` success · `1` error · `2` usage error.

## Next

- [apm.md](apm.md) · [ppm.md](ppm.md)
- [architecture/extensibility.md](../architecture/extensibility.md)
- [developer/runtime-compatibility-checks-v1-2026-08-06.md](../developer/runtime-compatibility-checks-v1-2026-08-06.md) — `--runtime-version`/`--allow-incompatible-runtime`/`--compat` in full context.
