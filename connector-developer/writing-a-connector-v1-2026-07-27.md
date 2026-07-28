# Writing a connector — the `IConnector` contract

Created: 2026-07-27
Last updated: 2026-07-27
Version: v1

This page covers the Java contract you implement, the model types you receive and return, and how errors are classified. For packaging/deployment, see [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md); for exercising your implementation before any runtime is involved, see [testing-and-delivery-checklist-v1-2026-07-27.md](testing-and-delivery-checklist-v1-2026-07-27.md).

## 1. Overview

A connector is a Java implementation of `ma.s2m.payos.connector.api.IConnector`, packaged in its own `.jar`, and run by the PayOS runtime in an isolated classloader. The runtime discovers your implementation through the standard Java `ServiceLoader` mechanism (SPI) — never by reflecting on a class name declared somewhere in configuration. Once loaded and initialized, a connector is invoked from JavaScript running in the runtime's scripting engine (via a `$Connector('Type', 'name')` object on the script side), and every script-level call becomes exactly one call to `execute(...)` on your implementation. All deduplication, retry, terminal routing (DLQ), and audit logging is handled by the platform — your connector only has to answer `execute(...)` correctly and nothing else.

## 2. The `IConnector` contract

Full interface (`ma.s2m.payos.connector.api.IConnector`), deliberately minimal and locked by an SDK-side regression test that guarantees it will never grow another entry point (see §5 below for why):

```java
package ma.s2m.payos.connector.api;

public interface IConnector {

    String CONNECTOR_API_VERSION = "1.0";

    void init(ConnectorConfig config) throws ConnectorInitializationException;

    ConnectorResponse execute(ConnectorExecutionContext context, Map<String, Object> payload)
            throws ConnectorExecutionException;

    void close();

    String getType();

    String getName();
}
```

- `init(ConnectorConfig)` is called exactly once by the runtime, right after instantiation, before any `execute(...)` call. This is where you validate the configuration you received and open a reusable connection (HTTP client, etc.). If initialization fails (`ConnectorInitializationException`), the connector moves to the `FAILED` state and is never exposed to scripts — the rest of the PayOS runtime keeps working normally; this is not fatal to the process.
- `execute(context, payload)` is called once per logical execution requested by a script. `payload` is whatever the script passed (amount, reference, ...), already deserialized as a `Map<String, Object>`. This is the only method that contains your real business logic.
- `close()` is called exactly once at runtime shutdown (or on a hot reload of the connector) — release your resources (connections, thread pools, ...) here.
- `getType()` / `getName()` identify the connector (e.g. `"PaymentGateway"` / `"cmi"`) — these values must match exactly what you declare in the descriptor (see §1 of [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md)).

## 3. The model types

All of the following are immutable Java `record`s with validation in their compact constructor — passing an invalid value throws `IllegalArgumentException` immediately, never a deferred failure later in processing.

**`ConnectorConfig(String type, String name, Map<String, Object> parameters)`** — passed to `init(...)`. `type` and `name` are required (non-blank). `parameters` comes from the matching entry in the operator's `connectors.json` (see [packaging-and-deployment-v1-2026-07-27.md §3](packaging-and-deployment-v1-2026-07-27.md#3-deployment-connectorsjson)) — this is where you receive your API credentials, endpoint URL, timeouts, etc. Its `toString()` automatically masks common sensitive-looking field names (API key, secret, password) so it's safer to log by accident — don't rely on this for real secret handling, but it avoids accidental leakage into debug logs.

**`ConnectorExecutionContext(String tenantId, String correlationId, String operation, Integer attempt, IdempotencyContext idempotencyContext, Map<String, String> metadata)`** — passed to every `execute(...)` call. `correlationId` is **required** (the constructor throws if blank) — use it consistently in your own logs so they correlate with runtime traces. `attempt` is the attempt number (1 for the first try, incremented by the platform on retry — see §4). `idempotencyContext` is nullable and read-only (see §6) — you may read its key to forward as an idempotency header to an external provider, but you have no influence over the deduplication decision itself.

**`IdempotencyContext(String key, Map<String, String> metadata)`** — `key` required non-blank when the object is present.

**`ConnectorResponse(ConnectorStatus status, Map<String, Object> data, ConnectorErrorCategory errorCategory, String errorCode, String errorMessage, String correlationId, String tenantId)`** — the return value of `execute(...)`. Never construct this directly: use the static factories `ConnectorResponse.success(data, correlationId, tenantId)` and `ConnectorResponse.error(category, code, message, correlationId, tenantId)`, or the protected `success(...)`/`error(...)` helpers `AbstractConnector` provides (§7) which fill `correlationId`/`tenantId` in from the context automatically.

**`ConnectorStatus`** — two-value enum: `SUCCESS`, `ERROR`.

**`ConnectorErrorCategory`** — five-value enum: `NONE` (success responses only), `RETRYABLE_ERROR`, `PERMANENT_ERROR`, `TIMEOUT`, `UNKNOWN_ERROR`. This category drives the platform's retry behavior (§4) — choose it carefully.

## 4. Error handling

**Signal success:** `return ConnectorResponse.success(data, context.correlationId(), context.tenantId());` (or `success(context, data)` with `AbstractConnector`).

