Created: 2026-09-09
Last updated: 2026-09-09
Version: 1

# Metrics Catalog (Micrometer / Prometheus)

This is the operational reference for every metric PayOS currently publishes through Micrometer, verified against the code on 2026-09-09 rather than derived from design intent. For the runtime design (contract layer, facade, recorder resolution, Prometheus mapping rules, tag policy) see [architecture/observability-metrics-architecture-v1-2026-09-06.md](../architecture/observability-metrics-architecture-v1-2026-09-06.md); this page only lists what exists, where it comes from, and what an operator needs to know to scrape, dashboard, and alert on it correctly.

## Scope and freshness

Every metric below is grounded in a real call site (`PayOSMetrics.increment/gauge/duration/ratio`) or, for the `$Metrics` script binding, direct `MetricSample` construction; no module was found calling Micrometer directly, so the `PayOSMetrics`/`MetricsRecorder` facade is a complete inventory boundary as of this date. All modules under `payos-parent` (workspace root `PayOS/`) were checked; the unrelated `PayOSV2/repositories/vue-app` frontend was excluded.

## How metrics are exposed

`GET /metrics` on `payos-server-http` returns Prometheus text format (`text/plain; version=0.0.4; charset=UTF-8`). Behavior is controlled by `PayOSConfig.settings.metrics`:

| Key | Default | Effect |
| --- | --- | --- |
| `enabled` | `true` | `false` returns `404 Not Found`. |
| `local-only` | `true` | A non-local client gets `403 Forbidden` and increments `payos.security.authz.denials` with `resource_type=metrics`. Set to `false` only if the endpoint is protected at the deployment boundary — the runtime does not add auth of its own. |

Every metric name is emitted internally without the `payos.` prefix and gets it added by `MicrometerMetricsRecorder`; this page lists names with the `payos.` prefix, as they appear in Prometheus and in dashboard/alert expressions. Tenant labels (`tenant_id`) are opt-in via `-Dpayos.metrics.tenant-tags-enabled=true` and are omitted from the tag lists below since they are identical everywhere.

A typical scrape config:

```yaml
scrape_configs:
  - job_name: payos
    metrics_path: /metrics
    static_configs:
      - targets: ["localhost:8080"]
```

## JVM and process metrics

Bound automatically at startup, independent of PayOS-specific instrumentation: `ClassLoaderMetrics`, `JvmMemoryMetrics`, `JvmGcMetrics`, `JvmThreadMetrics`, `ProcessorMetrics`, `UptimeMetrics`, `LogbackMetrics`. These use standard Micrometer meter names (`jvm.memory.used`, `jvm.gc.pause`, `process.uptime`, `logback.events`, etc.), not the `payos.*` namespace. A binder failure is logged at debug and does not block startup, so a missing JVM metric family is a silent gap worth checking for after upgrades.

## PayOS metric inventory

Grouped by module and functional area. `outcome` and `exception` are the two most common tags across the codebase; `exception` is a simple class name or `none`, `outcome` is one of a small fixed vocabulary per area (never a raw value).

### Runtime and configuration — `payos-kernel` (`BootServer`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.runtime.info` | Gauge | `version` | Static value 1, tagged with the running version — used as an `up`-style presence gauge. |
| `payos.runtime.startup.duration` | Timer | `outcome` | Only ever recorded with `outcome=success` — a failed boot exits the process before this line, so there is no failed-startup-duration series; use `runtime.startup.failures` for failure visibility instead. |
| `payos.runtime.startup.failures` | Counter | `stage`, `exception` | `stage` ∈ `config`, `license`, `runtime_compatibility`, `database`, `queue`, `secret`, `notification`, `connector_idempotency`, `connector_execution_state`, `cache`, `sliding_window`, `servers`. |
| `payos.runtime.servers.started` | Counter | `protocol`, `outcome`[, `exception`] | `exception` only present on the failure branch. |
| `payos.runtime.server.start.duration` | Timer | `protocol`, `outcome` | Only recorded on success — a start failure only increments the counter above, no duration sample. |
| `payos.runtime.service.initializations` | Counter | `service`, `outcome`, `exception` | `service` ∈ `database`, `queue`, `secret`, `notification`, `connector_idempotency`, `connector_execution_state`, `cache`, `sliding_window`. |
| `payos.runtime.service.initialization.duration` | Timer | `service`, `outcome`, `exception` | |
| `payos.config.reloads` | Counter | `outcome`, `exception` | |
| `payos.config.reload.duration` | Timer | `outcome`, `exception` | |
| `payos.config.reload.failures` | Counter | `outcome`, `exception` | Fires only when `outcome=error`; duplicates `config.reloads` on that path. |

