# `pdoc` — Static OpenAPI generator

`pdoc` generates OpenAPI 3.1 specifications from `@payos.openapi` annotations in your API scripts. It is **static-only**: it parses annotations and never executes code, databases, queues, or webhooks — which is what makes it safe for regulated, auditable delivery. Module `pdoc` (main class `ma.s2m.payos.pdoc.PdocApplication`, version `1.0.0-RELEASE`). The authoring side (annotation format) is in [developer/api-documentation.md](../developer/api-documentation.md).

## Install

```bash
# from the pdoc module
./scripts/install-pdoc-tools.sh        # or install-pdoc-tools.ps1 on Windows
```

Installs the `pdoc` command (typically into `~/.payos/bin`).

## What it does

- Scans `api/**/*.js` for `@payos.openapi` annotation blocks.
- Assembles a complete OpenAPI 3.1 document.
- **Never** runs your scripts or touches runtime services (runtime-safety guarantee).

## Usage

```bash
pdoc <target> [options]
```

### Target selection

| Flag | Purpose |
| --- | --- |
| `--app <id>` | Generate docs for a single application. |
| `--capability <id>` | Generate docs for a capability. |
| `--product <id>` | Generate docs for a product (its applications). |

### Options

| Flag | Purpose |
| --- | --- |
| `--bundle-path <path>` | The bundle to read from. |
| `--output <path>` | Where to write the generated spec. |

> If `--output` is omitted, `pdoc` writes to
> `target/openapi/<applications|capabilities|products>/<targetId>/openapi.yaml` — the
> directory segment matches the target kind (`--app` → `applications`, `--capability` →
> `capabilities`, `--product` → `products`).

## Examples

```bash
# generate the OpenAPI spec for the payments app
pdoc --app payments --bundle-path /opt/payos/bundle --output ./openapi

# for a whole product
pdoc --product gateway --bundle-path /opt/payos/bundle --output ./openapi
```

## Serving the spec

Point the HTTP server's `swaggerUI.openapi-yaml` at the generated document; it is served at
`/openapi.yaml` with Swagger UI under `/swagger/**` (local-only by default). See
[configuration/servers.md](../configuration/servers.md).

## Why static generation matters

Because generation has **no side effects**, you can produce API documentation as part of a
controlled, audited delivery pipeline without provisioning databases, brokers, or webhook
endpoints. The `pdoc` module documents this safety model in depth (annotation-format,
generation-ownership, runtime-safety, regulated-delivery-auditability).

## Next

- [developer/api-documentation.md](../developer/api-documentation.md) — annotation format.
- [configuration/servers.md](../configuration/servers.md) — serving the spec.
