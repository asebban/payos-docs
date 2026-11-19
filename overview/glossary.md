# Glossary

Precise definitions of the core PayOS concepts. Terms are cross-referenced throughout the
documentation; this page is the single source for their meaning.

## Deployment & application concepts

**Application**
: A unit of business functionality deployed *into* the runtime. It is a directory of
resources — API scripts (`api/`), pages (`page/`), components, menus (`menu/`), shared
libraries (`lib/`), hook scripts (`hook/`), and translations (`i18n/`) — registered in the
runtime configuration. Identified by an `id`; addressed as the first path segment of an
inbound URI (`/{appId}/...`). Modeled by `ma.s2m.payos.applications.Application`. Managed
with [`apm`](../cli-tools/apm.md). See [developer/application-model.md](../developer/application-model.md).

**Capability**
: An application with `category: "capability"`. It is a self-contained extension that other
applications declare via `extends`, and that can be installed and activated without
redeploying the runtime. Capability resources are only resolved when the capability is
*active* for the requesting app/tenant. Managed with [`cpm`](../cli-tools/cpm.md).

**Product**
: A bundle of applications plus shared server configuration, described by a product
manifest. Managed with [`ppm`](../cli-tools/ppm.md).

**Extends**
: The inheritance relationship between applications. An application lists parent app/ capability IDs in `extends`; the [resource locator](../architecture/request-processing.md) walks this chain to resolve a resource.

**Bundle**
: The on-disk runtime directory rooted at `payos.json` (passed via `--bundle-path`). It contains the merged configuration, the registered applications, and the capability state.

## Runtime concepts

**Kernel**
: The `payos` module (`ma.s2m.payos`). The rigid, high-performance core that owns the
bootstrap, configuration, request pipeline, scripting engine, security, and all SPI
contracts. Never modified to add a feature.

**Runtime**
: The running PayOS process, produced by `payos-runtime` and started by `BootServer`.
Applications run inside it as a container (the *externalized runtime* principle: Applications run inside the runtime as data, not as part of the server binary. The application is completely decoupled from the server/transport layer).

**BootServer**
: `ma.s2m.payos.BootServer` — the executable entry point (declared in the shade plugin
manifest). Loads configuration, starts the configured servers, and wires up services.

**Tenant**
: An isolated customer/organization context. Tenant identity is carried as the
`X-Tenant-Id` header (or context/MDC) and enforced by architecture, not configuration. See
[architecture/multi-tenancy.md](../architecture/multi-tenancy.md).

**Correlation ID**
: A per-request trace identifier carried as `X-Correlation-Id`. Generated as a UUID at
ingress if absent, then propagated unchanged through context, logs, and responses across
HTTP, TCP, and Queue transports.

## Extensibility concepts

**Connector (SPI)**
: A service-provider JAR placed in `service-adapters-dir` and discovered through Java's
`ServiceLoader`. Connectors implement an SPI factory (`IDatabaseServiceFactory`,
`IQueueClientFactory`, `IWebhookDispatcherFactory`, `ISecretProviderFactory`,
`INotificationServiceFactory`). See
[architecture/extensibility.md](../architecture/extensibility.md).

**Connector (business/payment framework)**
: An unrelated, newer mechanism sharing the same name: a JAR implementing `IConnector`
(from the standalone `connector-sdk` Maven module), configured via `connectors.json` and a
`META-INF/connector.properties` descriptor, callable from scripts via
`$Connector(type[, name]).execute(payload)`. Covers idempotency, platform-owned
deduplication, retry policy, execution-state persistence, and DLQ/terminal-state routing —
none of which the SPI connector above has any concept of. Wired into `BootServer` since
2026-07-27 via `ConnectorFrameworkInitializer`.
See [configuration/connector-framework-parameters-v3-2026-08-11.md](../configuration/connector-framework-parameters-v3-2026-08-11.md).

**Extension**
: A Java library JAR placed in `extensions-dir` whose classes become callable from
JavaScript via `Java.type('com.example.Foo')`. Unlike a connector, an extension implements
no PayOS interface. See [developer/java-extensions.md](../developer/java-extensions.md).

