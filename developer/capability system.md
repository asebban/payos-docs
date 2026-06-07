# Capability System

Capabilities are self-contained functional packages that extend applications at runtime without redeployment. They are managed by the external `cpm` tool and stored under `{configDir}/.capabilities/`.

## Capability as an Application

A capability is registered in `bootstrap.json` like any other application entry, with one additional field:

```json
{
  "id": "payment-links",
  "name": "Payment Links",
  "base.path": "/path/to/.capabilities/payment-links",
  "version": "1.0.0",
  "category": "capability"
}
```

The `category: capability` marker distinguishes it from regular applications so that auto-activation logic can target only non-capability apps.

## Activation via `extends`

Activating a capability adds its `id` to the `extends` array of the target application(s):

```json
{
  "id": "my-app",
  "base.path": "/opt/payos/apps/my-app",
  "extends": ["payment-links", "notifications"]
}
```

The runtime resolves the `extends` list at startup and merges the capability's resources, scripts, and config into the application scope.

## State Files (under `{configDir}/.capabilities/`)

| File | Description |
|------|-------------|
| `registry.json` | Installation records: `id`, `version`, `installedAt`, `status` (`INSTALLED` / `FAILED`). Contains the installed capabilities in the current bundle |
| `activation.json` | Scoped activation entries: `capabilityId`, `applicationId` (null = all), `tenants` (null = all), `active` (false = negative override). Contains the activation state and scope of installed capabilities (`activated` / `deactivated`) |
| `events.ndjson` | Append-only audit log of lifecycle events (`INSTALL`, `ACTIVATE`, `DEACTIVATE`, `UNINSTALL`) |
| `.capabilities/<id>/` | The capability package directory (scripts, config, manifest.json). Capabilities are usually installed in `${configDir}/.capabilities` folder |

## Capability Lifecycle Hooks

The lifecycle hooks are another differentiotor aspect that differentiate applications from capabilities. Each capability may provide `hooks/lifecycle.js` exporting up to four functions:

```js
exports.install   = function(ctx) { /* ... */ };
exports.activate  = function(ctx) { /* ... */ };
exports.deactivate = function(ctx) { /* ... */ };
exports.uninstall = function(ctx) { /* ... */ };
```

The `ctx` object provides: `capabilityId`, `version`, `configDir`, and optionally `appId`, `tenantId`, `dropSchema`. Hooks run in a GraalVM JS sandbox with `allowAllAccess=false`. A hook failure on `install` marks the capability `FAILED` and skips auto-activation.
