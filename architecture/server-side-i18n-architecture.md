# Server-Side i18n Architecture

Last Alignment: 2026-05-06

This document describes the PayOS server-side localization service exposed to backend JavaScript through `$I18n`.

## Design Goals

The i18n service follows the PayOS resource model:

- Localization resources belong to application bundles.
- Resources are resolved through the application `extends` chain.
- Capability applications are included only when active for the current application and tenant.
- Local application keys override inherited keys.
- JavaScript receives a stable runtime binding without direct filesystem access.

## Runtime Shape

```text
Application bundle
  config/
    i18n.json
  i18n/
    fr/
      common.json
      orders.json
    en/
      common.json
      orders.json
```

`config/i18n.json` is the i18n configuration resource.

`i18n/{locale}/*.json` are locale message resources. Each locale is a directory containing multiple JSON files. Those files are merged into one logical message map.

## Main Classes

| Class | Responsibility |
|---|---|
| `I18nResource` | Resource object representing either `config` or a locale path such as `fr-FR` |
| `I18nLoader` | Loads `config/i18n.json` or all JSON files under `i18n/{locale}/` |
| `I18nService` | Resolves inherited config/messages, locales, fallbacks, keys, and interpolation |
| `I18nProxy` | Java object injected into GraalVM as `$I18n` |
| `ILoader` | Registers `IResource.I18N_RESOURCE` and returns `I18nLoader` |
| `IResource` | Defines `I18N_RESOURCE = "i18n"` and `I18N_DIR = "i18n"` |

## Binding Injection

`$I18n` is injected into API scripts by `ApiResourceHandler` together with the existing bindings:

```java
scriptingEngine.putMember("$I18n", new I18nProxy(i18nService, application, request, principal, currentTenantId));
```

It is also injected by `HookEngine` when hooks run in standalone mode:

```java
engine.putMember("$I18n", new I18nProxy(I18N_SERVICE, context.getApplication(), context.getRequest(), context.getPrincipal(), context.getTenantId()));
```

When hooks run inside the shared API engine, they reuse the binding already installed by `ApiResourceHandler`.

## Resource Types

The i18n resource has two logical paths.

| Path | Physical Location | Purpose |
|---|---|---|
| `config` | `config/i18n.json` | Locale resolution and fallback configuration |
| locale tag, for example `fr` | `i18n/fr/*.json` | Message translations for that locale |

`I18nLoader.load(...)` branches on the resource path:

```java
if (I18nResource.CONFIG_PATH.equals(i18nResource.getPath())) {
    return loadConfigFile(application, i18nResource);
}
return loadLocaleDirectory(application, i18nResource);
```

This prevents PayOS from looking for the i18n configuration inside `i18n/config/`.

## Inheritance Resolution

`I18nService` resolves both configuration and message resources through the same recursive flow:

```text
resolveMergedResource(application, tenantId, path)
  -> collect parent applications first
  -> skip inactive capability parents
  -> load current application's i18n resource
  -> deep merge into accumulated result
```

The merge order is parent first, child last.

This means local application keys override inherited keys, while missing local keys are inherited.

Example:

Parent:

```json
{
  "orders": {
    "found": "Parent message",
    "missingLocal": "Inherited message"
  }
}
```

Child:

```json
{
  "orders": {
    "found": "Local message"
  }
}
```

Effective result:

```json
{
  "orders": {
    "found": "Local message",
    "missingLocal": "Inherited message"
  }
}
```

## Capability Awareness

The service uses the existing capability activation model.

During inheritance traversal, if an application is a capability, it is included only when active for:

```text
capabilityId + requestingAppId + tenantId
```

Inactive capabilities are skipped as if they were not present in the inheritance chain.

## Locale Resolution

Locale resolution is handled by `I18nService.buildLocaleCandidates(...)`.

Resolution order:

1. Explicit locale override from `$I18n.withLocale(locale)` or `$I18n.t(key, params, locale)`.
2. Header configured by `overrideHeaderName`, default `X-Locale`.
3. Principal fields: `locale`, `language`, `preferredLocale`.
4. Header configured by `headerName`, default `Accept-Language`.
5. `defaultLocale` from merged config.
6. `fallbackLocale` from merged config.
7. Platform default `en`.

If `supportedLocales` is configured, candidates outside the allow-list are discarded.

Locale values are normalized through `Locale.forLanguageTag(...)` after replacing `_` with `-`.

Examples:

```text
fr_FR -> fr-FR
fr-fr -> fr-FR
ar-ma -> ar-MA
```

## Translation Fallback

Translation lookup uses all acceptable request locales, not only the first one.

For example:

```text
Accept-Language: ar-MA,fr;q=0.9,en;q=0.8
```

The service can attempt:

```text
ar-MA
ar
fr
en
```

The configured `fallbackLocale` and platform default `en` are appended after request candidates. Duplicates are removed while preserving order.

For regional locales, the language-only fallback is added automatically:

```text
fr-FR -> fr-FR, fr
```

## Key Resolution

Translation keys use dot notation over nested JSON maps.

```text
orders.found
```

is resolved as:

```json
{
  "orders": {
    "found": "..."
  }
}
```

If the resolved value exists, it is converted to a string and interpolated.

## Interpolation

Interpolation is simple token replacement.

Template:

```text
Commande {orderId} trouvée
```

Parameters:

```json
{
  "orderId": "123"
}
```

Result:

```text
Commande 123 trouvée
```

There is no expression evaluation inside messages.

## Missing Key Policy

When a key cannot be resolved, `missingKeyMode` controls the result.

| Mode | Result |
|---|---|
| `key` | returns the original key |
| `empty` | returns an empty string |
| `bracket` | returns the key surrounded by brackets |

Default mode is `key`.

## Loader and Cache

`I18nLoader` caches loaded resources in memory.

Config cache key:

```text
absolute path to config/i18n.json
```

Config fingerprint:

```text
lastModified:length
```

Locale directory cache key:

```text
absolute path to i18n/{locale}/
```

Locale directory fingerprint:

```text
fileName:lastModified:length;fileName:lastModified:length;...
```

Only `.json` files directly under the locale directory are loaded. Files are sorted by name before merging, making the result deterministic.

When any file timestamp or size changes, the resource is reloaded.

## Security Boundaries

The JavaScript layer never receives filesystem paths or arbitrary file access. It receives only the `$I18n` proxy.

Allowed JS operations:

```javascript
$I18n.locale()
$I18n.t("orders.found")
$I18n.t("orders.found", { orderId: "123" })
$I18n.t("orders.found", { orderId: "123" }, "en")
$I18n.exists("orders.found")
$I18n.withLocale("en").t("orders.found")
```

The loader still uses `CryptoService.decryptIfEncrypted(...)`, so encrypted i18n files follow the same runtime decryption path as other encrypted PayOS resources.

## Error Handling

`I18nService.collectInto(...)` catches loading errors while resolving resources. A malformed i18n file does not break request processing; the affected messages behave like missing keys and follow `missingKeyMode`.

This keeps localization failures from interrupting financial request processing.

## Test Coverage

Current focused tests cover:

- inherited messages are merged without overriding local keys;
- multiple JSON files under one locale directory are merged;
- inherited `config/i18n.json` is merged with local config;
- locale resolution from `Accept-Language`;
- fallback from a selected locale to another accepted locale;
- missing key rendering through `missingKeyMode`.

## Extension Points

Potential future additions:

- tenant-specific i18n override folders;
- ICU pluralization support;
- typed formatting helpers such as currency, date, datetime, number;
- startup validation for missing keys;
- operational metrics for missing translations.
