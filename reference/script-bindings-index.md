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

## Available when the service is configured

| Binding | Service | Type | Doc |
| --- | --- | --- | --- |
| `$DB` | database-service | `IDatabaseService` | [data-access.md](../developer/data-access.md) |
| `$Queue` | queue-service | `IQueueClient` | [queue-messaging.md](../developer/queue-messaging.md) |
| `$Secrets` | secret-service | `SecretsBinding` (wraps `ISecretProvider`; exposes `get`/`list`/`tokenize`/`detokenize` only) | [secrets-usage.md](../developer/secrets-usage.md) |
| `$I18n` | i18n | i18n accessor | [internationalization.md](../developer/internationalization.md) |
| `$WebHooks` | webhooks | `WebhookHooksProxy` | [webhooks-and-hooks.md](../developer/webhooks-and-hooks.md) |
| `$Connector` | connector framework | `ConnectorBinding` (gated on `PayOSConfig.getConnectorRegistry()`; **not yet wired into `BootServer`**, so absent in a running deployment today) | [scripting-bindings.md](../developer/scripting-bindings.md), [connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md) |
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
