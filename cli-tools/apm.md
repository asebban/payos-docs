# `apm` — Application Package Manager

`apm` installs, uninstalls, and inspects **applications** in a bundle. It is part of the `payos-pm` module (picocli; main class `ma.s2m.payos.pm.apm.Apm`, version `1.1.2-RELEASE`). For the application model, see [developer/application-model.md](../developer/application-model.md); for capabilities and products use [`cpm`](cpm.md) and [`ppm`](ppm.md).

## Install the tool

```bash
# from the payos-pm module
./install-payos-tools.sh        # or install-payos-tools.ps1 on Windows
```

This installs `apm`, `cpm`, and `ppm` (typically into `~/.payos/bin`).

## Usage

```bash
apm [global-options] <action> [options]
```

### Actions

| Action | Purpose |
| --- | --- |
| `--install` | Install an application into the bundle. |
| `--uninstall` | Remove an application from the bundle. |
| `--status` | Show installed applications / a specific app's status. |

### Options

| Option | Purpose |
| --- | --- |
| `--app <id[@version] \| path>` | The application to act on — by id (optionally `@version`) or a path. |
| `--base-path <path>` | Override the application's base path. |

### Global options (shared by `apm`/`cpm`/`ppm`)

| Option | Default | Purpose |
| --- | --- | --- |
| `--bundle-path <path>` | `.` | The target bundle (directory or `payos.json`). |
| `-h`, `--help` | | Show help. |
| `-V`, `--version` | | Show version. |

## Examples

```bash
# install an application by id@version into a bundle
apm --install --app payments@1.0.0 --bundle-path /opt/payos/bundle

# install from a local path
apm --install --app ./build/payments --bundle-path /opt/payos/bundle

# show what is installed
apm --status --bundle-path /opt/payos/bundle

# uninstall
apm --uninstall --app payments --bundle-path /opt/payos/bundle
```

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | Success. |
| `1` | Error during the operation. |
| `2` | Usage error. |

## How it relates to the runtime

`apm` writes into the bundle's `applications` registry (and may resolve from the
[application catalog](../configuration/bootstrap-reference.md#applicationcatalog-optional)).
The runtime picks up registry changes via [hot reload](../operations/hot-reload.md).

## Next

- [cpm.md](cpm.md) — capabilities.
- [ppm.md](ppm.md) — products.
- [developer/application-model.md](../developer/application-model.md)
