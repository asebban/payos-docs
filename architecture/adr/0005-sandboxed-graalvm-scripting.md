# ADR-0005 — Sandboxed GraalVM scripting, deny-by-default

**Status:** Accepted

## Context

Business logic is authored in JavaScript and runs in a shared runtime that hosts many
tenants. Executing untrusted-by-default business code in-process is only acceptable if the
script cannot reach the filesystem, network, threads, processes, or sensitive host classes.

## Decision

Run scripts in a **GraalVM Polyglot** context configured **deny-by-default**:

- `allowAllAccess(false)`; grant only `allowHostAccess(HostAccess.ALL)` so scripts may call
  public methods on **explicitly injected** host bindings.
- `allowIO(IOAccess.NONE)`, `allowCreateThread(false)`, `allowNativeAccess(false)`,
  `allowCreateProcess(false)`.
- `allowHostClassLookup` permits `Java.type()` except for a blocklist (e.g.
  `java.lang.System`); when an extension classloader is set, `Java.type()` can reach
  whitelisted extension classes.
- Share a process-wide `Engine` and `Source` cache for performance, but use a **fresh
  per-request `Context`** for isolation.

Scripts interact with the platform exclusively through `$`-prefixed bindings and whitelisted
`Java.type()` classes.

## Consequences

- **Positive:** many tenants' code runs safely in one JVM; no ambient I/O; fast hot path via
  shared engine + source cache; no cross-request state leakage.
- **Negative:** capabilities that need real I/O must be provided as host bindings,
  connectors, or vetted extensions; some JavaScript libraries that assume Node.js APIs will
  not run.
- See [scripting-engine.md](../scripting-engine.md) and [extensibility.md](../extensibility.md).