### Transport (common pipeline) — `payos-kernel` (`Server`, shared by HTTP/TCP/queue)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.transport.active.requests` | Gauge | `server`, `app_id`, `method`, `resource_type` | In-flight requests at any instant. |
| `payos.transport.requests` | Counter | `server`, `app_id`, `method`, `resource_type`, `outcome`, `status_family` | Completed requests. |
| `payos.transport.request.duration` | Timer | same as above | End-to-end pipeline duration. |
| `payos.transport.errors` | Counter | `server`, `app_id`, `method`, `resource_type`, `outcome=error`, `exception` | Exceptions escaping the common pipeline. |

### API, security, script, resource — `payos-kernel` (`ApiResourceHandler`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.resource.not_found` | Counter | `app_id`, `resource_type`, `method`, `outcome=not_found`, `exception=ResourceException` | Missing application resource lookups. |
| `payos.security.auth.attempts` | Counter | `app_id`, `resource_type`, `method`, `outcome` (`success`/`failure`), `exception=none` | `exception` is always `none` here, even on failure — do not expect failure-cause detail from this metric's tags. |
| `payos.security.authz.denials` | Counter | `app_id`, `resource_type`, `method`, `outcome=denied`, `exception=none` | Also emitted by `payos-server-http`'s `/metrics` handler with `resource_type=metrics` when a remote client is denied under `local-only` — same metric name, two producers, distinguish by `resource_type`. |
| `payos.resource.requests` | Counter | `app_id`, `resource_type`, `method`, `outcome`, `exception`, `status_family` | Resource handler completions. |
| `payos.resource.duration` | Timer | same as above | Resource handler latency. |
| `payos.resource.errors` | Counter | same as above | Emitted only when `outcome=error`. |
| `payos.script.executions` | Counter | `app_id`, `resource_type`, `method`, `outcome=success`, `exception=none` | Only recorded on the success path. |
| `payos.script.execution.duration` | Timer | same, `outcome=success` always | Only recorded on success; script failures show up in `script.errors`, not as a duration sample. |
| `payos.script.errors` | Counter | `app_id`, `resource_type`, `method`, `outcome=error`, `exception` | |

### Idempotency — `payos-kernel` (`IdempotencyService`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.idempotency.checks` | Counter | `store_type`, `outcome` | `outcome` ∈ `disabled`, `unknown`, `rejected`, `miss`, `hit`; the `store_type` tag identifies which backend serviced the check but is not otherwise called out in the architecture doc. |
| `payos.idempotency.store.duration` | Timer | `store_type`, `outcome` | |
| `payos.idempotency.rejections` | Counter | `store_type`, `outcome=rejected` | Duplicate in-progress requests. |
| `payos.idempotency.misses` | Counter | `store_type`, `outcome=miss` | |
| `payos.idempotency.hits` | Counter | `store_type`, `outcome=hit` | Cached response replays. |
| `payos.idempotency.stores` | Counter | `store_type`, `outcome=stored` | Response persistence attempts. |

The `idempotency-service-redis` module itself has no metrics instrumentation — everything above comes from the kernel-side `IdempotencyService` wrapper, so Redis-level latency/errors are not separately observable.

### Audit — `payos-kernel` (`AuditLogger`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.audit.events` | Counter | Success path: `event_type`, `result`, `outcome` (no `exception`). Error path (`logEvent` catch): `event_type=unknown`, `outcome=error` (no `result`). Convenience methods (`recordAuditCall`, used by auth/authz/session/startup/shutdown/API/decryption helpers): `event_type`, `result`, `outcome`, `exception`. | The same metric name is produced with three different tag-key sets depending on the code path. Any PromQL that assumes a fixed label set on `payos.audit.events` (e.g. grouping by `exception`) will silently drop the `logEvent` success/error samples, which don't carry that label. |
| `payos.audit.log.duration` | Timer | `outcome` (from `logEvent`) or `event_type`, `outcome`, `exception` (from `recordAuditCall`) | Same tag-set caveat as above. |

Audit metrics answer volume/latency/failure-rate questions only; the structured audit log remains the record of truth (see [observability.md](observability.md)).

### Connector — `payos-kernel` (`ConnectorScriptHandle`)

All connector metrics share the same 6-tag set on every call: `connector_type`, `connector_name`, `operation`, `outcome`, `error_code`, `error_category`.