**SPI (Service Provider Interface)**
: A kernel-defined interface (e.g. `ISecretProvider`) implemented in a standalone module
and discovered at runtime. The kernel ships only the interface; implementations live
outside it.

**Transport / Server provider**
: An implementation of `ServerProvider` registered via `ServiceLoader`, contributing a new
protocol (`http`, `https`, `tcp`, `queue`). See [architecture/request-processing.md](../architecture/request-processing.md).

## Scripting concepts

**Script binding**
: A host object injected into the JavaScript sandbox before execution (a `$`-prefixed global such as `$Request`, `$Response`, `$DB`, `$Queue`, `$Secrets`). The full list is in [developer/scripting-bindings.md](../developer/scripting-bindings.md) and [reference/script-bindings-index.md](../reference/script-bindings-index.md).

**`loadControlData` / `execute` / `emitInsight`**
: The three-function contract a PayOS API script exposes. `loadControlData(request)` returns control data; `execute(request, controlData)` returns the response; `emitInsight(request, 
response, payload)` emits optional analytics/business insight. See [developer/writing-apis.md](../developer/writing-apis.md).

**Resource**
: A loadable artifact within an application — `api` (`.js`), `page`/`component` (`.vue`, `.html`), `menu`, `lib`, `i18n`, and `hook` (event-interception scripts). Resource types are defined by `ma.s2m.payos.resources.IResource`.

**Hook / System event**
: An interception point (`api.pre-request`, `api.post-request`, `api.on-error`, `security.login`, capability lifecycle events, page events, …) where PayOS can run hook scripts and dispatch native webhooks. The full list is in [reference/system-events.md](../reference/system-events.md).

## Service concepts

**Database service** (`$DB`)
: The `IDatabaseService` implementation providing multi-tenant data access. See [developer/data-access.md](../developer/data-access.md).

**Queue client** (`$Queue`)
: The `IQueueClient` implementation for publish/subscribe messaging (NATS by default). See [developer/queue-messaging.md](../developer/queue-messaging.md).

**Secret provider** (`$Secrets`)
: The `ISecretProvider` implementation for retrieving and storing secrets (filesystem or Vault). See [developer/secrets-usage.md](../developer/secrets-usage.md).

**Webhook dispatcher** (`$WebHooks`)
: The `IWebhookDispatcher` implementation that delivers webhook events over HTTP. See [developer/webhooks-and-hooks.md](../developer/webhooks-and-hooks.md).

## Task scheduling concepts

**Task Scheduling System (TSS)**
: `payos-job-scheduler`, the platform's clustered job scheduler built on Quartz. Runs standalone (its own process) or embedded in `payos-runtime`. See [ADR-0009 v5](../architecture/adr/0009-task-scheduling-system-v5-2026-11-13.md) and [operations/task-scheduling-system-v21-2026-11-15.md](../operations/task-scheduling-system-v21-2026-11-15.md).

**Job script**
: A JavaScript file exporting `execute(context)`, run by TSS on a cron schedule or on demand, with `$DB`, `$PlatformDB`, `$Parallel` and `$Logger`. Declared in `jobs/<tenantId>/jobs.json`, `custom-jobs.json` or `jobs/platform-jobs.json`. See [developer/task-scheduler-guide-v21-2026-11-19.md](../developer/task-scheduler-guide-v21-2026-11-19.md).

**Job reconciliation**
: The TSS step that aligns the scheduler with the `jobs/` files (the source of truth): it adds new jobs, applies changed cron expressions, `parallelizable` flags and arguments, and deletes jobs no longer declared. Runs at startup, on hot reload and on `sync`.

**Manual trigger**
: Running one TSS job now, outside its schedule, with the `trigger` command or the secured `/tss/admin/jobs/trigger` endpoint. The run is tagged `triggerType = manual` and audited as `TSS_JOB_TRIGGERED`.

**Jobs template** (`jobs/template.json`)
: The model TSS copies into `jobs/<tenantId>/jobs.json` when it creates the directory of a newly declared tenant, with every `{{tenantId}}` replaced by the tenant id. See [configuration/task-scheduling-system-v11-2026-11-15.md](../configuration/task-scheduling-system-v11-2026-11-15.md#jobstemplatejson--model-for-new-tenants).
