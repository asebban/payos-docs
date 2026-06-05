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
├── config/            configuration files (config.json, security.json, webhooks.json, …)
├── api/               API scripts (.js)            → resource type "api"
├── page/              pages (.vue, .html)          → resource type "page"
├── menu/              menu definitions             → resource type "menu"
├── lib/               shared JavaScript libraries  → resource type "lib"
└── i18n/              translation files            → resource type "i18n"
```

Resource types and directory names are defined by `ma.s2m.payos.resources.IResource`:

| Resource type | Directory | Extensions | Served at |
| --- | --- | --- | --- |
| `API_RESOURCE` | `api/` | `.js` | `/{appId}/api/...` |
| `PAGE_RESOURCE` | `page/` | `.vue`, `.html` | `/{appId}/page/...` |
| `COMPONENT_RESOURCE` | `page/` (components) | `.vue`, `.html` | `/{appId}/component/...` |
| `MENU_RESOURCE` | `menu/` | `.json` | `/{appId}/menu/...` |
| `LIB_RESOURCE` | `lib/` | `.js` | loaded via `$Library` |
| `I18N_RESOURCE` | `i18n/` | `.json` | loaded via `$I18n` |

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
| `category` | `"application"` or `"capability"`. |
| `extends` | String or array of parent app/capability IDs (resource inheritance). |
| `authorized-tenants` / `authorized.tenants` | Allowlist of tenants permitted to use the app. |
| `defaultUrl` | Post-login redirect URL. |
| `mapping-files` | Array of data-model mapping file paths (see [data access](data-access.md)). |
| `security` | Per-app security/OIDC overrides (see [security config](../configuration/security-oidc.md)). |
| `database-service` | Per-app database configuration (see [database config](../configuration/database-service.md)). |

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
| source type | `"local"` or `"git"`. |
| `baseUrl` | Git repository base URL (for `git`). |
| `path` | Local repository path (for `local`). |

The package managers ([`apm`](../cli-tools/apm.md), [`ppm`](../cli-tools/ppm.md)) can resolve
applications from this catalog.

## Next

- [Writing API scripts](writing-apis.md) — implement the `api/` resources.
- [Configuration reference](../configuration/README.md) — every application key in detail.
