# Script bindings index

Quick reference for the `$` bindings injected into API scripts. Full explanations are in
[developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Always available

| Binding | Type | Purpose |
| --- | --- | --- |
| `$Request` | `Request` | Body, headers, parameters, method, path, context data. |
| `$Response` | `Response` | Set status, headers, body. |
| `$Api` | API metadata | The API resource being executed. |
| `$App` | `Application` | Current application (id, name, version, config). |
| `$Principal` / `$User` | Map | Authenticated principal: `id`, `email`, `name`, `preferred_username`, `roles`. |
| `$Tenant` | String | Resolved tenant id. |
| `$Library` | loader | Load shared JS from `lib/`. |

## Available when the service is configured

| Binding | Service | Type | Doc |
| --- | --- | --- | --- |
| `$DB` | database-service | `IDatabaseService` | [data-access.md](../developer/data-access.md) |
| `$Queue` | queue-service | `IQueueClient` | [queue-messaging.md](../developer/queue-messaging.md) |
| `$Secrets` | secret-service | `ISecretProvider` | [secrets-usage.md](../developer/secrets-usage.md) |
| `$I18n` | i18n | i18n accessor | [internationalization.md](../developer/internationalization.md) |
| `$WebHooks` | webhooks | `WebhookHooksProxy` | [webhooks-and-hooks.md](../developer/webhooks-and-hooks.md) |

## Java interop

| Mechanism | Notes |
| --- | --- |
| `Java.type("...")` | Allowed except a denylist (e.g. `java.lang.System`); reaches whitelisted extension classes when an extensions dir is configured. See [java-extensions.md](../developer/java-extensions.md). |

## Injection

Bindings are injected by `ApiResourceHandler` via `putMember(name, value)` immediately before
script execution, after the tenant scope is opened and security has run. Optional bindings are
injected only when their service is present. See
[architecture/request-processing.md](../architecture/request-processing.md).
