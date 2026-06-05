# `ppm` — Product Package Manager

`ppm` installs, uninstalls, and inspects **products** — bundles that group a set of
applications together with shared server configuration. It is part of the `payos-pm` module
(picocli; main class `ma.s2m.payos.pm.ppm.Ppm`). For the unit definitions, see
[developer/application-model.md](../developer/application-model.md).

## Usage

```bash
ppm [global-options] <action> [options]
```

### Actions

| Action | Purpose |
| --- | --- |
| `--install` | Install a product into the bundle. |
| `--uninstall` | Remove a product. |
| `--status` | Show installed products / a product's status. |

### Options

| Option | Purpose |
| --- | --- |
| `--product <id>` | The product to act on. |
| `--all` | Apply the action to all products. |

### Global options

| Option | Default | Purpose |
| --- | --- | --- |
| `--bundle-path <path>` | `.` | The target bundle. |
| `-h`, `--help` | | Show help. |
| `-V`, `--version` | | Show version. |

## Examples

```bash
# install a product (its applications + shared server config)
ppm --install --product gateway --bundle-path /opt/payos/bundle

# show installed products
ppm --status --all --bundle-path /opt/payos/bundle

# uninstall
ppm --uninstall --product gateway --bundle-path /opt/payos/bundle
```

## Exit codes

`0` success · `1` error · `2` usage error.

## Relationship to `apm`/`cpm`

A product is the coarsest unit: installing a product typically pulls in the
[applications](apm.md) it comprises. Capabilities those applications `extends` are managed
with [`cpm`](cpm.md).

## Next

- [apm.md](apm.md) · [cpm.md](cpm.md)
- [developer/application-model.md](../developer/application-model.md)
