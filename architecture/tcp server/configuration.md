# Configuration

## Server Entry in `bootstrap.json`

The TCP module is activated through the `servers` array in the kernel bootstrap configuration.

Minimal example:

```json
{
  "servers": [
    {
      "protocol": "tcp",
      "host": "0.0.0.0",
      "port": 7070
    }
  ]
}
```

The entry is passed directly to `TcpServerProvider.create(serverConfig)` after the kernel resolves the protocol `tcp`.

## Supported Server Fields

| Field | Required | Meaning |
|---|---|---|
| `protocol` | yes | Must be `tcp` |
| `host` | no | Bind address, defaults to `127.0.0.1` if omitted by kernel bootstrap |
| `port` | no | Bind port, defaults to `8080` if omitted by kernel bootstrap |
| `tcp-handlers-dir` | no | Directory containing plugin JARs for decoder/encoder/handler overrides |

## Plugin Directory Resolution Order

`TcpServerProvider` resolves the plugin directory in this order:

1. Java system property `tcp-handlers-dir`
2. Environment variable `tcp-handlers-dir`
3. The current server configuration entry field `tcp-handlers-dir`

If none is present, the provider uses the built-in defaults.

## Example With Plugin Directory

```json
{
  "servers": [
    {
      "protocol": "tcp",
      "host": "0.0.0.0",
      "port": 7070,
      "tcp-handlers-dir": "/opt/payos/tcp-plugins"
    }
  ]
}
```

Equivalent JVM launch override:

```bash
java -Dtcp-handlers-dir=/opt/payos/tcp-plugins -jar target/payos-runtime-1.3.0-RELEASE.jar
```

## Plugin Loading Rules

When a plugin directory is configured:

- only `.jar` files are scanned
- every class in each JAR is inspected reflectively
- interfaces and abstract classes are ignored
- the first discovered concrete implementation of each of the following interfaces is selected:
  - `TcpMessageDecoder`
  - `TcpMessageEncoder`
  - `TcpMessageHandler`

If a JAR provides only one or two of these interfaces, the remaining pieces fall back to the built-in defaults.

## Operational Notes

- if the configured plugin directory does not exist, the server logs a warning and falls back to defaults
- if no JAR files are found, the server logs a warning and falls back to defaults
- if a JAR fails during loading, the provider skips it and continues scanning
- plugin changes are not hot-reloaded; restart the runtime after updating plugin JARs

## Configuration Examples by Use Case

### Local development

```json
{
  "servers": [
    {
      "protocol": "tcp",
      "host": "127.0.0.1",
      "port": 7070
    }
  ]
}
```

### Production with custom framing plugin

```json
{
  "servers": [
    {
      "protocol": "tcp",
      "host": "0.0.0.0",
      "port": 7070,
      "tcp-handlers-dir": "/srv/payos/plugins/tcp"
    }
  ]
}
```

## What the Module Does Not Configure

This module does not introduce transport-specific configuration for:
- TLS over TCP
- connection pooling
- socket read timeout tuning
- message framing size limits
- protocol version negotiation

If you need those capabilities, implement them in a custom decoder or handler plugin, or extend the module deliberately.