| Metric | Type | Meaning |
| --- | --- | --- |
| `payos.connector.executions` | Counter | Execution attempts by outcome. |
| `payos.connector.execution.duration` | Timer | Execution latency. |
| `payos.connector.errors` | Counter | `outcome` ∈ `error`, `rejected`, `suppressed`, `lookup_failed`. |
| `payos.connector.deduplication.decisions` | Counter | `outcome` ∈ `replayed`, `suppressed`. |
| `payos.connector.retry.decisions` | Counter | `outcome` ∈ `retry`, `no_retry`. |
| `payos.connector.terminal_routing` | Counter | `outcome` = lower-cased routing destination. |
| `payos.connector.execution.state` | Counter | `outcome` = lower-cased execution state. |

`payos-connector-api` and `payos-connector-sdk` themselves carry no metrics — all connector observability comes from the kernel.

### Queue and NATS — three independent producers, same metric names

The `payos.queue.*` family is emitted from three places that do not share a discriminator tag consistently — this is the single biggest cross-cutting gotcha in the catalog, worth reading in full before building a queue dashboard.

**`payos-server-queue` (`QueueServer`)** — tags always include `server=QueueServer`:

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.queue.connection.state` | Gauge | `server=QueueServer`, `operation` (`connect`/`disconnect`), `topic_count`, `outcome`, `exception` |
| `payos.queue.subscribe.duration` | Timer | `server=QueueServer`, `operation=subscribe`, `topic_count`, `outcome=success`, `exception=none` |
| `payos.queue.subscribe.failures` | Counter | same, `outcome=error` |
| `payos.queue.messages.consumed` | Counter | `server=QueueServer`, `operation=consume`, `topic_count`, `outcome`, `exception=none` |
| `payos.queue.consume.duration` | Timer | same |
| `payos.queue.acks` | Counter | `server=QueueServer`, `operation=consume`, `topic_count`, `outcome`, `exception` |
| `payos.queue.nacks` | Counter | same, `outcome` ∈ `stopped`, `ack_error` |

**`queue-service-nats` (`NatsQueueClient`)** — tags always include `client=nats`:

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.queue.connection.state` | Gauge | `client=nats`, `operation` (`connect`/`disconnect`/`is_connected`), `outcome`, `exception` |
| `payos.queue.connect.duration` | Timer | `client=nats`, `operation=connect`, `outcome`, `exception` |
| `payos.queue.subscribe.failures` | Counter | `client=nats`, `operation` (`subscribe`/`subscribe_jetstream`), `outcome=not_connected`, `exception=none` |
| `payos.queue.subscribe.duration` | Timer | `client=nats`, `operation` (`subscribe`/`subscribe_jetstream`), `outcome=success` |
| `payos.queue.messages.consumed` | Counter | `client=nats`, `operation` (`consume`/`consume_jetstream`), `outcome`, `exception` |
| `payos.queue.consume.duration` | Timer | same |
| `payos.queue.consume.failures` | Counter | same, `outcome=error` |
| `payos.queue.nacks` | Counter | `client=nats`, `operation=consume_jetstream`, `outcome=error`, `exception` (core NATS never nacks, only JetStream) |
| `payos.queue.publishes` | Counter | `client=nats`, `operation` (`publish`/`publish_jetstream`), `outcome`, `exception` |
| `payos.queue.publish.duration` | Timer | same |
| `payos.queue.publish.failures` | Counter | same, `outcome` ∈ `not_connected`, `error` |

