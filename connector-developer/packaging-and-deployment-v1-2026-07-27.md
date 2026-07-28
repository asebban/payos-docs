# Packaging and deploying a connector

Created: 2026-07-27
Last updated: 2026-07-27
Version: v1

This page covers everything between "I have a working `IConnector` implementation" (see [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md)) and "the operator can drop my JAR into a running PayOS deployment."

## 1. The descriptor (`META-INF/connector.properties`)

**Do not confuse this with the PayOS capability system's `manifest.json`** (`.capabilities/<id>/manifest.json`, a completely different, unrelated mechanism used by `payos-pm`/`cpm` for a different kind of plugin). A connector uses a plain Java properties file, at the fixed location `META-INF/connector.properties` inside the jar:

```properties
connector.type=PaymentGateway
connector.name=cmi
connector.api.version=1.0
connector.required.params=merchantId,apiKey,endpointUrl
connector.requires.idempotency=true
```

| Key | Required | Purpose |
| --- | --- | --- |
| `connector.type` | Yes | Must match exactly what `getType()` returns. |
| `connector.name` | Yes | Must match exactly what `getName()` returns. |
| `connector.api.version` | Yes | Must match the API version your implementation targets — currently `"1.0"` (`IConnector.CONNECTOR_API_VERSION`; see §4 for the distinction from the SDK's own Maven version). The runtime checks compatibility at jar-scan time and rejects an incompatible connector before ever instantiating it. |
| `connector.required.params` | No (comma-separated list) | Keys you require inside `ConnectorConfig.parameters()`. The runtime checks their presence in the matching `connectors.json` entry before instantiating your connector — you don't need to re-validate their presence yourself in `init(...)`. |
| `connector.requires.idempotency` | No (default `false`) | See [writing-a-connector-v1-2026-07-27.md §6](writing-a-connector-v1-2026-07-27.md#6-idempotency-platform-owned-not-connector-owned) — this is the flag that guarantees the platform will never call `execute(...)` without an idempotency key if set to `true`. |

## 2. SPI registration (`META-INF/services`)

The runtime discovers your implementation via `java.util.ServiceLoader`, not a class name declared in the descriptor. Your jar must contain a plain text file at this exact path:

```
META-INF/services/ma.s2m.payos.connector.api.IConnector
```

whose content is the fully-qualified name of your implementation class, for example:

```
com.example.connectors.cmi.CmiPaymentConnector
```

Your jar must declare **exactly one** `IConnector` implementation — zero declared providers, or more than one, fails loading that connector (the runtime stays operational for the rest of the application, but that specific connector is never put into service).

## 3. Deployment (`connectors.json`)

**Packaging:** a connector is a standalone `.jar` containing your compiled classes, `META-INF/connector.properties` (§1), and `META-INF/services/ma.s2m.payos.connector.api.IConnector` (§2). `connector-sdk` must be a Maven dependency in `provided` scope — **never** bundle/shade it into your jar (the runtime already supplies it at execution time, and a static certification check detects and rejects a jar that bundles it). Likewise, your connector must not depend on any other `ma.s2m.payos:*` artifact besides `connector-sdk`, and must not import any `ma.s2m.payos.*` package outside `ma.s2m.payos.connector.*`.

**Runtime isolation:** each connector is loaded in its **own classloader**, isolated from other connectors and from the rest of the runtime (except the PayOS core classloader, which remains accessible as a parent). Two connectors can therefore bundle different versions of the same third-party library without conflict.

**Deployment:** place the jar somewhere the runtime can reach, then declare it in `connectors.json` (at the runtime's base directory), with an entry shaped like:

```json
{
  "type": "PaymentGateway",
  "name": "cmi",
  "jar": "/path/to/cmi-connector.jar",
  "parameters": {
    "merchantId": "...",
    "apiKey": "...",
    "endpointUrl": "https://api.cmi.example/v1"
  }
}
```

At startup (or on a hot reload), the runtime scans the jar, validates the descriptor and API-version compatibility, checks that every declared `connector.required.params` is present in `parameters`, then instantiates and initializes the connector. A validation or initialization failure never crashes the runtime — the affected connector simply moves to the `FAILED` state and is never exposed to scripts, while the rest of the application keeps working normally.

For the full operator-facing configuration surface — every `connectors.json` field, API-version compatibility policy details, retry/DLQ parameters, diagnostics — see [configuration/connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md). That page also documents the split between what you (the connector author) declare in the descriptor versus what the operator declares in `connectors.json`.

## 4. Versioning

Two distinct version numbers, not to be confused. The **Maven version** of the `ma.s2m.payos:connector-sdk` artifact (currently `1.2.0-RELEASE`) is what you declare as a `provided` dependency in your `pom.xml` — it evolves with the SDK's own features (e.g. the addition of `requiresIdempotencyKey`). The **runtime API version** (`IConnector.CONNECTOR_API_VERSION`, currently `"1.0"`) is what you declare in the descriptor's `connector.api.version` — it's checked at load time for compatibility between your connector and whichever runtime executes it, independent of which SDK Maven version you compiled against.

## Next

- [testing-and-delivery-checklist-v1-2026-07-27.md](testing-and-delivery-checklist-v1-2026-07-27.md) — verify your connector before shipping it, and the full pre-delivery checklist.
- [configuration/connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md) — the complete operator-side configuration reference.
