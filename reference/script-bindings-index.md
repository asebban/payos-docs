# Script bindings index

Quick reference for the `$` bindings injected into API scripts. Full explanations are in
[developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Always available

| Binding | Type | Purpose |
| --- | --- | --- |
| `$Request` | `Request` | Body, headers, parameters, method, path, context data. |
| `$Response` | `Response` | Set status, headers, body. |
| `$Api` | `ApiProxy` | Call local or remote APIs (`get()`, `post()`, `put()`, `delete()`). |
| `$App` | `Application` | Current application (id, name, version, config). |
| `$Principal` / `$User` | Map | Authenticated principal: `id`, `email`, `name`, `preferred_username`, `roles`. |
| `$Tenant` | String | Resolved tenant id. |
| `$Library` | loader | Load shared JS from `lib/`. |
| `$Logger` | SLF4J logger | Plain log line, message + level only. |
| `$Errors` | `ErrorsProxy` | Typed business errors (`badRequest`, `notFound`, `conflict`, ...). |
| `$Analytics` | `AnalyticsBinding` (wraps the `AnalyticsRecorder` static facade; `logEvent`; best-effort — errors logged, never thrown) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |
| `$Metrics` | `MetricsBinding` (wraps `MetricsRecorder`; `record`; best-effort) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |
| `$Integration` | `IntegrationEventBinding` (wraps `IntegrationEventPublisher`; `publish`; best-effort) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |
| `$EventStore` | `EventStoreBinding` (wraps `EventStore`; `append`/`replay`; **propagates errors**, does not swallow them — unlike the other three) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |
| `$Audit` | `AuditBinding` (wraps `AuditLogger`; **only** the free-form `logEvent(eventType, result, extra)` — no typed lifecycle methods; **propagates errors**) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |
| `$Diagnostics` | `DiagnosticsBinding` (wraps `Diagnostics`; generic `logEvent(nature, stage, ...)`; best-effort) | [scripting-bindings.md](../developer/scripting-bindings.md), [event-category-payload-contracts-v7-2026-07-28.md](../developer/event-category-payload-contracts-v7-2026-07-28.md) |

## Available when the service is configured

| Binding | Service | Type | Doc |
| --- | --- | --- | --- |
| `$DB` | database-service | `IDatabaseService` | [data-access.md](../developer/data-access.md) |
| `$Queue` | queue-service | `QueueBinding` (wraps `IQueueClient`; exposes `publish`/`isConnected` only, no `subscribe`) | [queue-messaging.md](../developer/queue-messaging.md) |
| `$Secrets` | secret-service | `SecretsBinding` (wraps `ISecretProvider`; exposes `get`/`list`/`tokenize`/`detokenize` only) | [secrets-usage.md](../developer/secrets-usage.md) |
| `$Cache` | cache-service | `CacheBinding` (wraps `ICacheStore`; exposes `put`/`get`/`remove`/`exists`/`increment`, auto-scoped by tenant) | [cache-usage.md](../developer/cache-usage.md), [cache-service.md](../configuration/cache-service.md) |
| `$SlidingWindow` | sliding-window-service | `SlidingWindowBinding` (wraps `ISlidingWindowCounter`; exposes `count` only — read-only, auto-scoped by tenant) | [sliding-window-usage.md](../developer/sliding-window-usage.md), [sliding-window-service.md](../configuration/sliding-window-service.md) |
| `$I18n` | i18n | i18n accessor | [internationalization.md](../developer/internationalization.md) |
| `$WebHooks` | webhooks | `WebhookHooksProxy` | [webhooks-and-hooks.md](../developer/webhooks-and-hooks.md) |
| `$Connector` | connector framework | `ConnectorBinding` (gated on `PayOSConfig.getConnectorRegistry()`; **wired into `BootServer` since 2026-07-27** via `ConnectorFrameworkInitializer` — absent only when no connectors are configured/found) | [scripting-bindings.md](../developer/scripting-bindings.md), [connector-framework-parameters-v3-2026-08-11.md](../configuration/connector-framework-parameters-v3-2026-08-11.md) |
| `$Notification` | notification-service | `NotificationBinding` (gated on `PayOSConfig.getNotificationServiceFactory()`) | [scripting-bindings.md](../developer/scripting-bindings.md), [notification-service.md](../configuration/notification-service.md) |

## Java interop

| Mechanism | Notes |
| --- | --- |
| `Java.type("...")` | Allowed except a denylist (e.g. `java.lang.System`); reaches whitelisted extension classes when an extensions dir is configured. See [java-extensions.md](../developer/java-extensions.md). |

## Injection

Bindings are injected by `ApiResourceHandler` via `putMember(name, value)` immediately before
script execution, after the tenant scope is opened and security has run. Optional bindings are
injected only when their service is present. See
[architecture/request-processing.md](../architecture/request-processing.md).
