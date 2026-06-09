# PayOS Queue Server Module

**Module:** `payos-server-queue`  
**Version:** `1.0.6-RELEASE`  
**Group:** `ma.s2m.payos`

## Overview

`payos-server-queue` is the queue transport server for PayOS. It consumes queue messages, converts them to the shared `Request` model, delegates business processing to `payos-kernel`, and optionally publishes a reply message.

This module does not implement broker communication directly. It depends on the `IQueueClient` abstraction provided by a standalone connector (for example `queue-service-nats`).

## Runtime Activation

The module is discovered via Java SPI when a `servers` entry uses protocol `queue`.

Example `bootstrap.json`:

```json
{
  "servers": [
    {
      "protocol": "queue",
      "host": "127.0.0.1",
      "port": 4222
    }
  ]
}
```

`QueueServerProvider` resolves the queue client type from queue configuration and creates `QueueServer`.

## Request/Response Envelope

Inbound messages are expected as JSON envelopes with fields such as:

- `method` (default `POST`)
- `type` (default `api`)
- `path` (default `/`)
- `body`
- `headers`
- `parameters`
- optional `tenantId`, `correlationId`, and `appId`

If `correlationId` is absent, the server generates one UUID and propagates it.

When a reply is requested by the broker client, response metadata is serialized to JSON with:

- `statusCode`
- `message`
- optional `body`
- `headers`
- `tenantId` and `correlationId` context

## Module Structure

```text
payos-server-queue/
├── pom.xml
└── src/main/
    ├── java/ma/s2m/payos/servers/
    │   ├── impl/QueueServer.java
    │   └── providers/QueueServerProvider.java
    └── resources/META-INF/services/
        └── ma.s2m.payos.servers.ServerProvider
```

## Dependencies

| Dependency | Purpose |
|---|---|
| `ma.s2m.payos:payos-kernel` | Shared server abstractions, request processing pipeline |
| `com.fasterxml.jackson.core:jackson-databind` | Queue envelope JSON parsing/serialization |

## Build

```bash
mvn -q -DskipTests compile
mvn -q test
mvn -q -DskipTests package
```
