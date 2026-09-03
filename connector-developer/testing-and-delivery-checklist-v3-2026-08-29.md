# Testing a connector and pre-delivery checklist

Created: 2026-07-27
Last updated: 2026-08-29
Version: v3

## 1. Testing without a full PayOS runtime

`ConnectorTestHarness` (module `payos-connector-sdk`, package `ma.s2m.payos.connector.test`) lets you call `init`/`execute`/`close` directly in a plain Java (or JUnit) test, without any running PayOS runtime — the fastest way to exercise a connector's own logic in isolation, and still useful even now that `$Connector` is wired into `BootServer` (see [README.md](README.md)), since it doesn't require assembling a full `connectors.json` + runtime directory just to test your `init`/`execute`/`close` implementation.

```java
import ma.s2m.payos.connector.test.ConnectorTestHarness;
import ma.s2m.payos.connector.model.ConnectorConfig;
import ma.s2m.payos.connector.model.ConnectorExecutionContext;
import ma.s2m.payos.connector.model.ConnectorResponse;
import java.util.Map;

class SampleConnectorTest {

    @Test
    void executeReturnsSuccess() throws Exception {
        var harness = ConnectorTestHarness.forConnector(new SampleConnector())
                .init(new ConnectorConfig("PaymentGateway", "sample", Map.of("endpointUrl", "https://api.example.test")));

        ConnectorExecutionContext context = new ConnectorExecutionContext(
                "acme", "corr-123", "charge", 1, null, Map.of());

        ConnectorResponse response = harness.execute(context, Map.of("amount", 100));

        assertThat(response.status()).isEqualTo(ConnectorStatus.SUCCESS);
        assertThat(response.data()).containsEntry("accepted", true);

        harness.close();
    }
}
```

Notes on the harness's actual behavior (read directly from `ConnectorTestHarness.java`):
- `forConnector(connector)` throws `IllegalArgumentException` if `connector` is `null` — construct your connector instance yourself and hand it in.
- `init(config)` throws `IllegalArgumentException` if `config` is `null`, otherwise delegates straight to your `init(...)` — any `ConnectorInitializationException` you throw propagates unchanged.
- `execute(context, payload)` throws `IllegalArgumentException` if `context` is `null`; a `null` `payload` is normalized to an empty, immutable map (`Map.of()`) rather than passed through as `null` — your `execute(...)` implementation never has to null-check `payload` itself. The harness also defensively copies a non-null `payload` (`Map.copyOf(...)`), so mutating the map you passed in after the call has no effect on what your connector already received.
- `close()` delegates straight to your `close()`.
- The harness does **not** simulate deduplication, retry, terminal routing, or audit logging — those are platform behaviors layered on top of `IConnector` by the runtime itself (see [writing-a-connector-v1-2026-07-27.md §6](writing-a-connector-v1-2026-07-27.md#6-idempotency-platform-owned-not-connector-owned)), not something the harness reproduces. Use it to verify your own `init`/`execute`/`close` logic and error-category choices in isolation, not the platform's retry/DLQ behavior around it.

## 2. Checklist before delivery

The first six items below (descriptor, SPI registration, `connector-sdk` scope, forbidden dependencies/imports, approved external dependencies) are exactly what [connector-certification-v1-2026-08-29.md](connector-certification-v1-2026-08-29.md)'s `ConnectorCertificationCli` checks automatically, given your built jar and `pom.xml` — run it before working through this list by hand.

- `getType()`/`getName()` match exactly `connector.type`/`connector.name` in the descriptor.
- `META-INF/connector.properties` present, with `connector.type`, `connector.name`, `connector.api.version` filled in.
- `META-INF/services/ma.s2m.payos.connector.api.IConnector` present, containing the fully-qualified name of your single implementation.
- `connector-sdk` declared in `provided` scope, never bundled/shaded into the delivered jar.
- No dependency on any other `ma.s2m.payos:*` artifact, no import of `ma.s2m.payos.*` outside `ma.s2m.payos.connector.*`.
- Every bundled third-party (non-`ma.s2m.payos`) library is on the approved dependencies registry (see [external-dependency-approval-v5-2026-08-29.md §4](external-dependency-approval-v5-2026-08-29.md#4-the-approved-dependencies-registry)) — request approval before delivery if it isn't.
- Every foreseeable error (timeout, network failure, functional rejection) is converted to `ConnectorResponse.error(...)` or a `ConnectorExecutionException` with an explicit category — nothing is left to surface as an uncontrolled technical exception if a useful message for the caller is expected.
- `connector.requires.idempotency` set to `true` if your external integration requires it.
- `close()` actually releases every resource opened by `init(...)`.
- Exercised end-to-end with `ConnectorTestHarness` (§1) covering at least: successful execution, each error category you can realistically produce, and (if applicable) the required-params validation path.

## Next

- [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md) — the contract this checklist verifies.
- [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) — descriptor, SPI registration, `connectors.json`.
- [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md) — how to get a bundled third-party library approved before delivery.
- [connector-certification-v1-2026-08-29.md](connector-certification-v1-2026-08-29.md) — run `ConnectorCertificationCli` to check most of §2 automatically.
