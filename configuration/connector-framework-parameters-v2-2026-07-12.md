# Connector Framework — Configuration Parameters

> **Created:** 2026-07-10
> **Last updated:** 2026-07-12
> **Version:** v2

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
| `connector.requires.idempotency` | No | boolean (`true`/`false`) | `false` | When `true`, `ConnectorScriptHandle.execute(...)` refuses to invoke this connector unless the caller supplied a non-blank idempotency key — it returns a `CONNECTOR_IDEMPOTENCY_KEY_REQUIRED` error response before the connector or the deduplication gate is ever reached. Intended for connectors performing non-repeatable financial operations (card networks, switches); connectors with legitimately non-idempotent operations (e.g. read-only lookups) should leave this `false` (the default). |

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

## 6. Idempotency and platform-owned deduplication (Epic 5.1–5.2)

None of this section has a JSON/bootstrap config key — it's runtime behavior wired into every
`$Connector(...).execute(payload)` call. Documented here because it directly shapes what an
operator/support engineer sees when investigating connector execution issues.

### 6.1 Idempotency context passed to the connector

`ConnectorExecutionContext` (`payos-connector-sdk`, `ma.s2m.payos.connector.model`) is a record
`(tenantId, correlationId, operation, attempt, idempotencyContext, metadata)` — `correlationId`
is the only required field; the rest are nullable. `idempotencyContext` is an
`IdempotencyContext(String key, Map<String,String> metadata)` (`key` required non-blank),
built by `ConnectorScriptHandle` from the caller-supplied idempotency key (resolved from the
`X-Idempotency-Key` HTTP header by `ApiResourceHandler.resolveCurrentIdempotencyKey(...)`).
Connectors can read this context to propagate provider-side idempotency metadata, but — per
the framework's core design constraint — **they never decide whether to suppress, replay, or
persist a duplicate request**. That decision belongs entirely to the platform (§6.2).

### 6.2 Platform-owned deduplication gate

Before a connector is ever invoked, `ConnectorScriptHandle.execute(...)` calls
`ConnectorDeduplicationGate.evaluate(idempotencyKey)`, which returns one of three actions:

| Action | Meaning | Response returned to the script |
| --- | --- | --- |
| `EXECUTE` | No key supplied, or an unseen key claimed for this execution | Proceeds to invoke the connector normally |
| `REPLAY` | The key was already seen and completed | The cached `ConnectorResponse` from the original execution — the connector is **not** invoked again |
| `SUPPRESS` | The same key is already in flight (a race) | `ConnectorResponse.error(RETRYABLE_ERROR, "CONNECTOR_DUPLICATE_IN_PROGRESS", ...)` |

Both `REPLAY` and `SUPPRESS` also record a `CONNECTOR_DUPLICATE_DETECTED` audit event
(`AuditLogger`/`AuditEvent`, result `REPLAYED` or `SUPPRESSED`) via
`ConnectorScriptHandle.recordDuplicateAudit(...)`. The backing store is
`ma.s2m.payos.idempotency.ConnectorIdempotencyStores` — a static, swappable facade (default
`InMemoryConnectorIdempotencyStore`) — deliberately **separate** from the pre-existing
HTTP-level `IIdempotencyStore` used elsewhere in PayOS; the two are unrelated mechanisms that
happen to solve a similar problem at different layers.

If the connector's own descriptor sets `connector.requires.idempotency=true` (§1) and the
caller supplied no key, the request is rejected with `CONNECTOR_IDEMPOTENCY_KEY_REQUIRED`
**before** the dedup gate is even reached — this is a hard precondition, not a dedup outcome.

## 7. Normalized error categories (Epic 5.3)

Every connector execution outcome is classified into exactly one of five categories
(`ma.s2m.payos.connector.model.ConnectorErrorCategory`, payos-connector-sdk):

| Category | Meaning |
| --- | --- |
| `NONE` | Success — never produced by a failure path. |
| `RETRYABLE_ERROR` | Transient failure; eligible for retry under the platform's retry policy (§8). |
| `PERMANENT_ERROR` | Non-retryable business/protocol failure (e.g. issuer decline). |
| `TIMEOUT` | The call did not complete within the expected window; treated as retryable by the default policy. |
| `UNKNOWN_ERROR` | Anything the framework can't classify — including a connector declaring a root-cause category string the framework doesn't recognize, and any uncaught `RuntimeException` escaping the connector. |

Classification happens in `ConnectorScriptHandle`: a controlled `ConnectorExecutionException`
with a null or unmapped `rootCauseCategory()` string is classified `UNKNOWN_ERROR` (never
allowed to crash the classification step); an uncaught `RuntimeException` is always
`UNKNOWN_ERROR` regardless of its actual Java exception type.

## 8. Platform retry policy (Epic 5.4) — no config key today

