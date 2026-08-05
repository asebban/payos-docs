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
| `--all` | Apply the action to all products (`--status` or `--uninstall`; mutually exclusive with `--product` for `--uninstall`). |
| `--from-git[=<baseUrl>]` | Use a git product catalog for this install, optionally overriding its configured `baseUrl`. |
| `--from-local[=<dir>]` | Use a local product catalog for this install, optionally overriding its configured `path`. `<dir>` is the **root** of the catalog — it must contain one subdirectory per product id (`<dir>/<id>/manifest.json`, see [layouts below](#2-local-directory-catalog-productcatalogtype-local)) — not the directory of the single product being installed. |

> `--from-git`/`--from-local` only override the **product** catalog (`productCatalog`). The
> secondary `applicationCatalog` — used to pull a product's constituent applications when they
> aren't already present — stays config-driven and isn't affected by these flags.

### Global options

| Option | Default | Purpose |
| --- | --- | --- |
| `--bundle-path <path>` | `.` | The target bundle. |
| `-h`, `--help` | | Show help. |
| `-v`, `--version` | | Show version. |

## Examples

```bash
# install a product (its applications + shared server config)
ppm --install --product gateway --bundle-path /opt/payos/bundle

# install, overriding the configured productCatalog for this run only
# (./build/products is the catalog ROOT: it must contain ./build/products/gateway/manifest.json)
ppm --install --product gateway --from-local=./build/products --bundle-path /opt/payos/bundle

# show installed products
ppm --status --all --bundle-path /opt/payos/bundle

# uninstall
ppm --uninstall --product gateway --bundle-path /opt/payos/bundle
```

## Product manifest file naming

`ppm` resolves a product manifest through two distinct, non-interchangeable conventions depending on where it is found. **They use different filenames — do not assume `manifest.json` works everywhere.**

### 1. Local override (bundle working directory)

Checked first, before the catalog. Only the flat filename `<productId>.json` directly under the bundle's working directory is recognized:

```
<workingDir>/<productId>.json
```

`manifest.json` is **not** supported in this location — `ProductManager.resolveManifest` only looks for `<workingDir>/<productId>.json`.

Example: for product id `gateway` with working directory `/opt/payos/bundle`:

```
/opt/payos/bundle/gateway.json
```

### 2. Local-directory catalog (`productCatalog.type: "local"`)

When no local override exists, `ppm` falls back to the configured `productCatalog`. For a
`local`-type catalog, `LocalProductCatalogClient` supports **both** of the following layouts, tried in this order:

- **Preferred** — per-product directory: `<catalogRoot>/<productId>/manifest.json`
- **Fallback** — flat legacy file: `<catalogRoot>/<productId>.json`

When a `--product <id>@<version>` is given, the versioned equivalents are tried:

- **Preferred** — `<catalogRoot>/<productId>/<version>/manifest.json`
- **Fallback** — `<catalogRoot>/<productId>/<version>.json`

Example catalog layout (preferred, per-product-directory):

```
catalog/
  gateway/
    manifest.json
    1.2.0/
      manifest.json
```

Example catalog layout (flat legacy fallback):

```
catalog/
  gateway.json
  gateway/
    1.2.0.json
```

## Per-application `manifest.json` (constituent applications)

A product manifest's `applications[]` entries (`ApplicationRef`) only declare `id`, `basePath`,
and an optional `version` — enough to locate and pull an application, but not the richer fields
a standalone `apm install` picks up from that application's own `manifest.json` (`name`,
`category`, `extends`, `authorizedTenants`, `mappingFiles`, `security`,
`minRuntimeVersion`/`maxRuntimeVersion`).

To keep an application's `bootstrap.json` entry the same regardless of whether it was installed
via `apm install` directly or pulled in as part of a product, `ppm install` also checks for
`<appDir>/manifest.json` after the application directory is present (whether it was already
there or just pulled from the `applicationCatalog`):

- **Present** — its optional fields are merged into the `bootstrap.json` entry, on top of the
  `id`/`basePath`/`version` the product manifest's `ApplicationRef` already resolved (those three
  stay authoritative from the product side — a per-application `manifest.json`'s own `id`/
  `version` fields, if present, are ignored here).
- **Absent** — no-op; the entry is built solely from the `ApplicationRef` as before.

This means an application repo that already ships a `manifest.json` for standalone `apm install`
use doesn't need a separate variant to also declare, say, `minRuntimeVersion` when installed as
part of a product.

## Exit codes

`0` success · `1` error · `2` usage error.

## Relationship to `apm`/`cpm`

A product is the coarsest unit: installing a product typically pulls in the
[applications](apm.md) it comprises. Capabilities those applications `extends` are managed
with [`cpm`](cpm.md).

## Next

- [apm.md](apm.md) · [cpm.md](cpm.md)
- [developer/application-model.md](../developer/application-model.md)
- [developer/runtime-compatibility-checks-v1-2026-08-06.md](../developer/runtime-compatibility-checks-v1-2026-08-06.md) — how the per-application `manifest.json` merge above fits into the wider runtime-version compatibility feature.
