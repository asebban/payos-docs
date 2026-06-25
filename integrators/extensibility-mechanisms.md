# PayOS guide for integrators — customize a delivered application for a customer

> **Targeted audience:** technical teams of **partner integrators** PayOS — those who receive a PayOS application (or product) delivered by the publisher and must
> **personalize for an end customer** (bank, fintech, PSP, merchant, etc.) without modifying the sources delivered.
>
> **Objective:** to provide, in a single document, everything you need to know to carry out a personalization mission for a client using exclusively **mechanisms
> extensibility** of PayOS.
>
> **Prerequisites:** basic knowledge of [PayOS architecture](../architecture/README.md) and a functional PayOS bundle (delivered by the publisher or built with [`payos-runtime`](../operations/deployment.md)).

---

## Table of contents

1. [The role of the integrator in the PayOS ecosystem](#1-the-role-of-the-integrator-in-the-payos-ecosystem)
2. [The golden rule: never modify the delivered application](#2-the-golden-rule--never-modify-the-delivered-application)
3. [Panorama of extensibility mechanisms](#3-panorama-of-extensibility-mechanisms)
4. [Prepare your work environment](#4-prepare-your-work-environment)
5. [Set up the personalization application (“overlay” client)](#5-set-up-the-personalization-application--overlay--client)
6. [Customize via `extends` inheritance](#6-customize-via-extends-inheritance)
7. [Internal hooks — customer-specific business rules](#7-internal-hooks--customer-specific-business-rules)
8. [Outgoing webhooks — connect customer information system](#8-outgoing-webhooks--connect-customer-information-system)
9. [Capabilities — enable/disable optional modules](#9-capabilities--activate/disable-optional-modules)
10. [Multi-tenancy — isolate and configure client environment](#10-multi-tenancy--isolate-and-configure-client-environment)
11. [Client-specific secrets](#11-client-specific-secrets)
12. [Localization and branding](#12-localization-and-branding)
13. [Advanced mechanisms (under coordination with the publisher)](#13-advanced-mechanisms-under-coordination-with-the-publisher)
14. [Extending your applications with capabilities](#14-extending-your-applications-with-capabilities)
15. [Packaging, delivery and operation at the customer’s premises](#15-packaging-delivery-and-operation-at-the-customers-premises)
16. [Best practices, governance and security](#16-best-practices-governance-and-security)
17. [End-to-end case study](#17-end-to-end-case-study)
18. [Appendices](#18-appendices)

---

## 1. The role of the integrator in the PayOS ecosystem

PayOS distinguishes three players around the same installation:

| Actor | Role | What it delivers/has |
| --- | --- | --- |
| **Editor** (S2M) | Designs and maintains the PayOS **kernel** and standard **applications/products/capabilities** (e.g. `payment-gateway`, `wallet`, business capabilities). | The runtime (`payos-runtime`), the reference connectors/extensions, the “basic” applications/products. |
| **Integrator** (you) | Adapts an application/product from the publisher to the **specific needs of a client**: business rules, branding, integrations with the client's IS, languages, tenants. | One or more **personalization applications** (“overlays”), configuration, activated optional capabilities, client-specific secrets and webhooks. |
| **End customer** | Use the solution. | Its tenant, its users, its IS (ERP, core banking, CRM, etc.) connected via API/webhooks. |

This distribution is described in [architecture/ecosystem-surface.md](../architecture/ecosystem-surface.md): PayOS exposes a **governed surface** (APIs, events, SDKs, *extension points*) — the integrator operates **exclusively** through this surface, never on the internal runtime.

```
┌────────────────────────────── ───────────────────────────────┐
│ PayOS Runtime (1 JVM) │
│ │
│ Kernel (rigid — never modified) │
│ └─ query pipeline, JS sandbox, SPI, multi-tenancy │
│ │
│ Editor applications (delivered as is) │
│ └─ payment-gateway, wallet, capabilities standards… │
│ │
│ ┌─────────────────────────── ───────────────────────────┐ │
│ │ YOUR PLAYGROUND: “overlay” application(s) │ │
│ │ - extends: ["payment-gateway", ...] │ │
│ │ - api/, page/, hook/, lib/, i18n/, menu/, files/ │ │
│ │ - webhooks.json, config/*.json │ │
│ └─────────────────────────── ───────────────────────────┘ │
└────────────────────────────── ───────────────────────────────┘
```

---

## 2. The golden rule: never modify the delivered application

**Guiding principle of this entire guide:** an application delivered by the publisher (the “product” or the "base application") remains **intact on the client's disk**. Any customization lives in one or more **separate** applications, which `extends` the base application.

### Why this rule is non-negotiable

| Reason | Consequence if it is not respected |
| --- | --- |
| **Version upgrades** | The publisher delivers a new version of `payment-gateway` → if you have modified its files, the update overwrites or conflicts with your changes. |
| **Support & traceability** | The publisher cannot diagnose an incident on an application that it no longer recognizes. |
| **Compliance (PCI-DSS)** | The publisher's code has been audited/certified in its delivered state. |
| **Reversibility** | Disabling a customization (uninstalling your overlay / disabling a capability) should never require "fixing" the base application. |

### What this means in concrete terms

- You **never** touch files under `apps/payment-gateway/...` (or equivalent).
- All customization goes through:
  - an **overlay application** which `extends` the base application ([§6](#6-customize-via-lheritance-extends)),
  - **hooks** co-located in your overlay ([§7](#7-internal-hooks--client-specific-business-rules)),
  - **webhooks** declared in your overlay ([§8](#8-outgoing-webhooks--connect-the-client-information-system)),
  - **External custom connectors** that could be developed by the integrator for specific client needs
  - **Externally loadable Java classes** extensions
  - **capabilities** activated/deactivated via `cpm` ([§9](#9-capabilities--activate-deactivate-optional-modules)),
  - the **configuration** per tenant (`bootstrap.json`, `multitenancy.tenants[]`) ([§10](#10-multi-tenance--isolate-and-configure-the-client-environment)),
  - **secrets** scoped to the client tenant ([§11](#11-client-specific-secrets)).

> This approach is the direct application of the architectural principle **“Rigid Core + Flexible Guest Layer »** ([architecture/architecture-style.md](../architecture/architecture-style.md)) — except that here, the "rigid core" from your point of view is **the editor application**, and your overlay is the **flexible layer**.

---

## 3. Overview of extensibility mechanisms

PayOS defines **seven extensibility mechanisms** ([architecture/extensibility.md](../architecture/extensibility.md)). The table below reclassifies them from the perspective of an **integrator**: which ones are your daily bread, which are occasional, which require coordination with the publisher.

| # | Mechanism | Typical use for the integrator | Frequency | Section |
| --- | --- | --- | --- | --- |
| 1 | **Application inheritance (`extends`)** | Overload/add API endpoints, pages, menus, libraries, translations without touching the base app | ⭐⭐⭐ Daily | [§6](#6-customize-via-inheritance-extends) |
| 2 | **Internal hooks** | Client-specific business rules (validation, enrichment, error management) | ⭐⭐⭐ Daily | [§7](#7-internal-hooks--client-specific-business-rules) |
| 3 | **Outgoing Webhooks** | Notify the client's IS (ERP, core banking, anti-fraud, etc.) | ⭐⭐⭐ Daily | [§8](#8-outgoing-webhooks--connect-customer-information-system) |
| 4 | **Capabilities (`cpm`)** | Activate/deactivate optional business modules for this client/tenant | ⭐⭐ Frequent | [§9](#9-capabilities--activate/deactivate-optional-modules) |
| 5 | **Multi-tenant configuration** | Isolate the tenant from the client (DB, OIDC, quotas) | ⭐⭐⭐ Daily (initial setup) | [§10](#10-multi-tenancy--isolate-and-configure-the-client-environment) |
| 6 | **Secrets (`$Secrets` / `spm`)** | Store customer-specific identifiers (PSP API keys, certificates, etc.) | ⭐⭐ Frequent | [§11](#11-client-specific-secrets) |
| 7 | **SPI connectors / Java extensions / Transport providers / TCP plugins** | Plug in a specific protocol, store or library required by the client | ⭐ Occasional — **editor coordination recommended** | [§13](#13-advanced-mechanisms-under-coordination-with-the-editor) |

### Quick decision tree

```
The customer's need relates to...

├─ “The content/behavior of an existing screen or endpoint needs to change”
│ → extends + override of the resource (api/, page/, lib/, menu/, i18n/) [§6]
│
├─ “Before/after each API call, you must validate/enrich/log something, add fields in the body, ...”
│ → hook (hooks/pre-request.js, post-request.js, on-error.js) [§7]
│
├─ “The client’s IS must be notified of a business event”
│ → webhooks.json (+ $WebHooks.emit if custom business event) [§8]
│
├─ “An optional functionality in the catalog must be (de)activated”
│ → cpm --activate / --deactivate --tenant <client> [§9]
│
├─ “The client must be isolated (its own base, its own IdP, its quotas)”
│ → multitenancy.tenants[] + authorized-tenants [§10]
│
├─ “The client has their own API identifiers/keys (PSP, ERP, etc.)”
│ → $Secrets + spm/vault [§11]
│
├─ “The customer wants their language, their logo, their colors”
│ → i18n/ + files/ + page override via extends [§12] + CSS
│
└─ “Client requires an unsupported protocol/store/library”
      → SPI connector / Java extension / Transport / TCP plugin — see §13
        and validate the approach with the editor (impact on runtime deployment)
```

---

## 4. Prepare your work environment (usually already done by the editor)

### 4.1 Command line tools

Install the PayOS suite of tools (usually in `~/.payos/bin`):

| Tool | Role | Doc |
| --- | --- | --- |
| `apm` | Install/uninstall **applications** (your overlay) | [cli-tools/apm.md](../cli-tools/apm.md) |
| `cpm` | Install/enable/disable **capabilities** | [cli-tools/cpm.md](../cli-tools/cpm.md) |
| `ppm` | Install **products** (app bundles + config) | [cli-tools/ppm.md](../cli-tools/ppm.md) |
| `spm` | Provision **secrets** (provider filesystem) | [cli-tools/spm.md](../cli-tools/spm.md) |
| `edc` | Encrypt/package final bundle for delivery | [cli-tools/edc.md](../cli-tools/edc.md) |
| `pdoc` | Generate the OpenAPI doc for your new endpoints | [cli-tools/pdoc.md](../cli-tools/pdoc.md) |

```bash
# Linux/macOS — from the payos-pm module
./install-payos-tools.sh
#Windows
.\install-payos-tools.ps1
```

### 4.2 Retrieve the bundle delivered by the publisher

A PayOS bundle has the following structure ([operations/deployment.md](../operations/deployment.md)):

```
bundle/
├── payos.json # entry point (configDir, catalogs)
├── bootstrap.json # runtime configuration (servers, applications, security, ...)
├── apps/
│ └── payment-gateway/ # application(s) delivered by the publisher — DO NOT MODIFY
├── connectors/ # JARs of SPI connectors (DB, queue, secrets, webhooks)
├── extensions/ # Java JARs callable from scripts
└── tcp-handlers/ # TCP plugins (if applicable)
```

> You cannot modify the contents of `apps/payment-gateway/` since the bundle is delivered **encrypted**. Your additions go to a new folder `apps/<your-overlay>/` and a new entry `applications[]` in `bootstrap.json` ([§5](#5-set-up-the-personalization-application--overlay--client)).

### 4.3 Launch the runtime locally to develop/test

```bash
java -jar payos-runtime-<version>.jar --bundle-path ./bundle
```

- **Hot reload** ([operations/hot-reload.md](../operations/hot-reload.md)) means   that most of your modifications (application files, `bootstrap.json`, hooks,   webhooks) are taken into account **without restart**. Only the JARs of   connectors/extensions and listening ports require a reboot.
- In development, activate the **tenant simulator** to test without sending systematically `X-Tenant-Id`:

```json
{
  "multitenancy": {
    "tenantSimulator": { "enabled": true, "tenantId": "client-demo" }
  }
}
```

> ⚠️ The tenant simulator **must be disabled** in any environment delivered to the customer (recipe as production) — see [§16](#16-best-practices-governance-and-security).

---

## 5. Set up the personalization application (“client overlay”)

### 5.1 Naming convention

Create **one overlay application with the pair (client, base application)**, for example:

```
<ClientID>-<BaseApplicationID>
```

Example: the “Atlas” client personalizes the publisher’s `payment-gateway` application → overlay `atlas-payment-gateway`. This has nothing mandatory, it is just a good recommended convention.

Make the new application extend all other applications of the product (except applications that are already extended by other applications in the product), so that you have just a single entry point for all your developments.

### 5.2 Scaffolding

To create you application, you can either let it empty or use the generator provided by the runtime ([developer/create-application-guide.md](../developer/create-application-guide.md#3-scaffolding-rapide)):

```bash
# Linux/macOS
./generate_app.sh --app-id atlas-payment-gateway --output ./bundle/apps

#Windows
.\generate_app.ps1 -AppId atlas-payment-gateway -output .\bundle\apps
```

Then if required, delete the generated pages and/or JS endpoint examples that you are not using — an overlay can contain extensions, i.e. **resources that differ** from the base application and that are specific to your new application.

### 5.3 Declare the overlay in `bootstrap.json`

```json
{
  "apps": [
    {
      "id": "payment-gateway",
      "name": "Payment Gateway",
      "base.path": "/opt/payos/bundle/apps/payment-gateway",
      "version": "2.1.0"
    },
    {
      "id": "currency-converter",
      "name": "Currency converter",
      "base.path": "/opt/payos/bundle/apps/currency-converter",
      "version": "1.1.0"
    },
    {
      "id": "atlas-payment-gateway",
      "name": "Payment Gateway — Atlas",
      "base.path": "/opt/payos/bundle/apps/atlas-payment-gateway",
      "version": "1.0.0",
      "extends": ["payment-gateway", "currency-converter"],
      "authorized-tenants": ["atlas"]
    }
  ]
}
```

Key points:

- `extends: ["payment-gateway", "currency-converter"]` — your overlay **inherits** all resources of `payment-gateway` and `currency-converter` ([§6](#6-customize-via-inheritance-extends)).
- `authorized-tenants: ["atlas"]` — only the `atlas` tenant can access this overlay ([§10](#10-multi-tenancy--isolate-and-configure-the-client-environment)).
- From now on, **Atlas client users call `/atlas-payment-gateway/...`** (and no longer `/payment-gateway/...`). It's this new `id` that you reference in the security configuration (dedicated OIDC clientId), the frontend integrations, etc.

### 5.4 Typical tree structure of an overlay

```
atlas-payment-gateway/
├── manifest.json # used by apm
├──config/
│ ├── application.json # extends (alternative to bootstrap.json), overrides
│ ├── mappings.json # new endpoints / overloaded endpoints
│ ├── routes.json # new pages / overloaded pages
│ └── i18n.json # localization config
├── api/
│ └── payments/
│    └── create.js # payment-gateway/api/payments/create.js overload
├── page/
│ └── index.vue # personalized home page (Atlas branding)
├── lib/
│ └── atlas-rules.js # business rules specific to Atlas
├── i18n/
│ └── ar-MA/
│ └── common.json # Moroccan Arabic translation
├── files/
│ ├── logo-atlas.png
│ └── styles-atlas.css
├── hooks/
│ ├── pre-request.js # Atlas specific validation
│ ├── post-request.js # response enrichment
│ └── on-error.js # custom error handling
├── menu/
│ └── entries.json # additional menu entries
└── webhooks.json # notifications to SI Atlas
```

> You can also manage `extends` and `security` / `database-service` / blocks
> `webhooks` specific to the overlay in `config/application.json` rather than in
> `bootstrap.json` — the two are merged (see
> [configuration/json-configuration-reference.md §5](../configuration/json-configuration-reference.md#5-hierarchical-resolution-of-values)).

---

When an application is inherited (extended), its configuration is also inherited, unless the overlay application defines its own configuration keys that overload the inherited ones.

## 6. Customize via `extends` inheritance

This is the **most used** mechanism by an integrator: it allows you to add or replace an application's resources without touching its standard files.

### 6.1 Resolution principle

`ResourceLocator` resolves a requested resource (API endpoint, page, library, translation, etc.) by traversing the string `extends` **recursively, depth-first**, starting with the called application :

```
1. Search for the resource in atlas-payment-gateway (your overlay) → found?  we stop, she is the one who does it.
   → otherwise, recurse in extends, in the declared order.
2. For each extends entry:
   - if it is a capability that's inactive for this tenant → we ignore it entirely
   - otherwise → we repeat step 1 on this application (or activated capability)
```

Practical consequences:

- **“First found wins”** (`lib/`, `hooks/`, `files/`, `/api`, `/page`, ...) — if you   create `lib/atlas-rules.js` in your overlay while `payment-gateway/lib/atlas-rules.js` exists, **your version is loaded** and the editor's file is never read. 
- **API endpoints and pages follow the same "first found wins" rule**: `mappings.json` and `routes.json` are **inherited** from the parent chain — every endpoint and page declared in `payment-gateway` is already visible in your overlay. To **override** one, drop a same-named file in `api/` or `page/` — no mapping or route redeclaration is needed. To **add** a brand-new endpoint or page that does not exist in the parent, you must still declare it in `config/mappings.json` / `config/routes.json` — see [6.2](#62-endpoints-api--add-overload-extend) and [6.3](#63-pages-components-and-branding).
- **You can add** resources that don't exist in the base app — they are simply added to the application catalog.
- The **menus** and **i18n translations** follow ther same principle as routes and mappings. They use an **aggregation/merger** logic (see [6.5](#65-menus) and [6.6](#66-translations-i18n)).

### 6.2 API Endpoints — add, overload, extend

#### Add a new endpoint

Declare it in `config/mappings.json` of your overlay and create the corresponding script
in `api/`:

```json
//atlas-payment-gateway/config/mappings.json
{
  "mappings": {
    "api": {
      "/payments/{id}/atlas-receipt": {
        "GET": {
          "handler": "payments/atlas-receipt",
          "roles": ["ROLE_USER"]
        }
      }
    }
  }
}
```

```javascript
// atlas-payment-gateway/api/payments/atlas-receipt.js
function loadControlData(request) {
    return { id: request.getPathVariables()["id"] };
}

function execute(request, controlData) {
    var payment = $DB.unique("select p from Payment p where p.id = :id",
        $DB.newParams().put("id", controlData.id));
    if (payment == null) {
        $Errors.notFound("PAYMENT_NOT_FOUND", "Payment not found");
    }
    $Response.setJsonBody({
        reference: "ATLAS-" + payment.id,
        amount: payment.amount,
        currency: payment.currency
    });
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

#### Overload an existing endpoint

Since `mappings.json` is **inherited** from the parent chain, to overload an endpoint you only need to **create the handler script** at the same relative path under `api/` and with the same JS name — the inherited mapping entry (including its `roles` configuration) applies automatically.

Create `atlas-payment-gateway/api/payments/create.js` — `ResourceLocator` finds the inherited mapping for `POST /payments`, looks for the script in your overlay first, and serves your version. `payment-gateway`'s script is never reached for this route again.

> **To also change `roles` or other mapping metadata**, redeclare the entry in your overlay's `config/mappings.json` (child entry wins over the inherited one):
>
> ```json
> // atlas-payment-gateway/config/mappings.json — only needed to change metadata
> {
>   "mappings": {
>     "api": {
>       "/payments": {
>         "POST": {
>           "handler": "payments/create",
>           "roles": ["ROLE_AGENCY_USER"]
>         }
>       }
>     }
>   }
> }
> ```

```javascript
//atlas-payment-gateway/api/payments/create.js
function loadControlData(request) {
    return { body: request.getJsonBody() };
}

function execute(request, controlData) {
    var body = controlData.body;

    // Atlas specific rule: an “agencyCode” field is mandatory
    if (!body.agencyCode) {
        $Errors.badRequest("AGENCY_CODE_REQUIRED", "Agency code is required for Atlas");
    }

    // Delegate standard processing via an internal call to the legacy API
    // (or reproduce/adapt the logic as needed)
    var created = $DB.save("Payment", {
        amount: body.amount,
        currency: body.currency,
        status: "PENDING",
        agencyCode: body.agencyCode
    });

    $Response.setStatusCode(201);
    $Response.setJsonBody(created);
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

> **Extend without duplicating logic:** if you only want to *add* a control before/after standard processing without rewriting the entire endpoint, prefer a **hook**
> (`hooks/pre-request.js` / `post-request.js`, [§7](#7-internal-hooks--client-specific-business-rules)) — it applies to **all** endpoints without duplicating each script.

#### Overload an existing endpoint but keep using the editor's logic

Sometimes you only need to add a step around the standard logic — pre-process the input, post-process the output — without rewriting the whole endpoint. Use `$Api.setApp()` to retarget `$Api` to the parent application, then call the same path: `ApiProxy` runs the parent's script in the context of `payment-gateway` and returns its response.

```javascript
// atlas-payment-gateway/api/payments/create.js
function loadControlData(request) {
    return { body: request.getJsonBody() };
}

function execute(request, controlData) {
    var body = controlData.body;

    // --- Atlas-specific pre-processing ---
    if (!body.agencyCode) {
        $Errors.badRequest("AGENCY_CODE_REQUIRED", "Agency code is required for Atlas");
    }

    // --- Delegate to the editor's script ---
    // Switch $Api to target payment-gateway so the call resolves its own create.js,
    // not this overlay script (which would cause infinite recursion).
    $Api.setApp($App.get("payment-gateway"));
    $Api.setHeaders({ "X-Correlation-Id": request.getHeader("X-Correlation-Id") });
    var baseResponse = $Api.post("/payments", JSON.stringify(body));

    // --- Atlas-specific post-processing ---
    if (baseResponse.getStatusCode() === 201) {
        var created = baseResponse.getJsonBody();
        $Response.setStatusCode(201);
        $Response.setJsonBody({
            id:          created.id,
            reference:   "ATLAS-" + created.id,
            agencyCode:  body.agencyCode,
            amount:      created.amount,
            currency:    created.currency
        });
    } else {
        // Propagate the editor's error response unchanged
        $Response.setStatusCode(baseResponse.getStatusCode());
        $Response.setBody(baseResponse.getBody());
    }
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

> **Why `$App.get("payment-gateway")`?**  Without retargeting, `$Api.post("/payments", ...)` would resolve against the *current* application (the overlay) and call **this very script** again — infinite recursion. By switching the app to `payment-gateway`, `ApiProxy` resolves `/payments POST` in that app's context and finds the parent's `api/payments/create.js`.
>
> **Prefer a hook when possible.** If all you need is pre/post logic without inspecting or replacing the parent's response body, a `hooks/pre-request.js` / post-request.js` is simpler and keeps the two scripts fully independent.


### 6.3 Pages, components and branding

Like API endpoints ([6.2](#62-endpoints-api--add-overload-extend)), page resolution follows **"first found wins"**: `routes.json` is **inherited** from the parent chain, so every page declared in `payment-gateway` is already visible in your overlay. To override one, simply create the `.vue` component at the same relative path under `page/` with the same page name — no route redeclaration needed. Parent routes for any path you don't override remain fully accessible.

> **To also change route metadata** (component alias, props…), redeclare the entry in your overlay's `config/routes.json` (child entry wins over the inherited one):
>
> ```json
> // atlas-payment-gateway/config/routes.json — only needed to change metadata
> {
>   "routes": [
>     { "path": "/", "component": "index" }
>   ]
> }
> ```

Create the corresponding component:

```vue
<!-- atlas-payment-gateway/page/index.vue -->
<template>
  <div class="atlas-theme">
    <img src="/atlas-payment-gateway/files/logo-atlas.png" alt="Atlas" />
    <h1>Payments Area — Atlas</h1>
    <PaymentList /> <!-- component inherited from payment-gateway if not overridden -->
  </div>
</template>
```

> **Legacy limitation:** the page router does an **exact** match on the path (no `:id`/`{id}`). For detail screens, pass the identifier in *query parameter* — see [developer/create-application-guide.md §5.2](../developer/create-application-guide.md#52-configroutesjson--routes-de-pages).


### 6.4 Shared libraries

A `lib/<name>.js` library in your overlay can:

- **add** a client-specific library (`lib/atlas-rules.js`),
- **overload** a library of the base application by recreating `lib/<same-name>.js`.

```javascript
// atlas-payment-gateway/lib/atlas-rules.js
function isValidAgencyCode(code) {
    return /^AT-\d{4}$/.test(code);
}

({ isValidAgencyCode: isValidAgencyCode })
```

```javascript
// use in a hook or an endpoint of the overlay
var atlasRules = $Library.load("atlas-rules");
if (!atlasRules.isValidAgencyCode(body.agencyCode)) {
    $Errors.badRequest("INVALID_AGENCY_CODE", "Invalid agency code");
}
```

> `$Library.load(name)` first resolves in the current application, then moves up the chain
> `extends` — your overlay can therefore also **reuse** the libraries of
> `payment-gateway` without duplicating them.

### 6.5 Menus

Unlike APIs/pages/libs, the menu entries of the entire `extends` chain are **aggregated** (and not replaced) — see [developer/api-contracts.md](../developer/api-contracts.md#get-appidmenu).

```json
// atlas-payment-gateway/menu/entries.json
[
  { "id": "atlas-receipts", "label": "Atlas Supporting Documents", "page": "atlas/receipts" }
]
```

The user therefore sees the `payment-gateway` menu **plus** your additional entries. If a legacy and active capability also has a `menu/entries.json`, its entries are prefixed with `{capabilityId}/` automatically.

### 6.6 Translations (i18n)

The `i18n/` resources follow the `extends` chain with **merge + overload**: messages parent apps are loaded first, then your overlay overloads the keys that already exist and adds the new ones.

```json
// atlas-payment-gateway/i18n/fr/orders.json
{
  "orders": {
    "created": "Order {orderId} created for agency {agencyCode}"
  }
}
```

```json
// atlas-payment-gateway/i18n/ar-MA/common.json (new locale added by the integrator)
{
  "welcome": "مرحبا بكم في بوابة الدفع"
}
```

Don't forget to add `ar-MA` to `supportedLocales` in `config/i18n.json` of the overlay (this file overloads that of the base app) — see [§12](#12-localization-and-design-branding) and [configuration/json-configuration-reference.md §3.9](../configuration/json-configuration-reference.md#39-configi18njson-per-application).

### 6.7 Webhooks (merge with override by `id`)

`webhooks.json` of the entire `extends` chain is **merged**: your entries are taken into account first, then those of the parent applications, and **an entry from your overlay bearing the same `id` that an inherited entry replaces** (URL, secret, retry…). To **disable** a legacy notification without modifying it, redeclare its `id` with `"disabled": true`. Details and complete examples in [§8](#8-outgoing-webhooks--connect-the-client-information-system).

### 6.8 Limitations and precautions

- **Don't duplicate an entire file** just to change a minor detail — prefer a hook that modifies `$Response` after running the legacy script (this avoids drift if   the editor fixes a bug in its version). 
- Document (internal changelog) each overloaded resource: at each version upgrade of the base application, check if the original resource has changed and if your overhead must be updated.
- Systematically test the **legacy path AND the overloaded path** after all modification of the overlay (see [§16](#16-best-practices-governance-and-security)).

### 6.9 Complete example — extending an entity with a new field (end-to-end)

**Scenario:** `payment-gateway` already exposes a User profile feature — `GET /users/{id}/profile`, `PUT /users/{id}/profile`, and a `page/users/profile.vue` page. The Atlas bank requires a `loyaltyCardNumber` field on each user, which does **not** exist in the editor's data model or schema.

The integrator must:

1. Store the new field in a **dedicated extension table** (never altering the editor's schema).
2. **Enrich** the GET endpoint to include the field in the response.
3. **Extend** the PUT endpoint to validate and persist the field.
4. **Display and edit** it in the profile page.

#### Step 1 — Database: create the extension table

This migration runs once when deploying the overlay (e.g. via Liquibase/Flyway in your delivery pipeline). With `isolationMode: dedicated-schema`, the table lives in the `atlas` schema and is automatically invisible to other tenants.

```sql
-- atlas_user_extensions table — one row per (user, tenant)
CREATE TABLE IF NOT EXISTS atlas_user_extensions (
    user_id              VARCHAR(64) NOT NULL,
    tenant_id            VARCHAR(64) NOT NULL,
    loyalty_card_number  VARCHAR(32),
    PRIMARY KEY (user_id, tenant_id)
);
```

#### Step 2 — Override the GET endpoint

Create `atlas-payment-gateway/api/users/profile.js`. The mapping `GET /users/{id}/profile` is **inherited** from `payment-gateway` — only the handler script file is needed, no `mappings.json` change.

```javascript
// atlas-payment-gateway/api/users/profile.js
// GET /users/{id}/profile — enriched with loyaltyCardNumber for Atlas

function loadControlData(request) {
    return { userId: request.getPathVariables()["id"] };
}

function execute(request, controlData) {
    var userId = controlData.userId;

    // 1. Delegate to the editor's endpoint for the standard profile fields
    $Api.setApp($App.get("payment-gateway"));
    $Api.setHeaders({
        "X-Correlation-Id": request.getHeader("X-Correlation-Id"),
        "X-Tenant-Id":      $Tenant
    });
    var baseResponse = $Api.get("/users/" + userId + "/profile");

    if (baseResponse.getStatusCode() !== 200) {
        $Response.setStatusCode(baseResponse.getStatusCode());
        $Response.setBody(baseResponse.getBody());
        return $Response;
    }

    // 2. Load the Atlas extension row (returns null if not yet set)
    var ext = $DB.unique(
        "SELECT e FROM AtlasUserExtension e WHERE e.userId = :uid AND e.tenantId = :tid",
        $DB.newParams().put("uid", userId).put("tid", $Tenant)
    );

    // 3. Merge and return the enriched profile
    var profile = baseResponse.getJsonBody();
    profile.loyaltyCardNumber = ext != null ? ext.loyaltyCardNumber : null;

    $Response.setStatusCode(200);
    $Response.setJsonBody(profile);
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

#### Step 3 — Override the PUT endpoint

Create `atlas-payment-gateway/api/users/update-profile.js` (assuming the inherited mapping for `PUT /users/{id}/profile` uses handler `users/update-profile`).

```javascript
// atlas-payment-gateway/api/users/update-profile.js
// PUT /users/{id}/profile — validates and persists loyaltyCardNumber

function loadControlData(request) {
    return {
        userId: request.getPathVariables()["id"],
        body:   request.getJsonBody()
    };
}

function execute(request, controlData) {
    var userId = controlData.userId;
    var body   = controlData.body;

    // Atlas-specific validation
    if (body.loyaltyCardNumber !== undefined && body.loyaltyCardNumber !== null) {
        if (!/^ATLAS-\d{8}$/.test(body.loyaltyCardNumber)) {
            $Errors.badRequest(
                "INVALID_LOYALTY_CARD",
                "Loyalty card number must follow the format ATLAS-XXXXXXXX"
            );
        }
    }

    // Delegate standard fields to the editor's endpoint
    // Strip the extension field so the editor's script does not receive unknown data
    $Api.setApp($App.get("payment-gateway"));
    $Api.setHeaders({
        "X-Correlation-Id": request.getHeader("X-Correlation-Id"),
        "X-Tenant-Id":      $Tenant
    });
    var baseBody = JSON.parse(JSON.stringify(body));   // shallow copy
    delete baseBody.loyaltyCardNumber;
    var baseResponse = $Api.put("/users/" + userId + "/profile", JSON.stringify(baseBody));

    if (baseResponse.getStatusCode() !== 200) {
        $Response.setStatusCode(baseResponse.getStatusCode());
        $Response.setBody(baseResponse.getBody());
        return $Response;
    }

    // Upsert the Atlas extension row
    if (body.loyaltyCardNumber !== undefined) {
        var existing = $DB.unique(
            "SELECT e FROM AtlasUserExtension e WHERE e.userId = :uid AND e.tenantId = :tid",
            $DB.newParams().put("uid", userId).put("tid", $Tenant)
        );
        if (existing != null) {
            existing.loyaltyCardNumber = body.loyaltyCardNumber;
            $DB.save("AtlasUserExtension", existing);
        } else {
            $DB.save("AtlasUserExtension", {
                userId:             userId,
                tenantId:           $Tenant,
                loyaltyCardNumber:  body.loyaltyCardNumber
            });
        }
    }

    // Return the enriched updated profile
    var updated = baseResponse.getJsonBody();
    updated.loyaltyCardNumber = (body.loyaltyCardNumber !== undefined)
        ? body.loyaltyCardNumber
        : null;

    $Response.setStatusCode(200);
    $Response.setJsonBody(updated);
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

> **Why delete before forwarding?** The editor's endpoint does not know the `loyaltyCardNumber` field. Passing it through could trigger an unknown-field validation error in the editor's script or silently pollute the entity. Always strip extension fields before delegating to the parent.

#### Step 4 — Override the profile page

Create `atlas-payment-gateway/page/users/profile.vue`. The route is **inherited** from `payment-gateway`; only the component file is needed.

```vue
<!-- atlas-payment-gateway/page/users/profile.vue -->
<template>
  <div class="atlas-profile">
    <h2>{{ $t('profile.title') }}</h2>
    <form @submit.prevent="save">

      <!-- Standard fields — reuse the editor's base component unchanged -->
      <BaseProfileFields v-model="profile" />

      <!-- Atlas extension field -->
      <div class="form-group atlas-extension">
        <label for="loyaltyCardNumber">{{ $I18n.t('profile.loyaltyCardNumber') }}</label>
        <input
          id="loyaltyCardNumber"
          v-model="profile.loyaltyCardNumber"
          type="text"
          placeholder="ATLAS-XXXXXXXX"
          :class="{ 'is-invalid': errors.loyaltyCardNumber }"
        />
        <span v-if="errors.loyaltyCardNumber" class="error-msg">
          {{ errors.loyaltyCardNumber }}
        </span>
      </div>

      <button type="submit" :disabled="saving">
        {{ saving ? $t('common.saving') : $t('common.save') }}
      </button>
    </form>
  </div>
</template>

<script>
export default {
  name: 'AtlasUserProfile',

  data() {
    return {
      profile: {},
      errors:  {},
      saving:  false
    };
  },

  async created() {
    const userId = this.$route.query.id;
    // GET resolves to the overlay's enriched handler (step 2)
    const res = await this.$api.get(`/atlas-payment-gateway/api/users/${userId}/profile`);
    if (res.ok) {
      this.profile = await res.json();
    }
  },

  methods: {
    async save() {
      this.errors = {};
      this.saving = true;
      try {
        const userId = this.$route.query.id;
        const res = await this.$api.put(
          `/atlas-payment-gateway/api/users/${userId}/profile`,
          this.profile
        );
        if (!res.ok) {
          const err = await res.json();
          if (err.code === 'INVALID_LOYALTY_CARD') {
            this.errors.loyaltyCardNumber = this.$t('profile.errors.invalidLoyaltyCard');
          }
        }
      } finally {
        this.saving = false;
      }
    }
  }
};
</script>
```

#### Step 5 — Add the i18n keys

```json
// atlas-payment-gateway/i18n/en/profile.json
{
  "profile": {
    "title": "My Profile",
    "loyaltyCardNumber": "Loyalty Card Number",
    "errors": {
      "invalidLoyaltyCard": "Card number must follow the format ATLAS-XXXXXXXX"
    }
  }
}
```

#### Summary — files created in the overlay

```
atlas-payment-gateway/
├── api/
│   └── users/
│       ├── profile.js             ← GET override: delegates to parent, then merges extension field
│       └── update-profile.js      ← PUT override: validates + upserts extension row
├── page/
│   └── users/
│       └── profile.vue            ← page override: adds loyaltyCardNumber input
└── i18n/
    └── en/
        └── profile.json           ← translation keys for the new field
```

No `config/mappings.json` or `config/routes.json` changes are required — both endpoints and the page route are **inherited** from `payment-gateway`. Only the handler and component files are overridden. The `atlas_user_extensions` table must exist in the Atlas tenant's schema before the first request.

---

## 7. Internal hooks — customer-specific business rules

Hooks are the preferred mechanism for **adding transversal behavior** (validation, enrichment, logging, error handling, notifications) **without rewriting endpoints** of the basic application. They live in `hooks/` at the root of your overlay and are loaded **automatically** — no additional declaration is necessary.

### 7.1 Available interception points

| Hook file | Triggered | Specific bindings |
| --- | --- | --- |
| `hooks/pre-request.js` | Before running the API script | `$Request`, `$HookChain`, `$Principal` |
| `hooks/post-request.js` | After successful execution | `$Request`, `$Response`, `$HookChain`, `$Principal` |
| `hooks/on-error.js` | On exception in API pipeline | `$Error`, `$HookChain`, `$Principal` |
| `hooks/page-pre-serve.js` | Before assembling a View page | `$Page`, `$HookChain` |
| `hooks/page-post-serve.js` | After assembling a View page | `$Page`, `$HookChain` |
| `hooks/page-on-error.js` | On exception when rendering a page | `$Error`, `$Page`, `$HookChain` |

All hooks also receive the standard bindings (`$App`, `$Tenant`, `$Logger`, `$DB`, `$Queue`, `$WebHooks`, `$I18n`…) — see [architecture/hooks-webhooks.md §2.4](../architecture/hooks-webhooks.md#24-script-bindings-in-hooks).

### 7.2 Resolution and delegation — `$HookChain`

The resolution is **child-first** on the `extends` string: the **first** hook file found (your overlay first, then `payment-gateway`, then its own `extends`…) is the one that executes. Parent hook **is only executed if yours explicitly calls `$HookChain.proceed()`**.

| Call | Effect |
| --- | --- |
| *(nothing)* | The parent hook (if any) is **ignored**; the API script runs normally after the hook |
| `$HookChain.proceed()` | Execute the **next** hook found in the `extends` chain (e.g. that of `payment-gateway`) |
| `$HookChain.stop()` | **Interrupts** the pipeline — `$Response` is returned as is, API script does not run |

> If `payment-gateway` has **no** hook for a given point, your overlay hook becomes the effective hook and executes normally (nothing to delegate).
>
> ⚠️ `proceed()` and `stop()` must not be called **both in the same execution path** — calling one after the other in sequence results in undefined behaviour. Using them in separate branches of an `if/else` is perfectly valid, since only one branch runs per invocation. `proceed()` only has an effect once even if called multiple times.

### 7.3 Recipe — client-specific pre-request validation

```javascript
// atlas-payment-gateway/hooks/pre-request.js
// Applies to ALL overlay (and legacy) API endpoints called via this appId.
var path = $Request.getPath();

if (path.indexOf("/payments") === 0 && $Request.getMethod() === "POST") {
    var body = $Request.getJsonBody();
    if (!body || !body.agencyCode) {
        $Response.setStatusCode(400);
        $Response.setJsonBody({
            error: "AGENCY_CODE_REQUIRED",
            message: "The Atlas agency code is required"
        });
        $HookChain.stop();   // don't run the API script
        return;
    }
}

// Let the editor execute its possible pre-request hook, then the API script
$HookChain.proceed();
```

### 7.4 Recipe — systematically enrich the answers

```javascript
// atlas-payment-gateway/hooks/post-request.js
$Response.addHeader("X-Atlas-Processed", "true");
$Response.addHeader("X-Tenant", $Tenant);

$Logger.info("Atlas — query processed: {} {} ({})",
    $Request.getMethod(), $Request.getPath(), $Response.getStatusCode());

// Let the payment-gateway post-request execute afterwards (if it exists)
$HookChain.proceed();
```

### 7.5 Recipe — personalized and localized error handling

```javascript
//atlas-payment-gateway/hooks/on-error.js
$Logger.error("Atlas — pipeline error on {} {}: {}",
    $Request.getMethod(), $Request.getPath(), $Error.getMessage());

// Notify the Atlas IS in the event of an error on payments (see §8)
if ($Request.getPath().indexOf("/payments") === 0) {
    $WebHooks.emit("atlas.payment.error", {
        path: $Request.getPath(),
        error: $Error.getMessage(),
        tenantId: $Tenant,
        correlationId: $Request.getHeader("X-Correlation-Id")
    });
}

// Localized error response for end user
$Response.setStatusCode(500);
$Response.setJsonBody({
    error: "INTERNAL_ERROR",
    message: $I18n.t("errors.internal")
});
$HookChain.stop();
```

### 7.6 Recipe — customize page rendering (branding/A-B routing)

```javascript
//atlas-payment-gateway/hooks/page-pre-serve.js
// Example: redirect Atlas agents to a dedicated page variant
if ($Page.getRequestPath() === "/" && $Principal != null) {
    var roles = $Principal.get("roles");
    if (roles && roles.indexOf("ROLE_AGENCY") !== -1) {
        $Response.setHeader("X-Atlas-Variant", "agency");
    }
}
$HookChain.proceed();
```

### 7.7 The “publisher lock”

The editor can mark one of its hooks as `locked: true` in a `config/hooks.json` manifest inside the application directory:

```json
// payment-gateway/config/hooks.json
{
  "pre-request.js": { "locked": true }
}
```

When a hook node is locked, any child hook (yours) that calls `$HookChain.stop()` while that node is downstream will trigger a **pipeline error**: the engine halts the pipeline and logs a lock violation — the locked hook cannot be bypassed. Normal calls to `$HookChain.proceed()` are unaffected.

This mechanism protects controls that cannot be circumvented (e.g. anti-fraud checks, regulatory audit trails). If your hook encounters a lock violation at runtime, **do not look for a workaround** — escalate to the publisher.

---

## 8. Outbound webhooks — connect customer information system

Webhooks notify **asynchronously** (HTTP POST signed HMAC-SHA256, or Message Queue) to an external system — typically the client's **core banking**, **ERP** or **monitoring tool** — during a PayOS event. Full reference:
[architecture/eventing-webhooks.md](../architecture/eventing-webhooks.md) and
[developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

### 8.1 Declare a subscription in `webhooks.json`

```json
//atlas-payment-gateway/webhooks.json
{
  "webhooks": [
    {
      "id": "atlas-payment-completed",
      "event": "api.post-request", // sent automatically after a request
      "native": true, // automatic sending
      "filter": {
        "path": "/payments",
        "method": "POST",
        "statusCodes": [200, 201]
      },
      "url": "${ATLAS_CORE_BANKING_URL}/events/payment-completed",
      "secret": "${ATLAS_WEBHOOK_SECRET}",
      "headers": { "X-Source": "payos-atlas" },
      "retry": { "maxAttempts": 5, "backoffMs": 2000 }
    },
    {
      "id": "atlas-payment-error",
      "event": "atlas.payment.error",
      "native": false, // sent manually from an API endpoint
      "url": "${ATLAS_CORE_BANKING_URL}/events/payment-error",
      "secret": "${ATLAS_WEBHOOK_SECRET}"
    }
  ]
}
```

| Field | Notes for the integrator |
| --- | --- |
| `id` | Prefix with the client name (`atlas-...`) to avoid collision with base app `id`s when merging. |
| `event` | `native: true` ⇒ must be a [known system event](../architecture/eventing-webhooks.md#system-events) (`api.*`, `page.*`, `security.*`, `capability.*`). `native: false` ⇒ free name, triggered by `$WebHooks.emit(...)` in a hook/endpoint. |
| `url`, `secret` | Use **placeholders `${VAR}`** resolved from the environment — never hardcode client URLs/secrets (see [§11](#11-client-specific-secrets) and [configuration/env-var-resolution.md](../configuration/env-var-resolution.md)). |
| `filter` | Restricts triggering to relevant requests (path, method, status codes). |
| `retry` | `maxAttempts` / `backoffMs` — linear backoff; beyond that, the event is logged as an error (and sent as a *dead-letter* if configured). |

### 8.2 Overriding or disabling a legacy webhook

`webhooks.json` of the entire `extends` chain is **merged**; an entry from your overlay bearing the **same `id`** that a `payment-gateway` entry completely replaces it (URL, secret, filter, retry).

```json
// disable a legacy audit notification that the client does not want
{
  "webhooks": [
    { "id": "api-post-request-audit", "disabled": true }
  ]
}
```

### 8.3 Emit a custom business event (`$WebHooks.emit`)

For customer-specific events (not covered by the system catalog), use `$WebHooks.emit` from a hook or endpoint:

```javascript
// in an endpoint or a post-request hook
if ($WebHooks.hasSubscribers("atlas.large-transfer-detected")) {
    $WebHooks.emit("atlas.large-transfer-detected", {
        paymentId: created.id,
        amount: created.amount,
        agencyCode: created.agencyCode,
        tenantId: $Tenant
    });
}
```

> **Deduplication rule:** if you manually issue `api.post-request` with `$WebHooks.emit(...)`, the native kernel dispatch for `native: true` entries whose filter matches the current query is **automatically canceled** — no duplicate notification.

### 8.4 Client-side security and verification

Each outgoing call carries:

```
X-Payos-Event: atlas-payment-completed
X-Payos-Signature: sha256=<HMAC-SHA256(secret, raw-body)>
X-Tenant-Id: atlas
X-Correlation-Id: <uuid>
```

Communicate the **HMAC secret** to the customer (via a secure channel — never by clear email) so that its system validates `X-Payos-Signature` before any processing.

---

## 9. Capabilities — enable/disable optional modules

A **capability** is an application with `category: "capability"` than other applications extend with reference `extends`. It can be **installed and activated without redeploying the runtime** — ideal either for optional business modules that not all clients use (e.g. *Payment Links*, *Loyalty Points*, *Fraud Detection*), or for composing larger applications like pieces of LEGO.

Reference: [developer/capability system.md](../developer/capability%20system.md), [architecture/extensibility.md §1](../architecture/extensibility.md#1-capabilities-extending-applications), [cli-tools/cpm.md](../cli-tools/cpm.md).

### 9.1 Activate a capability for a given client (tenant)

```bash
# The capability is already installed in the bundle (delivered by the publisher or via catalog)
cpm --activate --id loyalty-points --app atlas-payment-gateway --tenant atlas \
    --bundle-path /opt/payos/bundle
```

- `--app atlas-payment-gateway` — limits activation to **your overlay** (others
  applications in the bundle do not see the capability).
- `--tenant atlas` — limits activation to the **client tenant** (multi-tenant: the others tenants sharing the same bundle are not affected).

### 9.2 Disable/uninstall

```bash
# Disable for this client only (reversible)
cpm --deactivate --id loyalty-points --app atlas-payment-gateway --tenant atlas \
    --bundle-path /opt/payos/bundle

# Uninstall completely (and remove its data schema if necessary)
cpm --uninstall --id loyalty-points --drop-schema --bundle-path /opt/payos/bundle
```

### 9.3 Reference the capability in your overlay

Once installed, it is added to `extends` of your overlay so that its resources (API, pages, hooks, menus) become visible **when it is active for the current tenant**:

```json
{
  "id": "atlas-payment-gateway",
  "extends": ["payment-gateway", "loyalty-points"],
  "authorized-tenants": ["atlas"]
}
```

> If `loyalty-points` is inactive for `atlas`, `ResourceLocator` **ignores entirely** its subtree — including its own `hooks/` and `menu/entries.json`. No action
> No additional is needed to "hide" the feature from clients who are not subscribed to it.

### 9.4 Where state and audit live

Under `{configDir}/.capabilities/`:

| File | Happy |
| --- | --- |
| `registry.json` | Capabilities installed in this bundle (id, version, status). |
| `activation.json` | Activation scope per capability/app/tenant. |
| `events.ndjson` | Audit log append-only (INSTALL/ACTIVATE/DEACTIVATE/UNINSTALL). |

Retain this information in your customer delivery documentation (see [§18.1](#181-delivery-checklist)).

---

## 10. Multi-tenancy — isolate and configure the customer environment

Each client corresponds to a PayOS **tenant**, identified by `X-Tenant-Id`. The multi-tenancy is **structural** (enforced by the architecture, not by your code) — your role is to **configure** it correctly.

### 10.1 Restrict the overlay to the client's tenant

```json
{
  "id": "atlas-payment-gateway",
  "authorized-tenants": ["atlas"]
}
```

Any request to `atlas-payment-gateway` with an `X-Tenant-Id` different from `atlas` is rejected by the tenant policy **before** the execution of the script.

### 10.2 Isolate customer data

Choose an insulation method adapted to the contract with the customer:

| Fashion | Description | When to use it |
| --- | --- | --- |
| `shared-schema` | All data in the same schema, separated by `tenant_id` column | SMB clients, low volume |
| `dedicated-schema` (`schema`) | Dedicated SQL schema in the same database | Average customers — good insulation/cost compromise |
| `dedicated-database` | Dedicated database | Tier 1/2 customers, strong regulatory requirements |

```json
{
  "multitenancy": {
    "default-isolation-mode": "shared-schema",
    "tenants": {
      "atlas": {
        "schema": "atlas",
        "isolationMode": "dedicated-schema",
        "quota": { "enabled": true, "requestsPerMinute": 1200 }
      }
    }
  }
}
```

For a **totally dedicated** database:

```json
{
  "multitenancy": {
    "tenants": {
      "atlas": {
        "isolationMode": "dedicated-database",
        "database-service": {
          "name": "atlas_db",
          "configuration": {
            "url": "jdbc:postgresql://atlas-db.example.com:5432/payos",
            "username": "atlas_user",
            "password": "${ATLAS_DB_PASSWORD}",
            "driver-class": "org.postgresql.Driver",
            "dialect": "org.hibernate.dialect.PostgreSQLDialect"
          }
        }
      }
    }
  }
}
```

### 10.3 Identity (OIDC) dedicated to the customer

If the customer has its own identity provider (Keycloak, Azure AD, etc.), override its
`security` block for this tenant — or directly for your overlay application:

```json
{
  "apps": [
    {
      "id": "atlas-payment-gateway",
      "security": {
        "provider": "nimbus",
        "clientId": "atlas-payos-client",
        "clientSecret": "${ATLAS_OIDC_SECRET}",
        "oidcProviderBaseUrl": "https://idp.atlas-bank.com",
        "realm": "atlas",
        "scope": "openid profile email",
        "sessionCookieSecure": true,
        "allowedOrigins": ["https://portal.atlas-bank.com"]
      }
    }
  ]
}
```

> Complete reference of `security` keys: [configuration/json-configuration-reference.md §3.2](../configuration/json-configuration-reference.md#bloc-security-dune-application) and [configuration/oidc-configuration-guide.md](../configuration/oidc-configuration-guide.md).

### 10.4 Quotas

Adapt the flow limits according to the customer's contract:

```json
{
  "multitenancy": {
    "default-tenant-quotas": { "enabled": true, "requestsPerMinute": 600 },
    "tenants": {
      "atlas": { "quota": { "enabled": true, "requestsPerMinute": 1200 } }
    }
  }
}
```

### 10.5 Verification

- All queries/logs carry `X-Tenant-Id: atlas` and `X-Correlation-Id` — check them in your tests (see [operations/observability.md](../operations/observability.md)).
- In production, `multitenancy.tenantSimulator.enabled` must be `false` and `requireTenantId` must be `true`.

---

## 11. Client-specific secrets

Each customer has their own secrets for third-party systems (PSP, ERP, anti-fraud, etc.). **Never** hardcode them into your scripts or configuration — use the service of secrets, scoped automatically by tenant. Reference: [developer/secrets-usage.md](../developer/secrets-usage.md), [configuration/secret-service.md](../configuration/secret-service.md), [operations/secrets-management.md](../operations/secrets-management.md).

### 11.1 Read a secret in a script

```javascript
//atlas-payment-gateway/api/payments/create.js
function execute(request, controlData) {
    var key = $Secrets.getSecret("atlas-psp-api-key");   // automatically resolve "atlas/atlas-psp-api-key"
    try {
        return callAtlasPsp(key.asString(), controlData);
    } finally {
        key.close();   // clear the value from memory
    }
}
```

`$Secrets` is injected only if `secret-service.enabled = true` in configuration and that a connector (`filesystem` or `vault`) is present in `connectors-dir` — check these two points before delivering.

### 11.2 Provisioning a secret for the client tenant (`spm`) with the filesystem secret provider

For filesystem secret provider, you should perform the following to create and manage a secret:

#### Create the secret with a name

```bash
#Provider filesystem
spm set --root /opt/payos/secrets --tenant atlas --connectors-dir /opt/payos/connectors \
        --name atlas-psp-api-key --value "sk_live_xxx"

# Check
spm list --root /opt/payos/secrets --tenant atlas
spm describe --root /opt/payos/secrets --tenant atlas --name atlas-psp-api-key
```
#### Configure the filesystem secret provider

Create the following configuration in the root folder of the bundle (you could either integrate it in bootstrap.json or create a separate json file for that purpose) :

```json
{
  "secret-service": {
    "configuration": {
      "enabled": true,
      "type": "filesystem",
      "root": "/opt/payos/bundles/client1/.secrets",
      "keyfile": "/opt/payos/bundles/client1/.secrets/master.key"
    }
  }
}
```

The `root` key is the folder where the filesystem secret provider stores the secrets, and the keyfile is the file where the secret provider master key resides. Filesystem secret provider is usually used in development, but if you want to use it in production, you should protect the `keyfile` with the usual OS access rights (linux / windows), so that only the super user or payos user (the user under which the runtime will execute) have the right to read it, since the master key inside the file is not encrypted (only base64 encoded).

#### Deploy the secret provider Jar in the connectors directory

Since the secret provider is a connector / adapter just as all other adapters (database-service, queue service, ...), the byte code (JAR) of the provider should be deployed in the configured connectors directory. The connectors directory should be configured as follows:

```json
{
  "connectors-dir": "/opt/payos/connectors"
}
```

### 11.3 Provisioning a secret for the client tenant with the Hashicorp Vault secret provider

For HashiCorp Vault, use the Vault CLI/API and AppRole or token configuration described in [operations/secrets-management.md](../operations/secrets-management.md) — PayOS Vault secret provider **reads** the secrets, it does not provision them through Vault. To use the Vault secret provider, you have to declare its configuration either in bootstrap.json or in a separate json file in the bundle configuration directory.

There are two ways to authenticate to vault: approle or access token. To configure access with an approle:

```json
{
  "secret-service": {
    "type": "vault",
    "address": "https://vault.example.com:8200",
    "role-id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "secret-id": "${file:/run/secrets/vault_secret_id}",
    "approle-mount": "approle", // URI path to approle configuration
    "kv-mount": "secret", // URI path to the secret storage configured in vault
    "timeout": 10
  }
}
```

To configure access with a token:

```json
{
  "secret-service": {
    "type": "vault",
    "address": "https://vault.example.com:8200",
    "token": "sv-xxxxxxx",
    "kv-mount": "secret", // URI path to the secret storage configured in vault
    "timeout": 10
  }
}
```

To read a secret from vault for tenant `atlas`, you should use vault API with the following URI:

```shell
GET /v1/secret/data/atlas/{keyname}
```

To create one:

```shell
POST /v1/secret/data/atlas/{keyname}
```
with body:

```json
{"data":{"<keyname>":"<base64-encoded-value>","type":"string"}}
```
### 11.4 Best practices

- **Naming**: prefix secret names by client/usage (`atlas-psp-api-key`, `atlas-erp-token`) to remain readable if several clients share a bundle.
- **Rotation**: a new call to `spm set` on the same name increments the version — none script change required.
- **Errors**: handle `SecretNotFoundException` / `SecretAccessDeniedException` properly (`$Errors.business(...)`) rather than letting the user report a raw 500.
- **Never in `webhooks.json` / `bootstrap.json` in plain text**: use placeholders `${VAR}` resolved by environment variables injected at container/service startup   (see [configuration/env-var-resolution.md](../configuration/env-var-resolution.md)), or encrypt the bundle with `edc` ([§15.3](#153-encrypt-edc-delivery-bundle)).

---

## 12. Localization and branding

### 12.1 Languages

1. Configure the supported locales in `config/i18n.json` of the overlay:

```json
//atlas-payment-gateway/config/i18n.json
{
  "defaultLocale": "fr",
  "fallbackLocale": "en",
  "supportedLocales": ["fr", "en", "ar-MA"],
  "missingKeyMode": "bracket",
  "headerName": "Accept-Language",
  "overrideHeaderName": "X-Locale"
}
```

2. Add/overload message files in `i18n/{locale}/*.json` — they are merged with those of `payment-gateway` (your keys win in case of conflict). See [§6.6] (#66-translations-i18n) and [developer/server-side-i18n-js-guide.md](../developer/server-side-i18n-js-guide.md).

### 12.2 Logos, colors, static assets

Place client brand assets in `files/` — accessible without scripting or additional configuration, with automatic HTTP cache:

```
atlas-payment-gateway/
└── files/
    ├── logo-atlas.png
    ├── favicon-atlas.ico
    └── styles-atlas.css
```

```
GET /atlas-payment-gateway/files/logo-atlas.png
GET /atlas-payment-gateway/files/styles-atlas.css
```

### 12.3 Home pages and personalized screens

The standard way in nuxt to include a global CSS file to apply the client's graphic charter: logo, colors, wording, layout is in `nuxt.config.ts` :

```javascript
export default defineNuxtConfig({
  css: [
    '~/files/css/main.css'
  ]
})
```
Then main.css is automatically loaded on every page. The inheritence principle is applicable here just like the API handlers or any other files. To override the UI entry point of the application (which is not recommended), you should override the Vue page pointed to by the `/home` route (this route is the standard route for the home page of the application).  

---

## 13. Advanced mechanisms (under coordination with the editor)

These mechanisms affect the **classpath and the deployment of the runtime** (JARs, directories plugins). They are rarely needed for a standard customization mission; they meet customer needs **specific to the protocol or third-party system**. Before you implement, **validate the approach with the editor** (impact on maintenance, security and runtime version upgrades).

### 13.1 Java Extensions (`extensions-dir`)

If the customer requires a specific Java library (e.g. **jPOS** for ISO 8583), it can be made accessible to scripts via `Java.type()`:

```javascript
function execute(request, controlData) {
    var ISOMsg = Java.type("org.jpos.iso.ISOMsg");
    var msg = new ISOMsg();
    msg.setMTI("0200");
    msg.set(2, controlData.pan);
    return { mti:msg.getMTI() };
}
```

- Either drop the JAR (and its dependencies) or a fat Jar (to be built in an empty project) in `extensions-dir`.
- No PayOS interface to implement — third-party library as is.
- `java.lang.System` and low-level I/O remain **blocked** by the sandbox no matter what.

Reference: [developer/java-extensions.md](../developer/java-extensions.md), [configuration/json-configuration-reference.md §3.13](../configuration/json-configuration-reference.md#313-extensions-dir-bootstrapjson).

### 13.2 SPI connectors (`connectors-dir`)

If the client imposes a **backend** not covered by the reference connectors (e.g. a proprietary secrets provider, a new `IQueueClientFactory` implementation), ... a connector can be developed. The already implemented services this way are servers (http, tcp and queue), the secret providers, the database service and the webhook dispatchers. The interfaces for these services reside in the payos kernel. Let's take an example from the secret provider interface :

1. Implement the SPI interface (e.g. `ISecretProviderFactory` + `ISecretProvider`) in a separate project.
2. Declare the service factory in `META-INF/services/services` folder (create a file that has the complete class name of the factory interface and declare the implementation class of the factory inside the file).

```
secret-service-acme/
└── src/
  └── main/
    └── resources/
      └── META-INF/
        └── services/
          └── ma.s2m.payos.queue.IQueueClientFactory
```
and the content of the `ma.s2m.payos.queue.IQueueClientFactory` file will be:

```
ma.s2m.payos.secret.filesystem.FileSystemSecretProviderFactory
```

3. Build the JAR and drop it into `connectors-dir` (always as a fat Jar).
4. Reference its `type` and its other meta-data in the configuration (`secret-service.type: "my-provider"`, etc.).

> - This adds an **artifact to maintain and evolve** with each kernel version, but the kernel does not depend on it
> - involve the publisher from the design stage.
> - If these services are not declared, or have not been deployed to `connectors-dir` the kernel starts anyway without the corresponding service

### 13.3 TCP plugins / new protocols

For historical channels (ISO 8583 over TCP, proprietary protocols), `payos-server-tcp` exposes an extension point per JARs (`tcp-handlers-dir`) implementing `TcpMessageDecoder` / `TcpMessageEncoder` / `TcpMessageHandler`. See [architecture/tcp server/plugin-development.md](../architecture/tcp%20server/plugin-development.md).

A whole new **transport protocol** (beyond http/tcp/queue) is done via `ServerProvider` (`ServiceLoader`) — this is a **platform** mechanism, to be reserved for the editor or developed under the supervision of the editor.

---

## 14. Extending your applications with capabilities

### 14.1 What is a capability?

A **capability** is a PayOS application declared with `"category": "capability"` in its bootstrap descriptor, and that has a capability lifecycle of installation, activation, deactivation, uninstallation. It packages a **cohesive, optional feature set** — APIs, pages, menus, hooks, translations, static files — that can be installed once in a bundle and selectively **activated per application and per tenant**, without modifying the runtime or redeploying the server.

Typical use cases:
- *Loyalty Points* module (optional for some bank clients, not others)
- *Payment Links* feature (activated only for e-commerce tenants)
- *Fraud Detection* add-on (premium tier only)
- A cross-product shared module reused by multiple applications (e.g. `notifications`, `audit-trail`)

From `ResourceLocator`'s perspective, a capability is an `extends` entry like any other application — with one key difference: **before traversing its resource subtree, the engine checks the activation state** for the current `(appId, tenantId)` pair. If inactive, the capability's entire subtree (APIs, pages, hooks, menus) is silently skipped.

### 14.2 Capability directory structure

A capability has the same layout as a regular application, plus a `manifest.json` at the root and an optional `hooks/lifecycle.js`:

```
loyalty-points/
├── manifest.json              # capability descriptor (required)
├── config/
│   ├── mappings.json          # API endpoint declarations
│   └── routes.json            # page route declarations
│   └── i18n.json              # Localization configuration
├── api/
│   └── loyalty/
│       ├── balance.js         # GET /loyalty/balance
│       └── redeem.js          # POST /loyalty/redeem
├── page/
│   └── loyalty/
│       └── dashboard.vue      # loyalty dashboard page
├── lib/
│   └── loyalty-rules.js       # shared business logic
├── hooks/
│   ├── lifecycle.js           # install/activate/deactivate/uninstall hooks
│   └── post-request.js        # transversal behavior (e.g. award points after payment)
├── menu/
│   └── entries.json           # navigation entries (shown only when capability is active)
├── i18n/
│   └── en/
│       └── loyalty.json       # translations
├── files/
│   └── loyalty-badge.png      # static assets
└── webhooks.json              # outbound notifications
```

### 14.3 Writing the `manifest.json`

`manifest.json` is the **only required file** in a capability package. It is read by `cpm` at install time.

```json
{
  "id": "loyalty-points",
  "name": "Loyalty Points",
  "version": "1.2.0",
  "description": "Manages loyalty point accrual and redemption for cardholders.",
  "dependencies": [
    { "id": "notifications", "version": "1.0.0" },
    { "id": "audit-trail" }
  ]
}
```

| Field | Required | Description |
| --- | --- | --- |
| `id` | ✅ | Unique identifier — matches the directory name and the `--id` argument of `cpm`. |
| `name` | — | Human-readable display name. |
| `version` | — | Semantic version string. Falls back to `0.0.0` if absent. |
| `description` | — | Short description for catalog or documentation listing. |
| `dependencies` | — | Other capabilities that must be installed first. `version` is optional per entry. |

> **`category` is injected automatically**: you do **not** declare `"category": "capability"` in `manifest.json` — `cpm` adds it to `bootstrap.json` when installing the capability.

### 14.4 API endpoints

Capabilities declare their endpoints in `config/mappings.json` and implement them under `api/`, exactly like a regular application:

```json
// loyalty-points/config/mappings.json
{
  "mappings": {
    "api": {
      "/loyalty/balance": {
        "GET": { "handler": "loyalty/balance", "roles": ["ROLE_USER"] }
      },
      "/loyalty/redeem": {
        "POST": { "handler": "loyalty/redeem", "roles": ["ROLE_USER"] }
      }
    }
  }
}
```

```javascript
// loyalty-points/api/loyalty/balance.js
function loadControlData(request) {
    return { userId: $Principal.get("sub") };
}

function execute(request, controlData) {
    var account = $DB.unique(
        "SELECT a FROM LoyaltyAccount a WHERE a.userId = :uid AND a.tenantId = :tid",
        $DB.newParams().put("uid", controlData.userId).put("tid", $Tenant)
    );
    $Response.setStatusCode(200);
    $Response.setJsonBody({
        userId: controlData.userId,
        points: account != null ? account.points : 0
    });
    return $Response;
}

function emitInsight(request, response, payload) { return null; }
```

When the capability is active, callers reach these endpoints via the **parent application**'s base path:

```
GET /atlas-payment-gateway/api/loyalty/balance
```

### 14.5 Pages

Declare pages in `config/routes.json` and create `.vue` components under `page/`:

```json
// loyalty-points/config/routes.json
{
  "routes": [
    { "path": "/loyalty/dashboard", "component": "loyalty/dashboard" }
  ]
}
```

```vue
<!-- loyalty-points/page/loyalty/dashboard.vue -->
<template>
  <div class="loyalty-dashboard">
    <h2>{{ $t('loyalty.dashboard.title') }}</h2>
    <p>{{ $t('loyalty.dashboard.balance', { points: balance }) }}</p>
  </div>
</template>

<script>
export default {
  name: 'LoyaltyDashboard',
  data() { return { balance: 0 }; },
  async created() {
    const appId = this.$route.query.appId || 'atlas-payment-gateway';
    const res = await this.$api.get(`/${appId}/api/loyalty/balance`);
    if (res.ok) this.balance = (await res.json()).points;
  }
};
</script>
```

### 14.6 Menu entries

Menu entries declared in `menu/entries.json` are **automatically prefixed with the capability id** when aggregated into the parent application's menu. They appear in navigation only when the capability is active for the current tenant:

```json
// loyalty-points/menu/entries.json
[
  { "id": "loyalty-dashboard", "label": "Loyalty Points", "page": "loyalty/dashboard", "icon": "star" },
  { "id": "loyalty-history",   "label": "Points History", "page": "loyalty/history",   "icon": "clock" }
]
```

The `GET /{appId}/menu` API returns these entries prefixed as `loyalty-points/loyalty-dashboard` and `loyalty-points/loyalty-history` when the capability is active.

### 14.7 Hooks

Hooks inside a capability participate in the `extends` hook chain **only when the capability is active** for the current scope. The resolution order follows child-first: `atlas-payment-gateway` → `loyalty-points` → `payment-gateway`.

```javascript
// loyalty-points/hooks/post-request.js
// Award loyalty points after a successful payment
if ($Request.getPath() === "/payments" && $Request.getMethod() === "POST"
        && $Response.getStatusCode() === 201) {
    var payment = $Response.getJsonBody();
    var userId = $Principal != null ? $Principal.get("sub") : null;
    if (payment && payment.id && userId) {
        var points = Math.floor(payment.amount / 10);
        $DB.save("LoyaltyTransaction", {
            userId:    userId,
            tenantId:  $Tenant,
            paymentId: payment.id,
            points:    points,
            type:      "ACCRUAL"
        });
    }
}
$HookChain.proceed();
```

### 14.8 Translations (i18n)

Add locale files under `i18n/{locale}/`. They are merged into the application's translation map when the capability is active:

```json
// loyalty-points/i18n/en/loyalty.json
{
  "loyalty": {
    "dashboard": {
      "title": "Your Loyalty Points",
      "balance": "Current balance: {points} pts"
    },
    "errors": {
      "notEnoughPoints": "Not enough points to redeem"
    }
  }
}
```

### 14.9 Lifecycle hooks (`hooks/lifecycle.js`)

Lifecycle hooks execute at install/activate/deactivate/uninstall time — **outside the HTTP request pipeline**. Use them for one-time setup or cleanup:

```javascript
// loyalty-points/hooks/lifecycle.js
var exports = {};

exports.install = function(ctx) {
    // ctx.capabilityId, ctx.version, ctx.configDir are always present
    // Typical use: write a flag file or generate a migration script
    // Note: $DB, $Secrets and other runtime bindings are NOT available here
};

exports.activate = function(ctx) {
    // ctx.capabilityId, ctx.configDir, and optionally ctx.appId, ctx.tenantId
    // Runs after the capability is added to the extends chain
};

exports.deactivate = function(ctx) {
    // Runs before the capability is removed from the extends chain
};

exports.uninstall = function(ctx) {
    // ctx.dropSchema is true when --drop-schema was passed to cpm
};
```

Context fields available per lifecycle hook:

| `ctx` field | Hooks | Notes |
| --- | --- | --- |
| `capabilityId` | all | Always present |
| `version` | install | Resolved version being installed |
| `configDir` | all | Absolute path of the bundle config directory |
| `appId` | activate, deactivate | Present only when scoped to a specific application |
| `tenantId` | activate, deactivate | Present only when scoped to a specific tenant |
| `dropSchema` | uninstall | `true` when `cpm --drop-schema` was passed |

> **Runtime bindings not available:** `$DB`, `$Secrets`, `$Queue` and all other script bindings are **not injected** in lifecycle hooks — they run in an isolated GraalVM context outside the HTTP pipeline. Interact with the filesystem (`java.nio.file.*`) or write SQL/shell scripts consumed by your deployment pipeline.

### 14.10 Declaring dependencies between capabilities

If your capability relies on another, declare it in `manifest.json`. `cpm` resolves and installs the full dependency chain automatically before installing your capability:

```json
{
  "id": "premium-loyalty",
  "version": "1.0.0",
  "dependencies": [
    { "id": "loyalty-points", "version": "1.2.0" },
    { "id": "notifications" }
  ]
}
```

Resolution order during `cpm --install`:
1. **Already installed** → skipped.
2. **Available in a configured product catalog** → pulled and installed automatically.
3. **Present as a sibling directory** at the same `--path` parent → installed from there.
4. **None of the above** → install aborts with an error **before any side-effect**.

### 14.11 Attaching a capability to an application

Once installed, `cpm` automatically adds the capability to the `extends` array of the target application. All resource resolution (APIs, pages, menus, hooks, translations) becomes active immediately — no restart needed.

**Install and activate in one step (recommended):**

```bash
cpm --install --path ./build/loyalty-points --bundle-path /opt/payos/bundle \
    --id loyalty-points --app atlas-payment-gateway --tenant atlas
```

**Separate install then activate (useful when deploying globally, then activating per client):**

```bash
# Install globally (no app/tenant scope)
cpm --install --path ./build/loyalty-points --bundle-path /opt/payos/bundle --id loyalty-points

# Activate later for a specific client
cpm --activate --id loyalty-points --app atlas-payment-gateway --tenant atlas \
    --bundle-path /opt/payos/bundle
```

The resulting `bootstrap.json` entry:

```json
{
  "id": "atlas-payment-gateway",
  "extends": ["payment-gateway", "loyalty-points"],
  "authorized-tenants": ["atlas"]
}
```

**Resource resolution when active:**

```
GET /atlas-payment-gateway/api/loyalty/balance  (tenant = atlas)

ResourceLocator:
1. atlas-payment-gateway/api/loyalty/balance.js  → not found
2. loyalty-points — activation check (atlas-payment-gateway, atlas) → ACTIVE
   loyalty-points/api/loyalty/balance.js          → FOUND ✓
```

**Resource resolution when inactive:**

```
GET /atlas-payment-gateway/api/loyalty/balance  (tenant = beta, capability not activated)

ResourceLocator:
1. atlas-payment-gateway/api/loyalty/balance.js  → not found
2. loyalty-points — activation check (atlas-payment-gateway, beta) → INACTIVE → skip entirely
3. payment-gateway/api/loyalty/balance.js         → not found
→ 404 Not Found
```

---

## 15. Packaging, delivery and operation at the customer's premises

### 15.1 Compose the final bundle with `cpm` / `ppm` / `apm`

In the case the intergator developed a whole new application or capability for the customer (besides the extension mechanism we talked about earlier), here is how to integrate it into the editor's product. 

#### Creating and installing a new capablity

Let's suppose the integrator developed a new capability, and names the capability `cap-currency-converter`.

1. Put the application/capability's main directory in a specific chosen path. Let's call it `${path}`.
2. Install the capability with `cpm` command

```bash
# Install/activate the required capabilities for this client for application atlas-payment-gateway
cpm --install --path ${path} --bundle-path /opt/payos/bundle --id cap-currency-converter --app atlas-payment-gateway

# Check all capabilities status
cpm --status --all
# or check just the specified capability status
cpm --status --id cap-currency-converter
```

#### Creating and installing a new application

Let's suppose the integrator developed a new application, whose name is `atlas-payment-gateway`

1. Put the application's main directory in a specific chosen path. Let's call it `${path}`.
2. Create a manifest file for the application and put it in the main application directory. The manifest file contains the same information as an application entry in `applications` key in bootstrap file `bootstrap.json`.
3. Install the application with `apm` command

```bash
# Install the application atlas-payment-gateway
apm --install --path ${path} --bundle-path /opt/payos/bundle --app atlas-payment-gateway

# Check all applications status
apm --status --all
# or check just the specified application status
apm --status --app atlas-payment-gateway
```

### 15.2 What applies hot vs. what requires a restart

| Change | Hot? |
| --- | --- |
| Your overlay files (`api/`, `page/`, `lib/`, `hook/`, `i18n/`, `menu/`, `webhooks.json`) | ✅ Yes (`ConfigWatcher` + hook cache invalidation) |
| `bootstrap.json` (application declaration, `multitenancy`, `security`, …) | ✅ Yes (atomic swap of configuration) |
| Enabling/disabling capability via `cpm` | ✅ Yes |
| New JAR in `connectors-dir` / `extensions-dir` | ❌ Reboot required |
| New listening port / new server | ❌ Reboot required |

See [operations/hot-reload.md](../operations/hot-reload.md).

### 15.3 Encrypt the delivery bundle (`edc`)

For compliant delivery (tamper-evident), package/encrypt the bundle before transfer:

```bash
edc --encryption pack --inputdir ./bundle --secret-provider filesystem \
    --root ./secrets --secret-tenant default --secret-name encryptionKey
```
If you are using vault as a secret provider (AppRole auth):

```bash
edc --encryption pack --inputdir ./bundle --secret-provider vault \
    --vault-address https://vault.example.com:8200 \
    --vault-auth-method approle \
    --vault-approle-mount approle \
    --vault-role-id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
    --vault-secret-id "${file:/run/secrets/vault_secret_id}" \
    --vault-kv-mount secret \
    --secret-tenant default --secret-name encryptionKey
```

Or with token auth:

```bash
edc --encryption pack --inputdir ./bundle --secret-provider vault \
    --vault-address https://vault.example.com:8200 \
    --vault-auth-method token \
    --vault-token sv-xxxxxxx \
    --vault-kv-mount secret \
    --secret-tenant default --secret-name encryptionKey
```

See [operations/bundle-encryption.md](../operations/bundle-encryption.md) and [cli-tools/edc.md](../cli-tools/edc.md).

### 15.4 Environments

Reproduce the same bundle (overlay + config) across environments, making no changes, but
vary the following :

- the **secrets** (`spm` / Vault — different per environment),
- the **URLs** of webhooks/SI client (environment `${VAR}`),
- `multitenancy.tenantSimulator` (`true` only in dev/local, never elsewhere),
- the `database-service.configuration` (dedicated instances per environment).
- the OIDC server provider

```
dev (local) → recipe (client sandbox) → production (real client tenant)
```

---

## 16. Best practices, governance and security

### 16.1 Personalization discipline

- **One overlay per pair (client, basic product)** — do not share several clients in the same overlay (an overlay is by definition specific to a client).
- **Minimum surface area**: only overload what really needs to change; prefer hooks to duplications of entire scripts.
- **Traceability**: keep a record (in your repository) of overloaded resources and version of the base application against which they were validated. With each rise of version delivered by the publisher, re-run your tests on overloaded resources.

### 16.2 Security and compliance

- No sensitive data (API key, password, HMAC secret) in plain text in the code or versioned configuration — always `${VAR}` + secrets ([§11](#11-client-specific-secrets)).
- Native webhooks payloads **never** contain a request/response body (PCI-DSS constraint); if a business event must carry sensitive data, do so consciously via `$WebHooks.emit(...)` in a hook, selecting only the necessary fields.
- `sessionCookieSecure: true` and HTTPS for any exposure to the client.
- `multitenancy.tenantSimulator.enabled = false` and `requireTenantId = true` outside of   development position.

### 16.3 Observability

- Each query/log/error has `X-Tenant-Id` and `X-Correlation-Id` — use them in your hooks (`$Logger.info("... {} {}", $Tenant, correlationId)`) so that the incidents side customer are traceable from end to end. See [operations/observability.md](../operations/observability.md).
- `$HookChain.getChain()` allows, in debug, to visualize the effective hook chain for a given point — useful for diagnosing an override that is not triggering as expected.

### 16.4 Tests before production

1. **Local**, with the `tenantSimulator` activated on the client tenant.
2. Test **each overloaded resource** (the URL should serve your version, not that of
   the base app) — `curl -H "X-Tenant-Id: atlas" ...`.
3. Test **non-overloaded legacy paths** to verify they still work
   through your `extends`.
4. Trigger **hooks** (success, error) and check `$HookChain.proceed()`/`stop()`.
5. Verify **delivery of webhooks** to a test endpoint (valid HMAC signature).
6. Validate **tenant isolation**: a query with a different `X-Tenant-Id` must be
   rejected by your overlay (`authorized-tenants`).

---

## 17. End-to-end case study

**Context:** the **Atlas** bank has subscribed to the editor's `payment-gateway` product. The contract provides: a mandatory “agency code” field, Atlas branding, activation of the capability *Loyalty Points*, a dedicated database, its own IdP, notifications to Atlas core banking, and an interface in Moroccan Arabic.

| # | Customer need | Mechanism | Section |
| --- | --- | --- | --- |
| 1 | Create the personalization space | Overlay `atlas-payment-gateway` which `extends: ["payment-gateway"]` | [§5](#5-set-up-the-personalization-application--overlay--client) |
| 2 | Make `agencyCode` mandatory when creating a payment | `hooks/pre-request.js` with `$HookChain.stop()` if absent | [§7.3](#73-recipe--client-specific-pre-request-validation) |
| 3 | Home page in Atlas colors | Change css in nuxt config + change assets in `files/` | [§6.3](#63-pages-components-and-branding), [§12.2](#122-logos-colors-static-assets) |
| 4 | Activate loyalty program | `cpm --activate --id loyalty-points --app atlas-payment-gateway --tenant atlas` | [§9](#9-capabilities--activate/deactivate-optional-modules) |
| 5 | Dedicated database and schema | `multitenancy.tenants.atlas` with `isolationMode: dedicated-database` | [§10.2](#102-isolate-client-data) |
| 6 | IdP Keycloak specific to Atlas | `security` dedicated on `atlas-payment-gateway` | [§10.3](#103-oidc-identity-dedicated-to-client) |
| 7 | Notify core banking of each completed payment | `webhooks.json` (`atlas-payment-completed`, `native: true`) | [§8.1](#81-declare-a-subscription-in-webhooksjson) |
| 8 | Interface in `ar-MA` | `config/i18n.json` + `i18n/ar-MA/*.json` | [§12.1](#121-languages) |
| 9 | Atlas PSP API Key | `$Secrets` + `spm set --tenant atlas --name atlas-psp-api-key` | [§11](#11-client-specific-secrets) |
| 10 | Secure delivery at Atlas | `apm --install` + `edc --encryption pack` | [§15](#15-packaging-delivery-and-operation-at-the-customers-premises) |

Each line corresponds to an **additive** mechanism, deployed in the overlay `atlas-payment-gateway` or in configuration — **no `payment-gateway` file is modified**. In the next version of `payment-gateway` delivered by the publisher, it is enough to replace the `apps/payment-gateway/` folder and revalidate the overloaded resources listed in the personalization register (point 16.1).

---

## 18. Appendices

### 18.1 Delivery checklist

- [ ] Overlay `<client>-<app>` created, `extends` correctly declared, **no file modified in   the base app**.
- [ ] `authorized-tenants` restricted to the client's tenant(s).
- [ ] `multitenancy.tenants.<client>` configured (isolation, schema/DB, security, quotas).
- [ ] `tenantSimulator.enabled = false`, `requireTenantId = true`.
- [ ] Hooks tested (success, error, `$HookChain` delegation).
- [ ] `webhooks.json`: URLs/secrets via `${VAR}`, HMAC signatures verified on the client side.
- [ ] Required capabilities installed and enabled for the correct `app`/`tenant`.
- [ ] Provisioned secrets (`spm`/Vault) for the client tenant — nothing clear in the       versioned code/config.
- [ ] i18n: locales added, `supportedLocales` updated, branding (`files/`) in place.
- [ ] Bundle packaged/encrypted (`edc`) for delivery if required by contract.
- [ ] Updated customization register (overloaded resources ↔ version of the base app).

### 18.2 Express glossary

| Term | Definition |
| --- | --- |
| **Basic app** | Application/product delivered by the publisher, never modified directly. |
| **Overlay** | Application created by the integrator, which `extends` the base application and carries all the customization. |
| **Capability** | Optional functional module, installable/activatable without redeployment, referenced via `extends`. |
| **Holding** | The client, identified by `X-Tenant-Id`, isolated by the architecture. |
| **Hook** | `hooks/*.js` script intercepting a point in the pipeline (pre/post-request, error, page). |
| **Webhook** | Asynchronous, signed outbound HTTP notification to a client system. |

Full glossary: ​​[overview/glossary.md](../overview/glossary.md).

### 18.3 To go further

| Subject | Document |
| --- | --- |
| Complete Application Template | [developer/application-model.md](../developer/application-model.md) |
| Create an app from scratch | [developer/create-application-guide.md](../developer/create-application-guide.md) |
| Script Bindings Reference | [developer/scripting-bindings.md](../developer/scripting-bindings.md) |
| API Endpoints JS Guide (`$Errors`, `$I18n`, …) | [developer/javascript-api-endpoint-guide.md](../developer/javascript-api-endpoint-guide.md) |
| All extensibility mechanisms | [architecture/extensibility.md](../architecture/extensibility.md) |
| Hooks & webhooks — full template | [architecture/hooks-webhooks.md](../architecture/hooks-webhooks.md) |
| Multi-tenancy — architecture | [architecture/multi-tenancy.md](../architecture/multi-tenancy.md) |
| Full JSON reference | [configuration/json-configuration-reference.md](../configuration/json-configuration-reference.md) |
| All CLI tools | [cli-tools/README.md](../cli-tools/README.md) |
| Deployment & operation | [operations/README.md](../operations/README.md) |