# Plugin Development

## Purpose

TCP plugins allow this module to support real wire protocols without modifying `payos-server-tcp` itself.

A plugin JAR can provide any combination of:
- `TcpMessageDecoder`
- `TcpMessageEncoder`
- `TcpMessageHandler`

If a plugin omits one of them, the provider falls back to the built-in default implementation for the missing part.

## Interfaces

### Decoder

```java
public interface TcpMessageDecoder {
    Request decode(java.io.InputStream in) throws Exception;
}
```

Use this when you need to:
- parse frames
- decode binary payloads
- map a protocol envelope to a PayOS request path or method

### Encoder

```java
public interface TcpMessageEncoder {
    void encode(Response response, java.io.OutputStream out) throws Exception;
}
```

Use this when you need to:
- return binary payloads
- serialize a protocol envelope
- add framing markers, lengths, or checksums

### Handler

```java
public interface TcpMessageHandler {
    Response handle(String appId, Request request) throws Exception;
}
```

Use this when you need to:
- perform protocol-specific dispatch before the kernel
- enrich the request before business execution
- completely override the default `super.processRequest` delegation

## Minimal Plugin Example

### Decoder example

```java
package com.example.payos.tcp;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import ma.s2m.payos.resources.IResource;
import ma.s2m.payos.servers.exchanges.Request;
import ma.s2m.payos.servers.tcp.TcpMessageDecoder;

public class SampleDecoder implements TcpMessageDecoder {
    @Override
    public Request decode(InputStream in) throws Exception {
        byte[] data = in.readAllBytes();
        String body = new String(data, StandardCharsets.UTF_8);
        return new Request(Request.METHOD_POST, IResource.API_RESOURCE, null, null, "/app2/api/orders", body);
    }
}
```

### Encoder example

```java
package com.example.payos.tcp;

import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import ma.s2m.payos.servers.exchanges.Response;
import ma.s2m.payos.servers.tcp.TcpMessageEncoder;

public class SampleEncoder implements TcpMessageEncoder {
    @Override
    public void encode(Response response, OutputStream out) throws Exception {
        if (response != null && response.getMessage() != null) {
            out.write((response.getMessage() + "\n").getBytes(StandardCharsets.UTF_8));
        }
    }
}
```

### Handler example

```java
package com.example.payos.tcp;

import ma.s2m.payos.servers.exchanges.Request;
import ma.s2m.payos.servers.exchanges.Response;
import ma.s2m.payos.servers.tcp.TcpMessageHandler;

public class SampleHandler implements TcpMessageHandler {
    @Override
    public Response handle(String appId, Request request) throws Exception {
        Response response = new Response();
        response.setStatusCode(200);
        response.setMessage("Handled by custom TCP plugin for appId=" + appId);
        return response;
    }
}
```

## Packaging Rules

Your plugin classes must:
- be concrete classes
- not be abstract
- expose a public no-argument constructor
- be packaged into a JAR placed in the configured plugin directory

No Java SPI registration file is required for these plugin classes, because `TcpServerProvider` scans the JAR contents reflectively.

## Discovery Rules

For each plugin JAR:
- every `.class` entry is inspected
- the first concrete implementation of each target interface is instantiated
- the scan stops early if decoder, encoder, and handler have all been found

This means one JAR can expose all three extension points, or only one of them.

## Recommended Design Guidelines

- keep framing, checksums, and protocol parsing inside the decoder/encoder
- keep business rules in the kernel or application scripts
- make the decoder responsible for setting a meaningful request path
- preserve UTF-8 explicitly when converting between bytes and strings
- avoid keeping mutable connection-specific state in plugin instances unless you control the threading model carefully

## Common Pitfalls

### Using the default path `/`

If your decoder returns `/`, application resolution may fail or route to the wrong application. Set a path that matches the target PayOS application and resource mapping.

### Reading unbounded streams

The default decoder uses `readAllBytes()`. If your protocol is long-lived or streaming, implement explicit framing instead of relying on end-of-stream semantics.

### Assuming handler is always called

The handler is optional. If no handler is found, the server falls back to kernel delegation. Plan your plugin set accordingly.

### Missing no-arg constructor

The provider instantiates plugin classes with reflection using `getDeclaredConstructor().newInstance()`. A missing no-argument constructor prevents loading.

## Testing Strategy

For a new plugin, test at three levels:

1. unit test the decoder and encoder with raw byte streams
2. integration test the plugin JAR loading path via the configured plugin directory
3. end-to-end test a real socket connection against a runtime with the kernel and target application loaded
