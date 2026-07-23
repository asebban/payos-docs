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
7. [Cache usage](cache-usage.md) — using `$Cache` (distributed `memory`/`redis` cache, shared across instances/bundles).
8. [Sliding window counter usage](sliding-window-usage.md) — using `$SlidingWindow` (read-only exact sliding-window quota/rate-limit counter).
9. [Tenant quota enforcement](tenant-quota-enforcement.md) — how the platform's own per-tenant `requestsPerMinute` quota is counted, and why it's a different counter from `$SlidingWindow`.
10. [Queue messaging](queue-messaging.md) — using `$Queue`.
11. [Queue setup guide (de A à Z)](queue-setup-guide.md) — configuring and using both the publisher (`$Queue`) and consumer (`queue` transport) sides end to end.
12. [Notification service guide (A à Z)](notification-service-guide.md) — configuring, starting, and using `$Notification` end to end.
13. [Connector framework usage](connector-framework-usage.md) — calling `$Connector` from a script (business/payment connectors); not yet reachable in a running deployment, see the doc's caveat.
14. [Internationalization](internationalization.md) — using `$I18n` and the i18n config.
15. [Webhooks & hooks](webhooks-and-hooks.md) — subscribing to events, `$WebHooks`.
16. [Java extensions](java-extensions.md) — calling Java libraries via `Java.type()`.
17. [API documentation](api-documentation.md) — `@payos.openapi` annotations and `pdoc`. For the full reference, see [PayOS OpenAPI documentation guide](./openapi-docs/openapi-documentation-guide.md).
18. [Debugging](debugging.md) — debugging server-side JavaScript.
19. [Event category payload contracts (v5, 2026-07-15)](./event-category-payload-contracts-v5-2026-07-15.md) - one abstraction per event category (audit, analytics, event-sourcing, metrics, integration, diagnostics); supersedes the older [single-envelope observability proposal](./observability-event-contract-proposal.md)

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