**Signal failure:** two equivalent ways. Either return `ConnectorResponse.error(category, code, message, correlationId, tenantId)` directly without throwing, or throw a `ConnectorExecutionException` using the full constructor, whose `rootCauseCategory` parameter must be **exactly** one of the strings `"RETRYABLE_ERROR"`, `"PERMANENT_ERROR"`, or `"TIMEOUT"` (exact, case-sensitive match with `ConnectorErrorCategory`'s enum names):

```java
throw new ConnectorExecutionException(
    getName(), getType(), context.tenantId(), context.correlationId(),
    "CONNECTOR_TIMEOUT", "TIMEOUT", "The external endpoint did not respond in time");
```

If `rootCauseCategory` is absent, blank, or doesn't match one of the three names above (typo, stale value, ...), the runtime silently converts it to `UNKNOWN_ERROR` without failing the execution — this isn't a hard failure for you, but it degrades the platform's retry classification, so avoid typos.

**Choosing the right category** entirely drives what the platform does next, on its own side (you implement nothing for this): `RETRYABLE_ERROR` and `TIMEOUT` are automatically retried (up to 3 attempts by default) before being routed to a state queue if the retry budget is exhausted; `PERMANENT_ERROR` is never retried and goes straight to the dead-letter queue (DLQ); `UNKNOWN_ERROR` is handled conservatively (requires operator inspection).

**If your code throws an uncaught exception** (an ordinary `RuntimeException`, not a `ConnectorExecutionException`), the runtime always catches it: your real exception (message, class, stack trace) is logged **server-side only** with full detail, but the caller (the script) never sees any of that — just a fully generic `ConnectorResponse.error(UNKNOWN_ERROR, "CONNECTOR_UNCAUGHT_EXCEPTION", "Connector execution failed unexpectedly", ...)`, with no internal detail exposed. This is a deliberate protection against leaking sensitive information to the calling script, not a bug to work around. Practical consequence: if you want the caller to receive a useful error code or message, catch your own internal exceptions and build a `ConnectorExecutionException` yourself with an `errorCode`/`message` meant to be seen by the caller — never rely on a technical exception propagating naturally to carry useful information. An unhandled exception in your connector never corrupts the runtime's execution thread: the very next call, even to a different connector, keeps working normally right after.

## 5. Why the interface is locked to exactly five methods

`IConnector` is deliberately locked to five methods precisely so a connector can never register its own deduplication store or influence that decision — a deliberate architecture choice, not an SDK gap. See §6 below and [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) for the descriptor flag that opts into this behavior.

## 6. Idempotency: platform-owned, not connector-owned

**Your connector never implements deduplication itself.**

What the platform does before it ever calls your `execute(...)`: if the descriptor declares `connector.requires.idempotency=true` and no non-blank idempotency key was supplied by the caller, the execution is rejected immediately with `PERMANENT_ERROR` / `CONNECTOR_IDEMPOTENCY_KEY_REQUIRED` — your code is never invoked in that case. Otherwise, if an idempotency key is present, the platform consults a dedicated store: a never-seen key triggers a normal execution; a key already in progress fails the call immediately with `RETRYABLE_ERROR` / `CONNECTOR_DUPLICATE_IN_PROGRESS` without ever calling your connector; a key already completed successfully returns the previously recorded response directly, again without re-invoking your connector.

What remains your responsibility: declare `connector.requires.idempotency=true` in your descriptor if your external integration (bank, card network) itself requires an idempotency key to avoid a double charge, and read `context.idempotencyContext().key()` if you want to forward it as-is in an HTTP header to your external provider. Nothing else — no cache, lock, or duplicate-detection logic to write yourself.

## 7. Complete minimal example

The simplest possible version, implementing `IConnector` directly:

```java
package com.example.connectors.sample;

import ma.s2m.payos.connector.api.IConnector;
import ma.s2m.payos.connector.model.*;
import ma.s2m.payos.connector.exception.ConnectorInitializationException;
import ma.s2m.payos.connector.exception.ConnectorExecutionException;
import java.util.Map;

public class SampleConnector implements IConnector {

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
        try {
            // ... real call to endpointUrl, using payload and context.correlationId() ...
            return ConnectorResponse.success(Map.of("accepted", true), context.correlationId(), context.tenantId());
        } catch (Exception e) {
            throw new ConnectorExecutionException(
                getName(), getType(), context.tenantId(), context.correlationId(),
                "CONNECTOR_CALL_FAILED", "RETRYABLE_ERROR", "External call failed");
        }
    }

    @Override
    public void close() {
        // release your resources here if needed
    }

    @Override
    public String getType() {
        return "PaymentGateway";
    }

    @Override
    public String getName() {
        return "sample";
    }
}
```

Recommended variant, building on `AbstractConnector` (an optional base class that implements `getType()`/`getName()` from a `ConnectorDescriptor`, and provides `success(...)`/`error(...)` shortcuts that fill in `correlationId`/`tenantId` automatically):

```java
public class SampleConnector extends AbstractConnector {

    public SampleConnector() {
        super(new ConnectorDescriptor("PaymentGateway", "sample", "1.0", List.of("endpointUrl")));
    }

    @Override
    public void init(ConnectorConfig config) { /* ... */ }

    @Override
    public ConnectorResponse execute(ConnectorExecutionContext context, Map<String, Object> payload) {
        return success(context, Map.of("accepted", true));
    }

    @Override
    public void close() { /* ... */ }
}
```

## Next

- [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) — descriptor, SPI registration, classloader isolation, `connectors.json`, versioning.
- [testing-and-delivery-checklist-v1-2026-07-27.md](testing-and-delivery-checklist-v1-2026-07-27.md) — exercising this contract with `ConnectorTestHarness`, and the pre-delivery checklist.
