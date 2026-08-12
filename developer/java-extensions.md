# Java extensions

When JavaScript is not enough — you need a vetted library, a financial protocol stack (e.g.
jPOS for ISO 8583), or a CPU-bound routine — PayOS lets scripts call **Java** through
`Java.type()`, subject to the sandbox's class allow/deny rules. This page explains the
developer workflow; the security model is in
[architecture/scripting-engine.md](../architecture/scripting-engine.md) and the packaging of
extension JARs is in
[configuration/extensions-connectors.md](../configuration/extensions-connectors.md).

## When to use an extension

| Need | Mechanism |
| --- | --- |
| Reuse a Java library / protocol stack from a script | **Extension** (this page) |
| Provide a pluggable backend service (DB, queue, secrets, webhooks) | **Connector** (SPI) — see [extensibility](../architecture/extensibility.md) |
| Add a new wire protocol | **Transport provider** — see [extensibility](../architecture/extensibility.md) |

## Calling Java from a script

```javascript
function execute(request, controlData) {
    var BigDecimal = Java.type("java.math.BigDecimal");
    var total = new BigDecimal(controlData.amount).multiply(new BigDecimal("1.20"));
    return { totalWithTax: total.toPlainString() };
}
```

### What `Java.type()` can reach

The sandbox is **deny-by-default**:

- Public methods on **injected bindings** are always callable (`$DB`, `$Response`, …).
- `Java.type()` is allowed except for a **denylist** (e.g. `java.lang.System`).
- When an **extension classloader** is configured, `Java.type()` can also resolve classes
  from your extension JARs (the classloader is set as the context's host classloader).
- I/O, threads, native access, and process creation are blocked regardless.

See [architecture/scripting-engine.md](../architecture/scripting-engine.md) for the exact
`HostAccess` / lookup configuration.

## Packaging an extension

1. Build a JAR containing your classes (and shaded dependencies as needed).
2. Place it in the **extensions** directory.
3. PayOS exposes it to scripts through the **extension classloader**.

Resolution order for the extensions directory:

```
system property  →  PAYOS_EXTENSIONS_DIR env var  →  bootstrap key (extensions-dir)
```

Connectors (SPI backends) use the analogous `service-adapters-dir` / `PAYOS_SERVICE_ADAPTERS_DIR`. The
classloader hierarchy is: **application → extension → connector → runtime**. Details:
[configuration/extensions-connectors.md](../configuration/extensions-connectors.md).

## Example: an ISO 8583 helper (jPOS)

```javascript
function execute(request, controlData) {
    var ISOMsg = Java.type("org.jpos.iso.ISOMsg");
    var msg = new ISOMsg();
    msg.setMTI("0200");
    msg.set(2, controlData.pan);
    msg.set(4, controlData.amount);
    return { mti: msg.getMTI(), field2: msg.getString(2) };
}
```

This works once the jPOS classes are available via an extension Jar on the extensions path.

## Guidance

- Keep extensions small and focused; prefer connectors for backend services.
- Do not attempt to bypass the sandbox; denied operations are denied by design.
- Pin dependency versions inside your extension build.

## Next

- [Configuration: extensions & connectors](../configuration/extensions-connectors.md)
- [Architecture: scripting engine](../architecture/scripting-engine.md)
- [Architecture: extensibility](../architecture/extensibility.md)
