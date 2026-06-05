# Scripting engine

PayOS business logic is JavaScript executed by `PolyglotScriptingEngine`
(`ma.s2m.payos.scripting.graalvm`) on top of **GraalVM Polyglot**. This document describes
the engine's architecture and, most importantly, its **security sandbox** — the structural
expression of the *secure by design* principle.

## Interfaces and factory

`ma.s2m.payos.scripting.IScriptingEngine` is the contract:

```java
Object executeScript(String script, Object... params);                  // params include the Request
Object executeScript(String script, String sourceUri, Object... params); // sourceUri aids debugging
void   evalScript(String script);                                       // no function contract (hooks)
void   evalScript(String script, String sourceUri);
Object getMember(String name);
void   putMember(String name, Object value);                            // inject a binding
Object loadLibrary(String name, String script);                         // load a lib/ module
```

Engine type constants: `ENGINE_NASHORN`, `ENGINE_GRAALVM_POLYGLOT`. The factory `ScriptingEngineFactory.createNewEngine(type)` returns a `PolyglotScriptingEngine` for `ENGINE_GRAALVM_POLYGLOT` (the engine used by the API pipeline).

## Engine vs. context

GraalVM distinguishes a shared, thread-safe **Engine** from a per-use, single-threaded **Context**. PayOS uses both deliberately:

| Element | Scope | Thread-safety | Purpose |
| --- | --- | --- | --- |
| `SHARED_ENGINE` (`static Engine`) | process-wide | thread-safe | Shares the parsed AST and JIT code cache across all requests. |
| `Context` | per request | **not** thread-safe | Enforces the sandbox and isolates request state. |
| `SOURCE_CACHE` (`ConcurrentHashMap<String,Source>`) | process-wide | thread-safe | Caches parsed `Source` objects, keyed by `"{length}:{hashCode}"`. |
| `MAPPER` (`ObjectMapper`) | process-wide | thread-safe | Shared Jackson mapper. |

Sharing the engine and source cache keeps the hot path fast; using a fresh context per request keeps tenants and requests isolated.

## The sandbox

`buildContext()` configures the GraalVM context to **deny by default** and grant only the
minimum needed:

| Setting | Value | Effect |
| --- | --- | --- |
| `allowAllAccess` | `false` | Nothing is allowed unless explicitly granted. |
| `allowHostAccess` | `HostAccess.ALL` | Scripts may call **public** methods on the host objects explicitly injected as bindings (`$Request`, `$DB`, …). |
| `allowHostClassLookup` | predicate | `Java.type()` is allowed except for blocked classes (e.g. `java.lang.System` is denied). |
| `allowIO` | `IOAccess.NONE` | No file I/O from scripts. |
| `allowCreateThread` | `false` | Scripts cannot spawn threads. |
| `allowNativeAccess` | `false` | No JNI / native calls. |
| `allowCreateProcess` | `false` | No process execution. |
| `hostClassLoader` | extension classloader | When set, `Java.type()` can reach extension JARs (see [extensibility.md](extensibility.md)). |

Consequences for application authors:

- You interact with the platform **only** through the injected `$` bindings and through whitelisted `Java.type()` classes — there is no ambient access to the filesystem, network, threads, or processes.
- This is what makes it safe to run many tenants' business logic in one JVM.

## Execution flow

```
executeScript(script, sourceUri, request)
  ├─ Source source = getOrCreateSource(script, sourceUri, "payos-api-script")   ← cached
  ├─ context.eval(source)                                  ← define functions in the context
  ├─ executeFn       = bindings.getMember("execute")
  ├─ loadControlFn   = bindings.getMember("loadControlData")
  ├─ emitInsightFn   = bindings.getMember("emitInsight")
  ├─ controlData = loadControlFn(request)                  ← Map<String,Object>
  ├─ result      = executeFn(request, controlData)         ← Response | Map | List
  ├─ response    = wrap result in Response if needed
  ├─ payload     = parse response body as JSON when possible
  ├─ insight     = emitInsightFn(request, response, payload) ← Map | List | scalar | null
  ├─ publish insight to queue if non-null and a queue client is configured
  └─ return response
```

The contract an API script must satisfy (`loadControlData` + `execute` + `emitInsight`) is documented for developers in [writing-apis.md](../developer/writing-apis.md).

## Bindings

Bindings are host objects placed into the context with `putMember`. The API pipeline injects the standard set before execution (see [request-processing.md](request-processing.md#binding-injection)). Each binding is a plain object whose **public** methods scripts may call thanks to `HostAccess.ALL`. Optional service bindings (`$DB`, `$Queue`, `$Secrets`, `$WebHooks`) appear only when the corresponding service is configured.

The developer-facing reference for every binding is [developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Libraries

`loadLibrary(name, script)` and the `$Library` binding (`LibraryProxy` over a `LibraryHandler`) let scripts reuse shared JavaScript modules from the application's `lib/` directory, so common helpers are written once per application. See [developer/writing-apis.md](../developer/writing-apis.md).

## Hooks use the engine too

Hook scripts run through `evalScript` (no `execute`/`loadControlData` contract) with the same sandbox. The `HookEngine` drives them at the hook points listed in [eventing-webhooks.md](eventing-webhooks.md).

## Performance notes

- Parsing is amortized by the `SOURCE_CACHE`; a script string is parsed at most once.
- The JIT-compiled code is shared through `SHARED_ENGINE`, so repeated executions of the same script benefit from prior compilation.
- Each request still gets a fresh `Context`, so there is no cross-request state leakage.

## Next

- [Security architecture](security-architecture.md) — authentication and the principal
  exposed as `$Principal`.
- [Extensibility](extensibility.md) — how `Java.type()` reaches extension classes.
