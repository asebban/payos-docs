# Developer guide

For developers **building applications on PayOS**. Applications are written primarily as JavaScript resources that run inside the [scripting sandbox](../architecture/scripting-engine.md) and interact with the platform through `$`-prefixed bindings.

## Reading order

0. [PayOS Build Guide](./build-guide.md) — Detailed build guide
1. [Getting started](getting-started.md) — build PayOS, run it locally, create your first app.
2. [Application model](application-model.md) — apps, capabilities, products, directory layout, `extends`.
3. [Capability system](./capability%20system.md) — capabilities as self-contained functional packages, managed by `cpm`, extending applications at runtime without redeployment.
4. [Writing API scripts](writing-apis.md) — the `loadControlData` / `execute` / `emitInsight` contract.
5. [Guide JavaScript des Endpoints API (FR)](javascript-api-endpoint-guide.md) — practical, detailed companion to "Writing API scripts", in French.
6. [Scripting bindings reference](scripting-bindings.md) — every `$` binding and its use.
7. [Data access](data-access.md) — using `$DB`.
8. [Secrets usage](secrets-usage.md) — using `$Secrets`.
9. [Vault secret-id secure injection](vault-secret-id-secure-injection.md) — securely injecting the Vault AppRole `secret-id` used by `$Secrets`.
10. [Cache usage](cache-usage.md) — using `$Cache` (distributed `memory`/`redis` cache, shared across instances/bundles).
11. [Sliding window counter usage](sliding-window-usage.md) — using `$SlidingWindow` (read-only exact sliding-window quota/rate-limit counter).
12. [Tenant quota enforcement](tenant-quota-enforcement.md) — how the platform's own per-tenant `requestsPerMinute` quota is counted, and why it's a different counter from `$SlidingWindow`.
13. [Queue messaging](queue-messaging.md) — using `$Queue`.
14. [Queue setup guide (de A à Z)](queue-setup-guide.md) — configuring and using both the publisher (`$Queue`) and consumer (`queue` transport) sides end to end.
15. [Notification service guide (A à Z)](notification-service-guide.md) — configuring, starting, and using `$Notification` end to end.
16. [Connector framework usage](connector-framework-usage.md) — calling `$Connector` from a script (business/payment connectors); not yet reachable in a running deployment, see the doc's caveat.
17. [Internationalization](internationalization.md) — using `$I18n` and the i18n config.
18. [Server-side i18n for JavaScript APIs](server-side-i18n-js-guide.md) — using `$I18n` from inside API handlers and hooks to return locale-translated messages.
19. [Webhooks & hooks](webhooks-and-hooks.md) — subscribing to events, `$WebHooks`.
20. [Java extensions](java-extensions.md) — calling Java libraries via `Java.type()`.
21. [API documentation](api-documentation.md) — `@payos.openapi` annotations and `pdoc`. For the full reference, see [PayOS OpenAPI documentation guide](./openapi-docs/openapi-documentation-guide.md).
22. [API contracts](api-contracts.md) — the complete HTTP API surface of the PayOS HTTP transport layer: endpoints, headers, status codes, CORS, security flow, and error contracts.
23. [Swagger UI configuration](swagger.md) — enabling the built-in Swagger UI and the companion `/openapi.yaml` endpoint.
24. [pdoc: install & CLI reference](./openapi-docs/install-pdoc-command.md) — the standalone PayOS OpenAPI documentation CLI used to generate OpenAPI artifacts without starting PayOS Runtime.
25. [Debugging](debugging.md) — debugging server-side JavaScript.
26. [Debug backend JavaScript in VS Code (Windows)](debug-backend-javascript-vscode.md) — exposing GraalVM Polyglot backend JavaScript over the Chrome DevTools Protocol and debugging it from VS Code.
27. [Debug backend JavaScript in VS Code (Linux)](debug-backend-javascript-vscode-linux.md) — the same verified debugging flow, using Bash commands and Unix-style paths.
28. [Event category payload contracts (v7, 2026-07-28)](./event-category-payload-contracts-v7-2026-07-28.md) - one abstraction per event category (audit, analytics, event-sourcing, metrics, integration, diagnostics), all six implemented and all six SPI-resolved; supersedes the older [single-envelope observability proposal](./observability-event-contract-proposal.md)
29. [Implementing a custom licence validator](license-validation.md) — plugging a custom license validator into PayOS.

## Related references

- [Configuration reference](../configuration/README.md) — every configuration key.
- [Reference: script bindings index](../reference/script-bindings-index.md) — quick lookup.
- [Architecture: request processing](../architecture/request-processing.md) — what runs your script.
- [Create an application guide](./create-application-guide.md) — Comprehensive guide for creating PayOS applications step by step
- [Runtime compatibility checks](./runtime-compatibility-checks-v1-2026-08-06.md) — `minRuntimeVersion`/`maxRuntimeVersion` declaration, `apm`/`cpm`/`ppm` install-time enforcement, and `BootServer`'s boot-time audit.
- [Session store design: distributed sessions (Redis)](./session-store-redis-design.md) — design doc for the `payos` core session-store abstraction and its `session-service-redis` implementation, replacing in-memory OIDC sessions not shared across nodes.

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