`ConnectorRetryPolicy` decides, given a normalized error category and how many attempts have
already run, whether another attempt should happen. **There is currently no `connectors.json`
or bootstrap key to override this policy** — it is Java-code-only, accessed exclusively through
the static swappable facade `ConnectorRetryPolicies.getInstance()`. An operator looking for a
`maxAttempts` config knob will not find one; changing the policy today requires calling
`ConnectorRetryPolicies.setInstance(ConnectorRetryPolicy.withBudget(...))` in code.

`ConnectorRetryPolicy.defaultPolicy()`:

| Setting | Default value |
| --- | --- |
| `maxAttempts` | `3` |
| Retryable categories | `{RETRYABLE_ERROR, TIMEOUT}` — `PERMANENT_ERROR`, `UNKNOWN_ERROR`, and `NONE` are never retried by default |

Decision rule (`evaluate(ConnectorRetryContext)`): category not in the retryable set →
`noRetry`; `attempt >= maxAttempts` → `noRetry` ("retry budget exhausted"); otherwise →
`retry(attempt + 1, ...)`. The same inputs always produce the same decision (no randomness,
no wall-clock dependency) — this determinism is itself a tested contract, not just an
implementation detail.

## 9. Connector execution state persistence (Epic 5.5)

Every execution attempt against a connector is recorded as a
`ConnectorExecutionStateRecord(connectorType, connectorName, tenantId, correlationId,
operation, attemptCount, idempotencyKey, errorCategory, state, recordedAt)`, keyed by
`(correlationId, connectorType, connectorName)` and **upserted** on each attempt (so `get(...)`
always returns the *current* state, not a history log). State values
(`ma.s2m.payos.connector.state.ConnectorExecutionState`): `RUNNING` (recorded before the
connector is invoked — this is what makes "recovery after runtime interruption" meaningful:
a crash mid-execution leaves a queryable `RUNNING` row), `SUCCEEDED`, `RETRYING`, `FAILED`.
`isTerminal()` is derived (`true` for `SUCCEEDED`/`FAILED`), not a fifth state.

The record deliberately **excludes** error code, error message, and any payload/parameter
data — only the fields above are ever persisted, keeping the record's leak surface minimal.

Stores (all `ma.s2m.payos.connector.state`, accessed via the static facade
`ConnectorExecutionStateStores.getInstance()`):

| Store | Durable across restarts? | Notes |
| --- | --- | --- |
| `InMemoryConnectorExecutionStateStore` | No | **Default.** |
| `DatabaseConnectorExecutionStateStore` | Yes | Table `payos_connector_execution_state`, mirrors `DatabaseIdempotencyStore`'s find-then-save-or-update pattern. **Not wired in by default** — swapping to it is a deployment-wiring decision an operator makes explicitly; no `connectors.json`/bootstrap key selects it. |

## 10. Terminal routing — DLQ vs. terminal Connector State (Epic 5.6)

Once an execution reaches a terminal `FAILED` state (§9) — either because the error category
was never retryable, or because the retry budget (§8) was exhausted — the platform decides
whether to route it to `DLQ` or leave it as terminal `CONNECTOR_STATE`
(`ma.s2m.payos.config.connector.ConnectorTerminalDestination`). Like the retry policy, this is
Java-code-only today: accessed via `ConnectorTerminalRoutingPolicies.getInstance()`, no
`connectors.json`/bootstrap key.

`ConnectorTerminalRoutingPolicy.defaultPolicy()`:

| Error category | Destination | Rationale |
| --- | --- | --- |
| `PERMANENT_ERROR`, `UNKNOWN_ERROR` | `DLQ` | Never retryable to begin with, or unclassifiable — needs operator inspection. |
| `RETRYABLE_ERROR`, `TIMEOUT` (only reachable here after retry-budget exhaustion) | `CONNECTOR_STATE` | The platform *did* attempt automatic recovery; this run simply gave up — no escalation needed. |

> **There is no real DLQ.** No dead-letter-queue infrastructure exists anywhere in this
> codebase or its sibling repos. "Routes to DLQ" means only that a
> `ConnectorTerminalRoutingRecord(connectorType, connectorName, tenantId, correlationId,
> idempotencyKey, attemptCount, errorCategory, destination, reason, recordedAt)` is written
> with `destination=DLQ` to `IConnectorTerminalRoutingStore` (default
> `InMemoryConnectorTerminalRoutingStore`, accessed via `ConnectorTerminalRoutingStores` — no
> database-backed variant exists, unlike the execution-state store in §9). Operators must not
> assume a message actually lands on a real queue/broker; consuming or replaying "DLQ" entries
> is not built.

## 11. Retry/DLQ diagnostics (Epic 5.7)

A dedicated observability event category — distinct from the PCI-DSS audit trail
(`IAuditLogger`/`AuditLogger`, see [system-events.md](../reference/system-events.md)) — surfaces
every retry and terminal-routing decision for incident triage, without requiring debug logging.

