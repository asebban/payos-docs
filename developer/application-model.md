# Application model

An **application** is a directory of resources deployed *into* the runtime and registered in
the configuration. This page describes the directory layout, the resource types, the
application configuration, and the relationships between applications, capabilities, and
products.

## Directory layout

An application's root is given by its `base.path` (default `.apps/{id}`, resolved relative to
`configDir`):

```
apps/{appId}/
├── manifest.json      application manifest consumed by apm to register the app (id, name, version, base.path, …)
├── webhooks.json      outbound webhook subscriptions, at the application root (see architecture/hooks-webhooks.md)
├── config/            configuration files — every *.json file here is merged into the app's effective config; conventional names are application.json, mappings.json, routes.json, i18n.json, hooks.json (hook lock manifest, see below), but the filenames themselves are arbitrary
├── api/               API scripts (.js)                        → resource type "api"
├── page/              pages (.vue, .html)                      → resource type "page"
├── component/         reusable Vue components (.vue, .html)    → resource type "component"
├── menu/              menu definitions                         → resource type "menu"
├── lib/               shared JavaScript libraries (.js)        → resource type "lib"
├── i18n/              translation files (.json), one subdirectory per locale → resource type "i18n"
├── files/             static files served as-is (images, CSS, PDF, fonts, …) → resource type "file"
└── hooks/             lifecycle hook scripts (.js), see "Hooks" below — not a generic resource type
```

Resource types and directory names are defined by `ma.s2m.payos.resources.IResource`:

| Resource type | Directory | Extensions | Served at |
| --- | --- | --- | --- |
| `API_RESOURCE` | `api/` | `.js` | `/{appId}/api/...` |
| `PAGE_RESOURCE` | `page/` | `.vue`, `.html` | `/{appId}/page/...` |
| `COMPONENT_RESOURCE` | `component/` | `.vue`, `.html` | `/{appId}/component/...` |
| `MENU_RESOURCE` | `menu/` | `.json` | `/{appId}/menu` |
| `LIB_RESOURCE` | `lib/` | `.js` | loaded via `$Library` |
| `I18N_RESOURCE` | `i18n/` | `.json` | loaded via `$I18n` |
| `FILE_RESOURCE` | `files/` | any | `/{appId}/files/...` — static assets, served with MIME detection, ETag/Last-Modified/304 caching, and directory-traversal protection |

### Hooks

The `hooks/` directory holds lifecycle scripts run by `HookEngine` around API and page requests — `pre-request.js`, `post-request.js`, `on-error.js`, `page-pre-serve.js`, `page-post-serve.js`, `page-on-error.js`. Hooks are resolved along the same `extends` chain as other resources but are **not** a generic `IResource` type (no `HOOK_RESOURCE` constant, no HTTP-served path); the presence of the file is enough to register it, no mapping entry is required. A parent app can prevent a child from short-circuiting one of its hooks by declaring it in `config/hooks.json`:

```json
{
  "pre-request.js": { "locked": true }
}
```

