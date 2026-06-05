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
| `--uninstall` | Remove a capability. |
| `--activate` | Activate a capability (optionally scoped to an app/tenant). |
| `--deactivate` | Deactivate a capability. |

### Options

| Option | Purpose |
| --- | --- |
| `--id <id>` | Capability id. |
| `--version <version>` | Capability version. |
| `--path <path>` | Install from a local path. |
| `--from-catalog` | Resolve the capability from the configured catalog. |
| `--app <appId>` | Scope activation/deactivation to an application. |
| `--tenant <tenantId>` | Scope activation/deactivation to a tenant. |
| `--cascade` | Cascade the operation (e.g. dependent capabilities). |
| `--drop-schema` | On uninstall, drop the capability's database schema. |

### Global options

| Option | Default | Purpose |
| --- | --- | --- |
| `--bundle-path <path>` | `.` | The target bundle. |
| `-h`, `--help` | | Show help. |
| `-V`, `--version` | | Show version. |

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
# install a capability from a path
cpm --install --path ./build/audit-capability --bundle-path /opt/payos/bundle

# activate it for one application and tenant
cpm --activate --id audit-capability --app payments --tenant acme \
    --bundle-path /opt/payos/bundle

# deactivate
cpm --deactivate --id audit-capability --app payments --tenant acme \
    --bundle-path /opt/payos/bundle

# uninstall and drop its schema
cpm --uninstall --id audit-capability --drop-schema --bundle-path /opt/payos/bundle
```

## Exit codes

`0` success · `1` error · `2` usage error.

## Next

- [apm.md](apm.md) · [ppm.md](ppm.md)
- [architecture/extensibility.md](../architecture/extensibility.md)
