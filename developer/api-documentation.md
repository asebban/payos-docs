# API documentation (`@payos.openapi`)

PayOS generates OpenAPI 3.1 specifications **statically** from annotations embedded in your
API scripts, using the [`pdoc`](../cli-tools/pdoc.md) tool. Generation never executes your
code, database, queues, or webhooks — it only parses `@payos.openapi` annotations. This page
covers how to annotate; the tool itself is documented in
[cli-tools/pdoc.md](../cli-tools/pdoc.md). For the complete consolidated reference, see
[PayOS OpenAPI documentation guide](openapi-docs/openapi-documentation-guide.md).

## Why static generation

`pdoc` is designed for **regulated, auditable delivery**: producing API documentation must not have side effects. It scans `api/**/*.js` for annotations and emits a spec — nothing runs.  (`pdoc` runtime-safety guarantees are detailed in the tool docs.)

## Annotating an endpoint

Add a `@payos.openapi` annotation block in a comment above (or associated with) the script's
operation:

```javascript
/**
 * @payos.openapi
 * summary: Create a payment
 * operationId: createPayment
 * tags: [payments]
 * requestBody:
 *   required: true
 *   content:
 *     application/json:
 *       schema:
 *         type: object
 *         properties:
 *           amount: { type: number }
 *           currency: { type: string }
 * responses:
 *   "201": { description: Created }
 *   "400": { description: Validation error }
 * /@payos.openapi
 */
function execute(request, controlData) {
    $Response.setStatusCode(201);
    return { id: controlData.id };
}
```

The annotation body follows OpenAPI Operation semantics. `pdoc` assembles per-endpoint operations into a complete OpenAPI 3.1 document, deriving paths from the application/resource layout.

The detailed annotation format, validation rules, generation flow, and regulated delivery
contract are described in [PayOS OpenAPI documentation guide](openapi-docs/openapi-documentation-guide.md).

## Generating the spec

```bash
pdoc --app payments --bundle-path ./my-bundle --output ./openapi
```

`pdoc` can target an `--app`, `--capability`, or `--product`. See
[cli-tools/pdoc.md](../cli-tools/pdoc.md) for all flags and the annotation format reference.

## Where the spec is served

The HTTP transport can serve an OpenAPI document and a Swagger UI:

- `/openapi.yaml` — the spec.
- `/swagger/**` — the UI (local-only by default; see
  [configuration/servers.md](../configuration/servers.md)).

Point the server at your generated spec via the `swaggerUI.openapi-yaml` setting.

## Next

- [Configure and use swagger](./swagger.md) — How to configure and use swagger to document APIs
- [PayOS OpenAPI documentation guide](openapi-docs/openapi-documentation-guide.md) — consolidated `pdoc` authoring, validation, generation, and delivery reference.
- [CLI: pdoc](../cli-tools/pdoc.md) — full annotation format and flags.
- [Configuration: servers (Swagger UI)](../configuration/servers.md)