A locked hook raises a pipeline error if a child hook calls `$HookChain.stop()` instead of `$HookChain.proceed()`. Full reference: [architecture/hooks-webhooks.md](../architecture/hooks-webhooks.md) and the step-by-step walkthrough in [create-application-guide.md](./create-application-guide.md#75-hooks-et-webhooks).

## Registering an application

An application is registered in the `applications` array of the effective runtime
configuration (typically `config/bootstrap.json` under `configDir`), or installed with
[`apm`](../cli-tools/apm.md). The model class is `ma.s2m.payos.applications.Application`.

```json
{
  "applications": [
    {
      "id": "payments",
      "name": "Payments",
      "base.path": "../apps/payments",
      "version": "1.0.0",
      "category": "application",
      "extends": ["audit-capability"],
      "authorized-tenants": ["acme", "globex"],
      "defaultUrl": "/payments/page/dashboard"
    }
  ]
}
```

### Application configuration keys

From `IConfigSpec.Applications.Application` (full reference in
[configuration/bootstrap-reference.md](../configuration/bootstrap-reference.md)):

| Key | Purpose |
| --- | --- |
| `id` (required) | Unique application identifier; the first URI path segment. |
| `name` | Display name. |
| `base.path` | Filesystem path to the application directory (default `.apps/{id}`), resolved relative to `configDir`. |
| `version` | Semantic version. |
| `minRuntimeVersion` | Lower bound (inclusive) of the compatible payos-runtime version. Optional, no constraint when absent. Checked by `apm --install --runtime-version <v>` (payos-pm; `--allow-incompatible-runtime` downgrades a mismatch to a warning), and re-checked at every `BootServer` startup — see [runtime compatibility](#runtime-compatibility-policy) below. |
| `maxRuntimeVersion` | Upper bound (inclusive) of the compatible payos-runtime version. Same checks as `minRuntimeVersion` above. |
| `category` | `"application"` or `"capability"`. |
| `extends` | String or array of parent app/capability IDs (resource inheritance). |
| `authorized-tenants` / `authorized.tenants` | Allowlist of tenants permitted to use the app. |
| `defaultUrl` | Post-login redirect URL. |
| `mapping-files` | Array of data-model mapping file paths (see [data access](data-access.md)). |
| `security` | Per-app security/OIDC overrides (see [security config](../configuration/security-oidc.md)). |
| `database-service` | Per-app database configuration (see [database config](../configuration/database-service.md)). |

The keys could equally be declared in separate `.json` files whose names are arbitrary but these files should all reside in the root of the base directory of the runtime ("configDir").

## Runtime compatibility policy

`BootServer` audits every entry in `bootstrap.json`'s `applications` array at startup (this covers installed capabilities too — both are written to the same array) and compares any declared `minRuntimeVersion`/`maxRuntimeVersion` against the running payos-runtime version. By default an incompatibility aborts boot (`System.exit(1)`, after logging every issue found). Set `runtimeCompatibility.warnOnly: true` in the runtime configuration (`bootstrap.json`, or any file merged from `configDir`) to only log a warning and keep starting instead — intended for dev/sandbox use, not production:

```json
{
  "runtimeCompatibility": {
    "warnOnly": true
  }
}
```

This is a defense-in-depth check on top of the install-time one (`apm`/`cpm --install --runtime-version <v>`): it also catches a capability dropped into the bundle by hand, or a runtime upgraded after the fact without revalidating what's already installed. Full reference, including the manifest schema and the `--compat` reporting flag: [developer/runtime-compatibility-checks-v1-2026-08-06.md](runtime-compatibility-checks-v1-2026-08-06.md).

## How resources are resolved: the `extends` chain

When a request targets a resource, `ResourceLocator` walks the application's `extends` chain
to find it. This is **resource inheritance**: an application can build on parent
applications and capabilities without copying their resources.

```
request → app "payments"  (extends ["audit-capability"])
            ├─ look for the resource in "payments"
            └─ then in "audit-capability"  (only if the capability is ACTIVE)
```

For any extended app whose `category` is `"capability"`, the locator consults
`IActivationStore.isActive(...)` and skips it when inactive. See
[architecture/extensibility.md](../architecture/extensibility.md).

## Applications vs. capabilities vs. products

| Unit | `category` | Deployed with | Purpose |
| --- | --- | --- | --- |
| **Application** | `application` | [`apm`](../cli-tools/apm.md) | A standalone unit of functionality. |
| **Capability** | `capability` | [`cpm`](../cli-tools/cpm.md) | A reusable extension other apps `extends`; installable/activatable without a runtime redeploy. |
| **Product** | n/a (a bundle) | [`ppm`](../cli-tools/ppm.md) | A set of applications plus shared server configuration. |

Definitions are in the [glossary](../overview/glossary.md); the architectural treatment of
capabilities is in [architecture/extensibility.md](../architecture/extensibility.md).

## Application catalog

Applications can be loaded from a **catalog** instead of a local path, configured in
`payos.json` under `applicationCatalog`:

| Field | Purpose |
| --- | --- |
| `type` | `"local"` or `"git"`. |
| `baseUrl` | Git repository base URL (for `git`). |
| `path` | Local repository path (for `local`). |

The package managers ([`apm`](../cli-tools/apm.md), [`ppm`](../cli-tools/ppm.md)) can resolve applications from this catalog. The `applicationCatalog` key can be declared in the main `payos.json` file of the runtime.

## Creating an appilcation from sratch

Here is the guide for creating a complete application from ground zero [Creating an application guide](./create-application-guide.md)

# Capability model

Capabilities are technically a particular form of an applications with minimal differences. A complete description of how to use capabilities can be found in [Capability System](./capability%20system.md) 

## Next

- [Writing API scripts](writing-apis.md) — implement the `api/` resources.
- [Configuration reference](../configuration/README.md) — every application key in detail.