**`payos-kernel` `$Queue` script binding (`QueueBinding`)** — carries neither `server` nor `client`:

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.queue.publishes` | Counter | `operation` (`default`/`reply`/`destination`), `outcome`, `exception` |
| `payos.queue.publish.duration` | Timer | same |
| `payos.queue.publish.failures` | Counter | same, `outcome=error` |
| `payos.queue.connection.state` | Gauge | `operation=connection`, `outcome` (`connected`/`disconnected`), `exception=none` |

Because these three producers emit the same metric name with different label sets, a single Prometheus series for e.g. `payos.queue.connection.state` mixes samples with `server=`, `client=`, and neither — `sum by (...)` queries must pick a discriminator deliberately (`server`, `client`, or the absence of both) rather than assuming one consistent shape.

### TCP — `payos-server-tcp` (`TcpServer`)

All tagged with `server=TcpServer` plus `outcome`/`exception`.

| Metric | Type | Meaning |
| --- | --- | --- |
| `payos.tcp.connections.accepted` | Counter | Accepted connections. |
| `payos.tcp.connections.active` | Gauge | Current active connections (`outcome=active`). |
| `payos.tcp.messages.decoded` | Counter | Successful inbound decodes. |
| `payos.tcp.decode.duration` | Timer | Decode latency. |
| `payos.tcp.messages.encoded` | Counter | Successful outbound encodes. |
| `payos.tcp.encode.duration` | Timer | Encode latency. |
| `payos.tcp.connection.errors` | Counter | Connection-level errors. |
| `payos.tcp.connections.closed` | Counter | Closed connections, `outcome` ∈ `success`, `error`. |
| `payos.tcp.connection.duration` | Timer | Connection lifetime. |
| `payos.tcp.server.errors` | Counter | Server-level failures. |

### Database — `dynamic-database-service` (`DynamicDataAccessService`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.db.operations` | Counter | `operation`, `outcome`, `exception` | `operation` ∈ `list`, `unique`, `first`, `execute_update`, `transaction_begin`, `transaction_commit`, `transaction_rollback`, `get`, `get_fresh`, `save`, `update`, `delete`, `delete_by_id`, `find_all`, `find` — stable, method-level, never a table/entity name. |
| `payos.db.operation.duration` | Timer | same | |
| `payos.db.errors` | Counter | `operation`, `exception` (no `outcome`) | |

### Webhook — `webhook-service-http` (`HttpWebhookDispatcher`)

All tagged with `dispatcher=http`.

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.webhook.serialization.failures` | Counter | `exception` | Payload serialization failures. |
| `payos.webhook.submissions` | Counter | `dispatcher=http` only | Submission attempts. |
| `payos.webhook.delivery.attempts` | Counter | `dispatcher=http`, `outcome`, `exception` | |
| `payos.webhook.delivery.duration` | Timer | same | |
| `payos.webhook.delivery.failures` | Counter | `dispatcher=http`, `exception` (no `outcome`) | |
| `payos.webhook.delivery.exhausted` | Counter | `dispatcher=http` only | Events that exhausted retry attempts. |
| `payos.webhook.http.responses` | Counter | `dispatcher=http`, `status_family`, `outcome` | Response status family from the webhook target. |

### Redis session store — `session-service-redis` (`RedisSessionStore`)

| Metric | Type | Tags | Meaning |
| --- | --- | --- | --- |
| `payos.session.store.operations` | Counter | `operation`, `backend=redis`, `outcome`, `exception` | `operation` ∈ `save`, `load`, `delete`, `touch`, `count_active`. |
| `payos.session.store.operation.duration` | Timer | same | |
| `payos.session.store.errors` | Counter | `operation`, `backend=redis`, `exception` (no `outcome`) | |
| `payos.session.store.hits` | Counter | `operation=load`, `backend=redis` (no `outcome`/`exception`) | |
| `payos.session.store.misses` | Counter | same | |
| `payos.session.store.active` | Gauge | `operation=count_active`, `backend=redis` | |

### Script bindings — `payos-kernel` (`ma.s2m.payos.scripting.*`)

These instrument the interfaces exposed to application scripts, not the backend modules directly (see gaps below).

**`$Cache` (`CacheBinding`)**

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.cache.operations` | Counter | `operation` (`put`/`get`/`remove`/`exists`/`increment`), `outcome`, `exception` |
| `payos.cache.operation.duration` | Timer | same |
| `payos.cache.hit` | Counter | same, fires when `outcome=hit` |
| `payos.cache.miss` | Counter | same, fires when `outcome=miss` |
| `payos.cache.errors` | Counter | same, fires when `outcome=error` |

**`$Secrets` (`SecretsBinding`)**

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.secret.operations` | Counter | `operation` (`get`/`list`/`tokenize`/`detokenize`/`encrypt`/`decrypt`/`sign`/`verify`), `outcome`, `exception` |
| `payos.secret.operation.duration` | Timer | same |
| `payos.secret.failures` | Counter | same, fires when `outcome` ∈ `error`, `unsupported` |

**`$SlidingWindow` (`SlidingWindowBinding`)**

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.ratelimit.decisions` | Counter | `operation=count`, `outcome`, `exception=none` |
| `payos.ratelimit.decision.duration` | Timer | same |
| `payos.ratelimit.errors` | Counter | `operation=count`, `outcome=error`, `exception` |

**`$Notification` (`NotificationBinding`)**

