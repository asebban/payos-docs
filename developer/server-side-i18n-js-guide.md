# Server-Side i18n for JavaScript APIs

Last Alignment: 2026-05-06

PayOS exposes server-side localization to backend JavaScript through the `$I18n` binding. Use it inside API handlers and hooks to return translated messages according to the current request locale.

## Directory Structure

Each application can provide its own localization resources under the application bundle.

```text
app1/
  config/
    i18n.json
  i18n/
    fr-FR/
      orders.json
      errors.json
      common.json
    fr/
      orders.json
      common.json
    en/
      messages.json
```

Locales are directories, not single JSON files. Every `.json` file inside a locale directory is loaded and merged as if it were one large locale file.

PayOS reads locale files in alphabetical order by file name, so prefer stable names such as:

```text
common.json
errors.json
orders.json
```

## Configuration

Create `config/i18n.json` in the application bundle when the defaults are not enough.

```json
{
  "defaultLocale": "fr",
  "fallbackLocale": "en",
  "supportedLocales": ["fr-FR", "fr", "en", "ar-MA"],
  "missingKeyMode": "bracket",
  "headerName": "Accept-Language",
  "overrideHeaderName": "X-Locale"
}
```

| Property | Purpose | Default |
|---|---|---|
| `defaultLocale` | Locale used when the request does not provide one | `en` |
| `fallbackLocale` | Locale used when a key is missing in the selected locale | `en` |
| `supportedLocales` | Optional allow-list of locales | all locales allowed |
| `missingKeyMode` | How missing keys are rendered | `key` |
| `headerName` | Header used for language negotiation | `Accept-Language` |
| `overrideHeaderName` | Header used to force a locale | `X-Locale` |

### Language Headers

`headerName` and `overrideHeaderName` both influence locale resolution, but they have different purposes.

`headerName` is the normal language negotiation header. By default it is `Accept-Language`, which is the standard HTTP header sent by browsers and API clients to describe the user's language preferences.

Example request:

```text
Accept-Language: ar-MA,fr;q=0.9,en;q=0.8
```

With the default configuration, PayOS reads this header and builds an ordered list of acceptable locales:

```text
ar-MA
fr
en
```

The first supported locale becomes the current locale returned by `$I18n.locale()`. During translation lookup, PayOS can still try the other accepted locales as fallbacks when the first locale does not contain the requested key.

Use `headerName` if your API gateway or clients send language preferences in another header.

```json
{
  "headerName": "X-Accept-Language"
}
```

Then PayOS reads:

```text
X-Accept-Language: fr-FR,fr;q=0.9,en;q=0.8
```

`overrideHeaderName` is stronger. It is used to force one locale for the current request. By default it is `X-Locale`.

Example request:

```text
X-Locale: en
Accept-Language: fr-FR,fr;q=0.9
```

In this case PayOS uses `en` before looking at `Accept-Language`. This is useful for backend-to-backend calls, tests, admin tools, or clients that expose an explicit language selector.

You can rename this override header if your platform already has a convention.

```json
{
  "overrideHeaderName": "X-PayOS-Locale"
}
```

Then PayOS reads:

```text
X-PayOS-Locale: ar-MA
```

In short:

| Header config | Default header | Meaning | Priority |
|---|---|---|---|
| `overrideHeaderName` | `X-Locale` | Force this request locale | higher |
| `headerName` | `Accept-Language` | Negotiate from client preferences | lower |

Missing key modes:

| Mode | Result for `orders.created` |
|---|---|
| `key` | `orders.created` |
| `empty` | empty string |
| `bracket` | `[orders.created]` |

## Translation Files

Translation files are regular JSON objects. Nested keys are addressed with dot notation.

`i18n/fr/orders.json`:

```json
{
  "orders": {
    "found": "Commande {orderId} trouvée",
    "notFound": "Commande {orderId} introuvable"
  }
}
```

`i18n/fr/errors.json`:

```json
{
  "errors": {
    "required": "Le champ {field} est obligatoire"
  }
}
```

The two files are merged into one logical locale bundle.

## Using `$I18n`

Use `$I18n.t(key)` for a simple message.

```javascript
function execute(request) {
  return {
    statusCode: 200,
    body: {
      message: $I18n.t("orders.found")
    }
  };
}
```

Use `$I18n.t(key, params)` to interpolate values.

```javascript
function execute(request) {
  const orderId = request.getPathVariables().OrderId;

  return {
    statusCode: 200,
    body: {
      message: $I18n.t("orders.found", { orderId })
    }
  };
}
```

If the translation is:

```json
{
  "orders": {
    "found": "Commande {orderId} trouvée"
  }
}
```

The result for `orderId = "123"` is:

```text
Commande 123 trouvée
```

Interpolation supports multiple parameters in the same message.

```json
{
  "orders": {
    "status": "Order {orderId} is {status} for {customerName}"
  }
}
```

```javascript
const message = $I18n.t("orders.status", {
  orderId: "123",
  status: "PAID",
  customerName: "ACME"
});
```

Result:

```text
Order 123 is PAID for ACME
```

## Locale Resolution

PayOS resolves the request locale in this order:

1. Explicit locale passed by `$I18n.withLocale(locale)` or `$I18n.t(key, params, locale)`.
2. `X-Locale` or the configured `overrideHeaderName`.
3. Principal locale fields: `locale`, `language`, `preferredLocale`.
4. `Accept-Language` or the configured `headerName`.
5. `defaultLocale`.
6. `fallbackLocale`.
7. Platform default `en`.

Locale tags are normalized. For example:

```text
fr_FR -> fr-FR
fr-fr -> fr-FR
ar-ma -> ar-MA
```

## Fallback Behavior

For a selected locale `fr-FR`, PayOS tries translations in this order:

```text
fr-FR
fr
fallbackLocale
en
```

When the HTTP request contains:

```text
Accept-Language: ar-MA,fr;q=0.9,en;q=0.8
```

PayOS first resolves the current locale as `ar-MA`, but translation lookup can still fall back to `fr` if a key does not exist in `ar-MA`.

## Forcing a Locale

Use `withLocale(locale)` when one part of the response must use a specific language.

```javascript
const userMessage = $I18n.t("orders.found", { orderId: "123" });
const auditMessage = $I18n.withLocale("en").t("orders.found", { orderId: "123" });
```

You can also pass the locale directly to `t`.

```javascript
const message = $I18n.t("orders.found", { orderId: "123" }, "en");
```

## Checking Key Existence

Use `$I18n.exists(key)` when the script needs conditional behavior.

```javascript
const label = $I18n.exists("orders.specialStatus")
  ? $I18n.t("orders.specialStatus")
  : $I18n.t("orders.defaultStatus");
```

## Inheritance

Localization resources follow the application `extends` chain.

If `app1` extends `base-payments`, PayOS loads parent resources first and then local resources. Local keys win over inherited keys.

Parent `base-payments/i18n/fr/orders.json`:

```json
{
  "orders": {
    "found": "Commande {orderId} depuis base",
    "missingLocal": "Message hérité"
  }
}
```

Child `app1/i18n/fr/orders.json`:

```json
{
  "orders": {
    "found": "Commande {orderId} locale"
  }
}
```

Results:

```text
orders.found        -> Commande 123 locale
orders.missingLocal -> Message hérité
```

The same inheritance behavior applies to `config/i18n.json`.

## Recommended Conventions

Use namespaced keys:

```text
orders.found
orders.notFound
errors.required
validation.amount.invalid
```

Keep files small and domain-oriented:

```text
i18n/fr/common.json
i18n/fr/orders.json
i18n/fr/errors.json
i18n/fr/validation.json
```

Avoid putting UI or API logic in translation strings. Translation files should contain messages, labels, and templates only.
