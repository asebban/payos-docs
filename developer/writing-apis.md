# Writing API scripts

API scripts live in an application's `api/` directory and are executed by the [GraalVM scripting engine](../architecture/scripting-engine.md) for every matching request. This page describes the **script contract**, the request/response model, error handling, and shared libraries.

Vous pourrez trouver [ici -> javascript-api-endpoint-guide](./javascript-api-endpoint-guide.md) des explications plus détaillées sur l'écriture de scripts JS.

## The three-function contract

The engine drives each API script through three well-known functions:

```javascript
function loadControlData(request) {
    // Phase 1: gather everything the request needs (lookups, validation inputs).
    // Return an object; it is passed to execute() as controlData.
    return { /* control data */ };
}

function execute(request, controlData) {
    // Phase 2: perform the business logic and produce the response body.
    return { /* response body */ };
}

function emitInsight(request, response, payload) {
    // Phase 3: emit observability or business insight from the completed response.
    // Return null when there is nothing to publish.
    return null;
}
```

- `loadControlData(request)` runs first. Use it to load reference data and pre-compute inputs.
  It may be empty (`return {};`).
- `execute(request, controlData)` runs second and returns the response payload.
- `emitInsight(request, response, payload)` runs after the response has been produced and normalized. Use it for analytics, business events, or observability payloads derived from the request and response. It may return `null`.

In the current runtime implementation, all three functions must exist and be executable. If you do not need insight emission, define `emitInsight` and return `null`.

The return value of `execute` may be:

- a plain JSON-compatible object/array → serialized as the response body and wrapped in a `Response` automatically, or
- a `Response` object (when you need full control of status, headers, and body).

Returning a plain object is the usual application pattern:

```javascript
function execute(request, controlData) {
    return { id: controlData.id, status: "APPROVED" };
}
```

The runtime accepts objects/maps and arrays/lists from `execute`. It does not accept `null` or scalar values (such as a bare string or number) as the final response; use a JSON object or return an explicit `Response` instead.

The return value of `emitInsight` may be a map/object, list/array, string, number, boolean, or `null`. When it returns a non-null value and a queue client is configured, the runtime serializes that value to JSON and publishes it to the configured queue client on a best-effort basis. Publishing failures are logged and do not interrupt the request flow.

## The `request` argument

`request` is the transport-neutral [`Request`](../architecture/request-processing.md) for the current call. Common accessors:

| Access | Returns |
| --- | --- |
| `request.getBody()` / `request.getBodyAsString()` | Raw / string body. |
| `request.getHeaders()` | Header map. |
| `request.getParameters()` | Query/form parameters. |
| `request.getMethod()` | HTTP-style method. |
| `request.getPath()` | Resource path. |
| `request.getContextData()` | Context map (tenant, correlation, appId). |

The same script runs unchanged whether the request arrived over HTTP, TCP, or a queue.

`getHeaders()` returns a Java `Map<String, String>`, not a plain JS object — iterate it with
`entrySet()` in a `for...of` loop:

```javascript
for (var entry of request.getHeaders().entrySet()) {
    $Logger.info(entry.getKey() + ": " + entry.getValue());
}
```

## Producing a response

The simplest pattern is to return an object and set the status through `$Response`:

```javascript
function execute(request, controlData) {
    $Response.setStatusCode(201);
    $Response.setHeader("Location", "/payments/api/payment/" + controlData.id);
    return { id: controlData.id, status: "CREATED" };
}
```

`$Response` is the injected response binding (see [scripting bindings](scripting-bindings.md)). Status/headers set on `$Response` are merged
into the transport response.

## Reading input

```javascript
function loadControlData(request) {
    var payload = JSON.parse(request.getBodyAsString() || "{}");
    if (!payload.amount) {
        throw new BusinessException("amount is required");
    }
    return { amount: payload.amount };
}
```

## Error handling

Throwing from `loadControlData`, `execute`, or `emitInsight` routes the request through the error pipeline (`API_ON_ERROR` hook + `api.on-error` webhook). A thrown `BusinessException` is **unwrapped** into a structured error response; other exceptions become a generic error while still being
logged with the request's correlation and tenant IDs (see [request-processing](../architecture/request-processing.md)).

```javascript
function execute(request, controlData) {
    if (controlData.amount <= 0) {
        throw new BusinessException("amount must be positive");
    }
    // ...
}
```

## Using shared libraries

Code shared across scripts lives in `lib/` and is loaded with `$Library` (`loadLibrary`-backed):

```javascript
// lib/money.js
function format(amount, currency) { return amount.toFixed(2) + " " + currency; }

// api/quote.js
var money = $Library.load("money");
function execute(request, controlData) {
    return { display: money.format(12.5, "EUR") };
}
```

## Bindings available in your script

Inside `loadControlData`, `execute`, and `emitInsight` you have access to the injected bindings — `$Request`, `$Response`, `$Api`, `$App`, `$Principal`/`$User`, `$Tenant`, and (when configured) `$DB`, `$Queue`, `$Secrets`, `$I18n`, `$WebHooks`. Each is documented in [scripting bindings reference](scripting-bindings.md).

## Documenting the endpoint

Add `@payos.openapi` annotations so [`pdoc`](../cli-tools/pdoc.md) can generate an OpenAPI spec — see [API documentation](api-documentation.md).

## Next

- [Scripting bindings reference](scripting-bindings.md)
- [Data access](data-access.md) · [Secrets usage](secrets-usage.md) · [Queue messaging](queue-messaging.md)
