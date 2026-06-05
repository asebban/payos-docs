# Internationalization (`$I18n`)

PayOS resolves user-facing messages per request locale. Translation files live in an
application's `i18n/` directory and are exposed to scripts through the `$I18n` binding. This
page covers usage; the global i18n configuration keys are in
[configuration/i18n.md](../configuration/i18n.md).

## Translation files

Place locale resources in `i18n/`, one file per locale (resource type `I18N_RESOURCE`):

```
apps/{appId}/i18n/
├── en.json
├── fr.json
└── ar.json
```

```json
// i18n/en.json
{ "payment.declined": "Payment declined", "greeting": "Hello, {name}" }
```

```json
// i18n/fr.json
{ "payment.declined": "Paiement refusé", "greeting": "Bonjour, {name}" }
```

## Resolving messages

```javascript
function execute(request, controlData) {
    return {
        title:   $I18n.t("payment.declined"),
        welcome: $I18n.t("greeting", { name: $Principal.get("name") })
    };
}
```

`$I18n` resolves against the **request locale**, falling back per the configured fallback
locale.

## Locale resolution

The locale is taken from a configurable request header and validated against the supported
list. The relevant configuration (`i18n` block, from `IConfigSpec`):

| Key | Purpose |
| --- | --- |
| `headerName` | Request header carrying the locale (e.g. `Accept-Language`). |
| `defaultLocale` | Locale used when none is provided. |
| `fallbackLocale` | Locale used when a key is missing in the requested locale. |
| `supportedLocales` | Allowed locales. |

Full detail: [configuration/i18n.md](../configuration/i18n.md).

## If `$I18n` is missing

`$I18n` is injected when i18n is configured. Without it, resolve text directly in your script
or add an `i18n` configuration block.

## Next

- [Configuration: i18n](../configuration/i18n.md)
