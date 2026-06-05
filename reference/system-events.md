# System events index

The system events PayOS fires, each of which can trigger in-process [hook scripts](../developer/webhooks-and-hooks.md)
and outbound [native webhooks](../configuration/webhook-service.md). Defined in
`IConfigSpec.Webhooks.SystemEvents`. The model is in
[architecture/eventing-webhooks.md](../architecture/eventing-webhooks.md).

## API events

| Event | Fired when | Hook point |
| --- | --- | --- |
| `api.pre-request` | Before an API script executes. | `API_PRE_SCRIPT` |
| `api.post-request` | After an API script succeeds. | `API_POST_SCRIPT` |
| `api.on-error` | When an API script throws. | `API_ON_ERROR` |

## Security events

| Event | Fired when |
| --- | --- |
| `security.login` | On successful authentication. |
| `security.logout` | On logout. |

## Capability lifecycle events

| Event | Fired when |
| --- | --- |
| `capability.installed` | A capability is installed. |
| `capability.uninstalled` | A capability is uninstalled. |
| `capability.activated` | A capability is activated. |
| `capability.deactivated` | A capability is deactivated. |

## Page events

| Event | Fired when |
| --- | --- |
| `page.pre-serve` | Before a page is served. |
| `page.post-serve` | After a page is served. |
| `page.on-error` | When page rendering fails. |

## Subscribing

Declare native webhook subscriptions per application in `config/webhooks.json`, referencing
the event name. See [developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).
