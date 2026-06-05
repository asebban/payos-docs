# Getting started

Build PayOS from source, run it locally, and create your first application.

## Prerequisites

| Tool | Version |
| --- | --- |
| JDK | 21 |
| Maven | 3.9+ |

Confirm your toolchain:

```bash
java -version    # should report 21
mvn -version
```

## Build the runtime

The runnable server is produced by the `payos-runtime` module, which shades the kernel, the transport servers, and the bundled services into a single executable JAR.

```bash
# from the workspace root, build the runtime (and the modules it depends on)
mvn -q -DskipTests -f payos-runtime/pom.xml package
```

The artifact is written to `payos-runtime/target/payos-runtime-<version>.jar` with the main class `ma.s2m.payos.BootServer`. (For the multi-module build and versioning, see [build-and-release](../build-and-release/README.md).)

## Create a bundle

A **bundle** is the directory the runtime runs against, rooted at `payos.json`. A minimal
bundle:

```
my-bundle/
├── payos.json
├── config/
│   └── bootstrap.json
└── app/
    ├── config/
    │   └── config.json
    └── api/
        └── greet.js
```

`payos.json` (bundle entrypoint — it points to the configuration directory):

```json
{
  "configDir": "config"
}
```

`config/bootstrap.json` (runtime configuration — see the [configuration reference](../configuration/bootstrap-reference.md)):

```json
{
  "servers": [
    { "protocol": "http", "host": "0.0.0.0", "port": 8080 }
  ],
  "applications": [
    { "id": "hello", "name": "Hello", "base.path": "../apps/hello", "version": "1.0.0" }
  ],
  "multitenancy": {
    "tenantSimulator": { "enabled": true, "tenantId": "dev-tenant" }
  }
}
```

> The `tenantSimulator` is enabled here so you can call the API without sending `X-Tenant-Id`.
> **Never enable it in production** — see [multi-tenancy](../architecture/multi-tenancy.md).
> Note that `base.path` is resolved relative to `configDir`; since `bootstrap.json` lives in
> `config/` and the app is in `apps/hello/`, we use `../apps/hello`.

`apps/hello/api/greet.js`:

```javascript
function loadControlData(request) {
    return {};
}

function execute(request, controlData) {
    $Response.setStatusCode(200);
    return { message: "Hello from PayOS", tenant: $Tenant };
}

function emitInsight(request, response, payload) {
  return null;
}
```

## Run

```bash
java -jar payos-runtime/target/payos-runtime-<version>.jar --bundle-path ./my-bundle
```

The server starts the configured HTTP listener. Built-in endpoints such as `/health` are
available immediately (see [reference/http-endpoints.md](../reference/http-endpoints.md)).

## Call your API

```bash
# health check
curl http://localhost:8080/health

# your API
curl -X POST http://localhost:8080/hello/api/greet
```

You should receive:

```json
{ "message": "Hello from PayOS", "tenant": "dev-tenant" }
```

## What just happened

1. The HTTP transport received `POST /hello/api/greet`, opened a (pre-auth) tenant scope,
   and built a `Request`.
2. The kernel resolved the `hello` application and located the `greet` API resource.
3. The [scripting engine](../architecture/scripting-engine.md) ran `loadControlData`,
   `execute`, and `emitInsight`, with `$Response` and `$Tenant` injected as bindings.
4. The returned object from `execute` was serialized as the HTTP response, with `X-Tenant-Id`
   and `X-Correlation-Id` headers added. (`emitInsight` returned `null`, so no insight was
   published.)

The full path is described in
[architecture/request-processing.md](../architecture/request-processing.md).

## Next

- [Application model](application-model.md) — how applications are structured and registered.
- [Writing API scripts](writing-apis.md) — the script contract in depth.
- Want to add config, a database, secrets, or i18n? Continue through the
  [developer guide](README.md).
