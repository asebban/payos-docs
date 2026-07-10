# Connector Framework — Configuration Parameters

> **Created:** 2026-07-10
> **Last updated:** 2026-07-10
> **Version:** 1

## Scope

This page documents **every** configuration parameter of the PayOS **connector framework** —
the business/payment connector plugin system (`IConnector`, `$Connector(...)` script binding,
`ConnectorDescriptor`, `TenantConnectorRegistry`) built across Epics 1–5. It does **not** cover
the older, unrelated `connectors-dir` SPI-backend loader (database/queue/secret factory
plugins) described in [extensions-connectors.md](extensions-connectors.md) — the two
mechanisms share the word "connector" but are otherwise independent. See
[Naming clash with the legacy `connectors-dir` loader](#naming-clash-with-the-legacy-connectors-dir-loader)
below.

> **Status note:** every parameter below is implemented and covered by tests in `payos` /
> `payos-connector-sdk`. As of this writing, `BootServer` does not yet call
> `ConnectorConfigurationLoader`, `ConnectorJarScanner`, or `ConnectorRuntimeInitializer` — the
> framework is fully built and testable in isolation, but not yet wired into the production
> bootstrap sequence. Treat the parameters below as the stable contract that future wiring will
> honor, not as something you can drop into a running deployment today.

## Two configuration surfaces

The connector framework splits configuration into two deliberately separate surfaces, owned by
two different people:

| Surface | Who declares it | Where it lives | Mutable per deployment? |
| --- | --- | --- | --- |
| **Descriptor** | The connector author (the person who wrote the connector JAR) | `META-INF/connector.properties`, packaged inside the connector JAR | No — fixed facts about the connector's own contract and business nature |
| **Configuration** | The platform operator (the person deploying PayOS) | `connectors.json` | Yes — per environment/tenant deployment |

This split exists so that facts intrinsic to a connector (its type, its SDK version, whether it
requires an idempotency key) cannot be silently overridden by a deployment config file — see
`payos/_bmad-output/implementation-artifacts/5-2-connector-prevent-connector-owned-deduplication-decisions.md`
(Addendum) for the rationale behind the most recent addition, `requiresIdempotencyKey`.

## 1. Connector descriptor (`META-INF/connector.properties`)

Packaged inside the connector JAR at `META-INF/connector.properties`, parsed by
`ma.s2m.payos.connector.descriptor.ConnectorDescriptorParser` (payos-connector-sdk) into a
`ConnectorDescriptor`. Keys are defined in `ConnectorDescriptorKeys`.

| Property key | Required | Type | Default | Description |
| --- | --- | --- | --- | --- |
| `connector.type` | Yes | string | — | Logical connector family, e.g. `CardNetwork`, `Switch`, `PaymentGateway`, `Fraud`. Selects which connectors a script's `$Connector('type')` call can resolve by default. |
| `connector.name` | Yes | string | — | Unique instance name within its type, e.g. `visa`, `cmi`. Used for `$Connector('type', 'name')` and direct `$Connector('name')` lookups. |
| `connector.api.version` | Yes | string (`major.minor`) | — | The SDK API version this connector was built against (currently `1.0`, `IConnector.CONNECTOR_API_VERSION`). Checked at load time by `ConnectorCompatibilityPolicy` — see [API version compatibility](#2-api-version-compatibility) below. |
| `connector.required.params` | No | comma-separated string | empty | Names of `parameters` (see `connectors.json` below) this connector cannot function without. Whitespace around each name is trimmed; a doubled comma (empty segment) fails descriptor parsing. Enforced at JAR-scan time by `ConnectorJarScanner` — a configured connector missing one of these is rejected as an invalid JAR before it ever reaches the lifecycle initializer. |
| `connector.requires.idempotency` | No | boolean (`true`/`false`) | `false` | **New.** When `true`, `ConnectorScriptHandle.execute(...)` refuses to invoke this connector unless the caller supplied a non-blank idempotency key — it returns a `CONNECTOR_IDEMPOTENCY_KEY_REQUIRED` error response before the connector or the deduplication gate is ever reached. Intended for connectors performing non-repeatable financial operations (card networks, switches); connectors with legitimately non-idempotent operations (e.g. read-only lookups) should leave this `false` (the default). |

Example `META-INF/connector.properties`:

```properties
connector.type=CardNetwork
connector.name=visa
connector.api.version=1.0
connector.required.params=baseUrl,clientId,clientSecret
connector.requires.idempotency=true
```

## 2. API version compatibility

`ConnectorCompatibilityPolicy` (payos-connector-sdk) evaluates `connector.api.version` against
the runtime's own `IConnector.CONNECTOR_API_VERSION` (currently `1.0`) at JAR-scan time:

| Situation | Result |
| --- | --- |
| Descriptor omits `connector.api.version` | `COMPATIBLE_WITH_WARNING` — treated as a legacy connector, logged with a warning. |
| Same major version, connector minor ≤ runtime minor | `COMPATIBLE` |
| Same major version, connector minor > runtime minor | `COMPATIBLE_WITH_WARNING` — targets a newer minor than the runtime knows about. |
| Different major version, within a configured transition window (`ConnectorCompatibilityPolicy.withTransitionWindow(...)`) | `COMPATIBLE_WITH_WARNING` |
| Different major version, no transition window configured | `INCOMPATIBLE` — JAR rejected with error code `CONNECTOR_INCOMPATIBLE`; connector never reaches `INITIALIZING`. |
| Unparseable version string | `INCOMPATIBLE` |

A transition policy (`withTransitionWindow`) may support at most two major versions
simultaneously, and requires at least one minor release window when it does.

## 3. Connector configuration (`connectors.json`)

Operator-facing file, loaded and structurally validated by
`ma.s2m.payos.config.connector.ConnectorConfigurationLoader` into a `ConnectorConfiguration`
(a list of `ConnectorConfigurationEntry`). Loaded from the runtime base directory (the same
root as `bootstrap.json`) by `loadFromRuntimeDirectory(...)`; a missing file is treated as a
valid empty configuration (PayOS can run with zero connectors configured).

| Field | Required | Type | Description |
| --- | --- | --- | --- |
| `connectors` | No (defaults to `[]`) | array of objects | Top-level array; each element configures one connector instance. |
| `connectors[].type` | Yes | string | Must match the connector's own `connector.type` descriptor value — used (together with `name`) to match this entry to a scanned JAR. |
| `connectors[].name` | Yes | string | Must match the connector's own `connector.name` descriptor value. |
| `connectors[].jar` | Yes | string | Path to the connector JAR, relative to the runtime base directory (e.g. `connectors/visa.jar`). Also usable as a fallback match key if `type`/`name` don't align with the scanned descriptor. |
| `connectors[].parameters` | No (defaults to `{}`) | object (string → any) | Free-form key/value parameters passed to `IConnector.init(ConnectorConfig)` at startup. Must cover every name listed in the connector's `connector.required.params` descriptor key. Values may contain `${...}` credential reference tokens (see below). Never logged or `toString()`-rendered in the clear — always passed through `SensitiveFieldMasker` first. |

Example `connectors.json`:

```json
{
  "connectors": [
    {
      "type": "CardNetwork",
      "name": "visa",
      "jar": "connectors/visa.jar",
      "parameters": {
        "baseUrl": "https://visa.example.internal",
        "clientId": "${VISA_CLIENT_ID}",
        "clientSecret": "${VISA_CLIENT_SECRET}"
      }
    },
    {
      "type": "Fraud",
      "name": "risk-score",
      "jar": "connectors/risk.jar"
    }
  ]
}
```

Structural validation errors (missing `type`/`name`/`jar`, `parameters` present but not an
object, `connectors` present but not an array) raise `ConnectorConfigurationException` with a
message identifying the offending array index and, where available, `type`/`name` — parameter
values themselves are never included in the error message, since they may contain unresolved
secret tokens.

### Credential reference tokens in `parameters` and `jar`

`ConnectorCredentialReferenceResolver.resolve(...)` resolves `${...}` tokens in every
`parameters` value and in `jar` using the same placeholder syntax as the rest of PayOS
configuration (see [env-var-resolution.md](env-var-resolution.md) for the full `${VAR}`,
`${VAR:-default}`, `${file:/path}`, `${config:a.b}` grammar) — **with one difference**: it calls
`EnvVarResolver.resolveRequired(...)`, so a simple `${VAR}` token that resolves to an unset or
blank value **fails fast** with a `ConnectorInitializationException` instead of silently
substituting an empty string. This is deliberate — an empty `clientSecret` reaching a payment
connector is a worse failure mode than refusing to start.

## 4. Hot-reload runtime setting

`ConnectorRuntimeSettingsEvaluator.evaluate(bootstrapSettings)` reads a single boolean from the
bootstrap settings map:

| Bootstrap key | Type | Default | Effect |
| --- | --- | --- | --- |
| `config-hot-reload-enabled` (`IConfigSpec.CONFIG_HOT_RELOAD_ENABLED`) | boolean | `true` | Governs both the general config-file watcher (see [operations/hot-reload.md](../operations/hot-reload.md)) **and** whether `ConnectorRuntimeReloader` is permitted to switch in a replacement connector JAR. When `false`, `ConnectorRuntimeReloader.reload(...)` returns `DISABLED` immediately and the current connector instance is kept; replacing a connector then requires a restart. |

This is the **same** global key already documented for the config-file watcher — the connector
framework does not define a connector-specific hot-reload toggle.

### Reload mechanics (not separately configurable, but load-bearing)

`ConnectorRuntimeReloader.reload(...)` uses a fixed default drain timeout,
`ConnectorRuntimeReloader.DEFAULT_DRAIN_TIMEOUT` = **30 seconds**, before giving up on a
graceful switch-over (callers may pass an explicit `Duration` instead). Sequence: validate the
replacement JAR reaches `READY` → wait for in-flight executions against the current connector
to drain (`ConnectorDrainBarrier`) → close the current connector → switch. A drain timeout
closes the *replacement* (not the current) connector and keeps the old one active
(`DRAIN_TIMEOUT` result), so a slow-draining connector never leaves PayOS without a working
instance.

## 5. Tenant scoping

`TenantConnectorRegistry` binds `ConnectorLifecycleEntry` lists to tenant IDs
(`TenantConnectorRegistry.builder().tenant(tenantId, entries).build()`). There is currently
**no dedicated JSON configuration** for which tenants see which connectors — this is a
programmatic API surface only; a future story would need to define how `connectors.json`
entries map to tenants (e.g. a per-tenant connector list, or a shared list visible to every
tenant). Only lifecycle entries in `ConnectorLifecycleState.READY` are visible to scripts
(`scriptVisible()`); duplicate `type::name` registrations for the same tenant raise
`IllegalArgumentException` at registry-build time.

## Naming clash with the legacy `connectors-dir` loader

PayOS has an older, unrelated mechanism also called "connectors": `ma.s2m.payos.config.ConnectorLoader`,
configured via the `connectors-dir` bootstrap key (see
[extensions-connectors.md](extensions-connectors.md)). That loader only builds a
`URLClassLoader` over JARs in a directory so that **SPI factory** plugins (`IDatabaseServiceFactory`,
`IQueueClientFactory`, `ISecretProviderFactory`) can be discovered — it has no descriptor
format, no `connectors.json`, no idempotency/audit/dedup semantics, and **is** wired into
`BootServer` today. Do not confuse the two when reading configuration or code named
`Connector*`:

| | Legacy SPI loader | Connector framework (this page) |
| --- | --- | --- |
| Config key | `connectors-dir` (bootstrap) | `connectors.json` + `META-INF/connector.properties` |
| Wired into `BootServer`? | Yes | Not yet |
| Purpose | Load database/queue/secret backend plugins | Business/payment connectors callable from scripts via `$Connector(...)` |
| Descriptor format | None | `ConnectorDescriptor` (this page, §1) |

## References

- `payos-connector-sdk/src/main/java/ma/s2m/payos/connector/descriptor/` — `ConnectorDescriptor`, `ConnectorDescriptorKeys`, `ConnectorDescriptorParser`.
- `payos-connector-sdk/src/main/java/ma/s2m/payos/connector/compatibility/ConnectorCompatibilityPolicy.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorConfigurationLoader.java`, `ConnectorConfigurationEntry.java`, `ConnectorCredentialReferenceResolver.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorRuntimeSettings.java`, `ConnectorRuntimeSettingsEvaluator.java`, `ConnectorRuntimeReloader.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/TenantConnectorRegistry.java`.
- `payos/_bmad-output/implementation-artifacts/` — Epic 1–5 connector framework stories, in particular 5-2's Addendum for `requiresIdempotencyKey`.
- [env-var-resolution.md](env-var-resolution.md) — full `${...}` placeholder grammar.
- [extensions-connectors.md](extensions-connectors.md) — the unrelated legacy SPI-backend loader.