`ma.s2m.payos.connector.diagnostics.Slf4jConnectorDiagnosticsRecorder` (the default and only
shipped implementation, swappable via the static facade `ConnectorDiagnostics`) logs one JSON
line **at `WARN`** per event to the SLF4J logger category **`CONNECTOR_DIAGNOSTICS`** —
operators should route/filter this category independently in their logback configuration, the
same way `AUDIT` is routed separately (see [operations/observability.md](../operations/observability.md)).

`ConnectorDiagnosticEvent` fields: `stage` (`RETRY_SCHEDULED` | `TERMINAL_ROUTING`),
`connectorType`, `connectorName`, `tenantId`, `correlationId`, `errorCode`,
`rootCauseCategory`, `attemptCount`, `destination` (null unless `stage=TERMINAL_ROUTING`),
`reason`, `recordedAt`. Like the execution-state record, it never carries payload/parameter
data — there is nothing to mask.

A diagnostic event is "linked" to the corresponding execution-state (§9) and terminal-routing
(§10) records by the same composite key (`correlationId`, `connectorType`, `connectorName`) —
there is no separate cross-reference field; look the key up in those stores directly.

## 12. Putting it together — the full `execute()` decision sequence

`ConnectorScriptHandle.execute(Value)` is the single choke point for every behavior in §6–§11,
always in this order:

1. **Idempotency-key precondition** (§6.2 tail) — reject with `CONNECTOR_IDEMPOTENCY_KEY_REQUIRED` if required and missing.
2. **Deduplication gate** (§6.2) — `REPLAY`/`SUPPRESS` short-circuit here; only `EXECUTE` continues.
3. Record `RUNNING` execution state (§9).
4. **Invoke the connector.**
5. On success → record `SUCCEEDED`; on failure → classify the error category (§7), then:
6. **Retry decision** (§8) → record `RETRYING` and emit a `RETRY_SCHEDULED` diagnostic (§11), **or**
7. **Terminal routing decision** (§10) → record `FAILED` execution state, write the terminal-routing record, and emit a `TERMINAL_ROUTING` diagnostic (§11).

Audit events (`CONNECTOR_EXECUTION` for `CardNetwork`/`Switch` connector types only, and
`CONNECTOR_DUPLICATE_DETECTED` on replay/suppress) are emitted alongside this sequence but are
a separate mechanism — see [system-events.md](../reference/system-events.md).

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
- `payos-connector-sdk/src/main/java/ma/s2m/payos/connector/model/ConnectorExecutionContext.java`, `IdempotencyContext.java`, `ConnectorErrorCategory.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorConfigurationLoader.java`, `ConnectorConfigurationEntry.java`, `ConnectorCredentialReferenceResolver.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorRuntimeSettings.java`, `ConnectorRuntimeSettingsEvaluator.java`, `ConnectorRuntimeReloader.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/TenantConnectorRegistry.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorDeduplicationGate.java`, `ConnectorDeduplicationDecision.java`, `ConnectorDeduplicationAction.java`, and `payos/src/main/java/ma/s2m/payos/idempotency/ConnectorIdempotencyStores.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorRetryPolicy.java`, `ConnectorRetryPolicies.java`, `ConnectorRetryContext.java`, `ConnectorRetryDecision.java`.
- `payos/src/main/java/ma/s2m/payos/connector/state/` — `ConnectorExecutionState.java`, `ConnectorExecutionStateRecord.java`, `IConnectorExecutionStateStore.java`, `InMemoryConnectorExecutionStateStore.java`, `DatabaseConnectorExecutionStateStore.java`, `ConnectorExecutionStateStores.java`, `ConnectorTerminalRoutingRecord.java`, `IConnectorTerminalRoutingStore.java`, `InMemoryConnectorTerminalRoutingStore.java`, `ConnectorTerminalRoutingStores.java`.
- `payos/src/main/java/ma/s2m/payos/config/connector/ConnectorTerminalDestination.java`, `ConnectorTerminalRoutingDecision.java`, `ConnectorTerminalRoutingPolicy.java`, `ConnectorTerminalRoutingPolicies.java`.
- `payos/src/main/java/ma/s2m/payos/connector/diagnostics/` — `ConnectorDiagnosticEvent.java`, `IConnectorDiagnosticsRecorder.java`, `ConnectorDiagnostics.java`, `Slf4jConnectorDiagnosticsRecorder.java`.
- `payos/src/main/java/ma/s2m/payos/scripting/ConnectorScriptHandle.java` — the wiring point tying §6–§11 together.
- `payos/_bmad-output/implementation-artifacts/` — Epic 1–5 connector framework stories (`1-1-*.md` through `5-7-*.md`), in particular 5-2's Addendum for `requiresIdempotencyKey`.
- [env-var-resolution.md](env-var-resolution.md) — full `${...}` placeholder grammar.
- [extensions-connectors.md](extensions-connectors.md) — the unrelated legacy SPI-backend loader.
- [developer/scripting-bindings.md](../developer/scripting-bindings.md) — the `$Connector` script binding itself.
