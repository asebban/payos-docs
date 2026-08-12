# Connector developer guide

Documentation for a third-party or partner team writing a **business/payment connector** — a Java implementation of `IConnector` (module `connector-sdk`) that a PayOS runtime loads, isolates in its own classloader, and invokes from scripts via the `$Connector('Type', 'name')` binding (e.g. a payment gateway, a card network, a routing switch).

**Not to be confused with:** the older, unrelated *SPI connector* mechanism (`service-adapters-dir`, used for database/queue/secret-provider/webhook backend implementations such as `secret-service-vault` or `queue-service-nats`) — the two mechanisms share the word "connector" but are otherwise independent. If you're building one of those instead, see [integrators/extensibility-mechanisms.md §13.2](../integrators/extensibility-mechanisms.md#132-spi-connectors-service-adapters-dir).

> **Status note (updated 2026-07-27):** the business/payment connector framework (JAR scanning, descriptor validation, API-version compatibility, isolated instantiation, lifecycle tracking, per-tenant registry, deduplication, retry, terminal routing) is fully built, tested, **and wired into `BootServer`** since 2026-07-27 (`ma.s2m.payos.config.connector.ConnectorFrameworkInitializer`, called at boot and on hot reload). A correctly packaged connector dropped into `<runtimeBaseDir>/connectors/` and declared in `connectors.json` is reachable via `$Connector(...)` in a real deployment. See [developer/connector-framework-usage.md §6](../developer/connector-framework-usage.md#6-comment-connector-devient-disponible-et-son-modèle-de-tenant-scoping) for the tenant-scoping model (every tenant in `multitenancy.tenants` shares the same connector list — there's no per-tenant `connectors.json` entry).

## Documents

| Document | Covers |
| --- | --- |
| [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md) | The `IConnector` contract, the model types (`ConnectorConfig`, `ConnectorExecutionContext`, `ConnectorResponse`, ...), error-category semantics, the idempotency ownership split, and a complete minimal example (plain `IConnector` and via `AbstractConnector`). |
| [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) | The `META-INF/connector.properties` descriptor, `ServiceLoader` (SPI) registration, classloader isolation rules, `connectors.json` deployment, and the two independent version numbers you need to track. |
| [testing-and-delivery-checklist-v1-2026-07-27.md](testing-and-delivery-checklist-v1-2026-07-27.md) | Exercising your connector with `ConnectorTestHarness` before any real PayOS runtime is involved, and the checklist to run through before shipping a connector JAR. |

## Quick start

```java
public class ExampleConnector implements IConnector {

    private String endpointUrl;

    @Override
    public void init(ConnectorConfig config) throws ConnectorInitializationException {
        this.endpointUrl = (String) config.parameters().get("endpointUrl");
        if (endpointUrl == null || endpointUrl.isBlank()) {
            throw new ConnectorInitializationException("endpointUrl is missing from configuration");
        }
    }

    @Override
    public ConnectorResponse execute(ConnectorExecutionContext context, Map<String, Object> payload)
            throws ConnectorExecutionException {
        return ConnectorResponse.success(Map.of("accepted", true), context.correlationId(), context.tenantId());
    }

    @Override
    public void close() { /* release resources opened in init(...) */ }

    @Override
    public String getType() { return "PaymentGateway"; }

    @Override
    public String getName() { return "example"; }
}
```

Five methods, no more — see [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md) for the full contract, and why the interface is deliberately locked to exactly these five.

## Reference source

Everything in this section is verified directly against `payos-connector-sdk`'s source and against the host runtime (`payos`, packages `ma.s2m.payos.config.connector` / `ma.s2m.payos.scripting`) that executes connectors. The module's own README, [`payos-connector-sdk/README.md`](../../payos-connector-sdk/README.md) (French), is the original, most detailed version of this same material and stays in sync with it — treat the two as equivalent; use whichever language you're comfortable with.

## Next

- [developer/connector-framework-usage.md](../developer/connector-framework-usage.md) — the other side of the contract: how an **application script** resolves and calls a connector you've already written, via `$Connector(...)`.
- [configuration/connector-framework-parameters-v3-2026-08-11.md](../configuration/connector-framework-parameters-v3-2026-08-11.md) — the full operator-facing configuration reference (`connectors.json`, API-version compatibility policy, retry/DLQ parameters, diagnostics) for whoever deploys your connector.
- [integrators/extensibility-mechanisms.md §13.2bis](../integrators/extensibility-mechanisms.md#132bis-businesspayment-connector-framework-connector--not-to-be-confused-with-spi-connectors-above) — where this framework sits among PayOS's other extension points.
