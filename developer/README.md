# Developer guide

For developers **building applications on PayOS**. Applications are written primarily as JavaScript resources that run inside the [scripting sandbox](../architecture/scripting-engine.md) and interact with the platform through `$`-prefixed bindings.

## Reading order

0. [PayOS Build Guide](./build-guide.md) — Detailed build guide
1. [Getting started](getting-started.md) — build PayOS, run it locally, create your first app.
2. [Application model](application-model.md) — apps, capabilities, products, directory layout, `extends`.
3. [Writing API scripts](writing-apis.md) — the `loadControlData` / `execute` / `emitInsight` contract.
4. [Scripting bindings reference](scripting-bindings.md) — every `$` binding and its use.
5. [Data access](data-access.md) — using `$DB`.
6. [Secrets usage](secrets-usage.md) — using `$Secrets`.
7. [Queue messaging](queue-messaging.md) — using `$Queue`.
8. [Internationalization](internationalization.md) — using `$I18n` and the i18n config.
9. [Webhooks & hooks](webhooks-and-hooks.md) — subscribing to events, `$WebHooks`.
10. [Java extensions](java-extensions.md) — calling Java libraries via `Java.type()`.
11. [API documentation](api-documentation.md) — `@payos.openapi` annotations and `pdoc`. For the full reference, see [PayOS OpenAPI documentation guide](./openapi-docs/openapi-documentation-guide.md).
12. [Debugging](debugging.md) — debugging server-side JavaScript.
13. [Event category payload contracts (v1, 2026-07-10)](./event-category-payload-contracts-v1-2026-07-10.md) - one abstraction per event category (audit, analytics, event-sourcing, metrics, integration); supersedes the older [single-envelope observability proposal](./observability-event-contract-proposal.md)

## Related references

- [Configuration reference](../configuration/README.md) — every configuration key.
- [Reference: script bindings index](../reference/script-bindings-index.md) — quick lookup.
- [Architecture: request processing](../architecture/request-processing.md) — what runs your script.
- [Create an application guide](./create-application-guide.md) — Comprehensive guide for creating PayOS applications step by step

## The shortest possible application

An application is a directory of resources registered in the runtime. The smallest useful piece is a single API script:

```javascript
// apps/hello/api/greet.js
function loadControlData(request) {
    return {};                       // no extra control data needed
}

function execute(request, controlData) {
    $Response.setStatusCode(200);
    return { message: "Hello from PayOS" };
}

function emitInsight(request, response, payload) {
    return null;
}
```

Reachable (once the app `hello` is registered and served over HTTP) at:

```
POST /hello/api/greet
```

The next pages explain each part: the [application model](application-model.md), the
[script contract](writing-apis.md), and the [bindings](scripting-bindings.md) you used
(`$Response`).