| Metric | Type | Tags |
| --- | --- | --- |
| `payos.notification.submissions` | Counter | `channel`, `outcome`, `exception=none` |
| `payos.notification.delivery.duration` | Timer | same |
| `payos.notification.delivery.failures` | Counter | `channel`, `outcome=error`, `exception` |

**`$Queue` (`QueueBinding`)** — listed under Queue and NATS above.

**`$Metrics` (`MetricsBinding`) — not covered by the architecture doc**

`ApiResourceHandler` exposes a `$Metrics` binding to scripts with a single `record(name, type, value, unit, tags)` entry point. It builds a `MetricSample` directly and sends it through `MetricsRecorder`, bypassing every `PayOSMetrics` convenience method. In practice this means script code can publish **any metric name, any type (counter/gauge/histogram), any unit, and any tag set** — including high-cardinality tags — with only a try/catch-and-log-at-warn as a safety net, no naming or cardinality enforcement. This is the one place in the platform where the metrics category does not go through a fixed-shape facade the way audit/connector/queue metrics do; treat any script-emitted metric name as untrusted for cardinality purposes, and if this binding is used in production, budget dashboard/alert time to check what names and tags scripts are actually publishing.

## Known gaps

These modules ship no `PayOSMetrics` or Micrometer instrumentation at all, so their internal success/failure/latency is invisible to Prometheus even though several are runtime-critical dependencies: `idempotency-service-redis`, `cache-service-memory`, `cache-service-redis`, `sliding-window-counter-memory`, `sliding-window-counter-redis`, `secret-service-filesystem`, `secret-service-vault`, `payos-connector-api`, `payos-connector-sdk`, `payos-service-notification`, `payos-notification-connector`, `payos-runtime`, `payosv2-packer`, `payos-pm`. For cache, secrets, rate limiting, and notification, the kernel-side script bindings (`$Cache`, `$Secrets`, `$SlidingWindow`, `$Notification` above) give partial visibility at the interface level, but backend-specific failure modes (e.g. a Redis connection pool exhausting, a Vault token expiring) are not separately observable from those bindings.

`PayOSMetrics.ratio(...)` exists as a public helper (maps to a `GAUGE` with `unit=ratio`) but has zero call sites anywhere in the codebase today — it is dead API, not a bug, but don't expect any `*.ratio` style metric until something starts calling it. Similarly, `MicrometerMetricsRecorder`'s `DistributionSummary` mapping (for `HISTOGRAM` samples with a non-time unit) is unreachable through any `PayOSMetrics` call today — every `duration(...)` call uses a time unit and becomes a `Timer` — so the only way a `DistributionSummary` appears in Prometheus today is a script explicitly requesting it through `$Metrics`.

## Building dashboards and alerts

- Filter `payos.queue.*` by the `server`/`client` discriminator tag (or its absence, for the `$Queue` script binding) before aggregating — see the Queue and NATS section above.
- Don't group `payos.audit.events` or `payos.audit.log.duration` by `exception` without checking whether the code path you care about actually sets that tag — the `logEvent` success/error paths don't.
- `payos.runtime.startup.duration` and `payos.runtime.server.start.duration` only have success samples; alert on `payos.runtime.startup.failures` / `payos.runtime.servers.started{outcome="error"}` for failure detection, not on the absence of a duration sample.
- A non-zero rate of `payos.security.authz.denials{resource_type="metrics"}` from a remote client usually means a scrape target was misconfigured against a `local-only` instance rather than an active attack, but is still worth alerting on.
- Suggested alert set (mirrors [architecture/observability-metrics-architecture-v1-2026-09-06.md](../architecture/observability-metrics-architecture-v1-2026-09-06.md) §Alerts): `payos.runtime.startup.failures` any increment; sustained `payos.transport.errors` / `payos.resource.errors` rate; `payos.script.errors` rate; `payos.queue.connection.state == 0` sustained; `payos.audit.events{outcome="error"}`; `payos.session.store.errors` rate; `payos.webhook.delivery.exhausted` rate.

## Related

- [architecture/observability-metrics-architecture-v1-2026-09-06.md](../architecture/observability-metrics-architecture-v1-2026-09-06.md) — runtime design, contract layer, recorder resolution, tag/cardinality policy, extension guidance.
- [observability.md](observability.md) — correlation IDs, logging/MDC, audit trail, diagnostics, health endpoints.
- [reference/http-endpoints.md](../reference/http-endpoints.md) — full HTTP endpoint reference including `/metrics`.
