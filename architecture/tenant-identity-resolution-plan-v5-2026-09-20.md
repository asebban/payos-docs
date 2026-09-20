
# Identity-first tenant resolution & centralized transport security — Implementation record

**Created:** 2026-09-18
**Last updated:** 2026-09-20
**Version:** 5 — supersedes v4 (2026-09-20): adds §9, a related but distinct fix in `DynamicDataAccessService`/`ApiResourceHandler` found while reviewing `TenantScope`'s save-and-restore pattern — the ambient DB tenant context (`CURRENT_TENANT`) was being cleared unconditionally on nested `$Api.*` calls, not restore-safe the way `TenantScope`/MDC already was.

## 1. What changed since v2

v2 planned two things: identity-first tenant priority, and moving scope-opening into `Server.processRequest`. v3 implemented both, plus a third thing added in that round: **principal resolution and API-key (Bearer token) authentication are now also centralized in `Server.processRequest`**, uniformly across every transport, with only the interactive Keycloak login/callback flow left as HTTP-specific (it inherently requires a browser to redirect). This revision (v4) fixes a design flaw in how `TenantPolicyService` obtained the principal, caught in review — see §3.4. The guiding principle applied throughout remains: `Server.processRequest` owns everything common to every transport; each transport server (`HttpServer`, `TcpServer`, `QueueServer`) keeps only what is genuinely specific to it.

## 2. Target behavior (unchanged since v1)

For every transport, in this exact order:

1. **Principal present (authenticated request) → the tenant is always derived from the principal's realm/`tenantId` claim.** A client-supplied `X-Tenant-Id` header is never consulted once a principal is present.
2. **Principal absent:**
   a. Tenant simulator enabled → simulated tenant wins outright.
   b. Otherwise → `X-Tenant-Id` header, then request `contextData`, then MDC.

**Any request carrying an `Authorization: Bearer ...` header must resolve to a valid principal, or be rejected with `401` immediately** — uniformly, on every transport, regardless of whether the resource it's about to reach even requires roles.

## 3. What was actually implemented

### 3.1 `payos-foundation`

- **`IServer`** (`ma/s2m/payos/servers/IServer.java`): a new default method:
  ```java
  default boolean supportsInteractiveLogin() {
      return false;
  }
  ```
  This single hook answers two questions with one transport characteristic ("can this transport complete a browser-redirect login"): whether an anonymous request gets a lenient pre-auth tenant scope (`Server.processRequest`), and whether a role-gated resource is even allowed to attempt `ISecurityService.authenticate()` (`ApiResourceHandler`, `VueResourceHandler`).
  <br>*(v3 also added `CONTEXT_PRINCIPAL`/`CONTEXT_SECURITY_SERVICE` context-data key constants here — removed again in this revision; see §3.4.)*
- **`ISecurityService`**: added a pure, non-authenticating default method:
  ```java
  default String resolveTenantId(Map<String, Object> principal) {
      Object tenantId = principal == null ? null : principal.get("tenantId");
      return tenantId != null && !tenantId.toString().isBlank() ? tenantId.toString() : null;
  }
  ```
  The counterpart to `resolveAuthenticatedTenantId(Request)` for a caller that already holds a resolved principal and just needs its tenant read back off it, without re-touching the request/session.

### 3.2 `payos` kernel — `security/oidc/nimbus/NimbusSecurityService.java` and `security/oidc/pac4j/SecurityService.java`

- Nimbus's existing *private* `resolveTenantId(Map<String, Object> principal)` was promoted to `@Override public` — it already did the right thing, it just wasn't reachable through the interface.
- pac4j's `toPrincipalMap` did not previously put a `tenantId` entry on the principal map — added `user.put("tenantId", extractRealmFromOidcProfile(oidcProfile));`, so the interface's default `resolveTenantId(Map)` now works correctly for pac4j too, with no override needed there.
- **Operational consequence worth knowing, not a bug**: pac4j's `getCurrentPrincipal` has no bearer-token path at all (only session-based). Under the uniform rule in §2, **a pac4j-configured tenant cannot accept `Authorization: Bearer` API-key traffic at all** — such a request is rejected at `Server.processRequest`, because pac4j will never resolve a principal from it. Arguably correct per the stated policy (an unverifiable credential must not be silently ignored), but a real capability gap between the two `ISecurityService` implementations worth knowing about operationally.

### 3.3 `payos` kernel — `servers/Server.java`

`processRequest` now:

1. Resolves `Application` (the same null-safe `try/catch (ApplicationException)` `HttpServer.resolveApplication` used to have — that method is gone from `HttpServer`, folded in here).
2. Builds `ISecurityService` and resolves the principal, once, for every request on every transport — **for this method's own two purposes only** (the bearer-credentials gate below, and the strict-vs-lenient scope choice): it is not stashed anywhere for another class to read back — see §3.4 for why.
3. Rejects with `401` immediately if the request carries bearer credentials but no principal resolved — before any tenant scope opens, before any resource is located.
4. Decides strict vs. lenient tenant scope from `principal == null && supportsInteractiveLogin()`, opens the corresponding scope, and catches `TenantPolicyException` right there, turning it into a normal error `Response` rather than letting it propagate — every transport's own `catch (TenantPolicyException e)` block was removed as a result; it would now be unreachable dead code.
5. Delegates to a new, overridable `dispatch(String appId, Application application, Request request)` protected method (default: `ResourceHandler.getHandler(request.getType()).handle(application, request)`) instead of inlining that dispatch directly — this is what lets `TcpServer` keep its pluggable-handler extensibility point (§3.5) without needing its own separate scope-opening call.

### 3.4 `payos` kernel — `multitenancy/TenantPolicyService.java`, corrected in this revision

**v3's mistake, caught in review**: `enforceAndOpenScope`'s principal-resolution read `request.getContextData().get(IServer.CONTEXT_PRINCIPAL)` — a value populated earlier, in the same request, by `Server.processRequest`. Asked directly: *"is this really how the principal should be retrieved — shouldn't it come from the session?"* The honest answer is that the session-based retrieval was happening correctly, just one hop away (`Server.processRequest`'s own `securityService.getCurrentPrincipal(request)` — which for pac4j goes through `ProfileManager`+`sessionStore`, and for Nimbus checks a stateless bearer token, then the session). Reading a stashed copy of that result was pure plumbing to avoid re-authenticating twice per request. But it made `enforceAndOpenScope`'s correctness **depend on which caller ran first**: `TenantPolicyServiceTest` already calls it directly, with no `Server.processRequest` involved at all, and any future caller could do the same — silently falling back to the client-header chain even for a genuinely authenticated request, with no error to reveal the mistake. That is exactly the class of bug already deliberately avoided for `ApiResourceHandler`/`VueResourceHandler` (§3.6), for a different but related reason (nested `$Api.*` calls bypass `Server.processRequest` too) — v3 avoided it there but introduced it here, inconsistently.

**Fix**: `resolveTenantIdentity` (private, inside `TenantPolicyService`) now resolves `Application`, `ISecurityService`, and the principal **itself**, directly — the same pattern `ApiResourceHandler` already uses:
```java
private static String resolveTenantIdentity(Request request, String appId) throws TenantPolicyException {
    Application application = resolveApplication(appId);
    ISecurityService securityService = SecurityServiceFactory.create(application, request);
    Map<String, Object> principal = securityService.getCurrentPrincipal(request);
    if (principal == null) {
        return resolveNoPrincipalTenantId(request, true);
    }
    String tenantId = securityService.resolveTenantId(principal);
    if (tenantId != null && !tenantId.isBlank()) {
        return tenantId;
    }
    throw new TenantPolicyException(Response.STATUS_FORBIDDEN, "...");
}
```
`resolveApplication(String appId)` is a small private helper added to `TenantPolicyService`, identical in shape to the one already in `Server`. The now-unused `CONTEXT_PRINCIPAL`/`CONTEXT_SECURITY_SERVICE` constants and the stashing calls in `Server.processRequest` were removed as dead code rather than left behind unused.

**Cost accepted, deliberately**: this is now a *third* principal resolution in the worst case for an authenticated request reaching a role-gated resource — `Server.processRequest` resolves one (for the bearer gate and the scope choice), `TenantPolicyService.enforceAndOpenScope` resolves another (for the tenant), and `ApiResourceHandler`/`VueResourceHandler` resolve a third (for the role check) — three separate `getCurrentPrincipal` calls, i.e. up to three session-store round trips or JWT validations per request. This is the same trade-off already accepted elsewhere in this change (HTTP's `isAuthenticated` double-check, `ApiResourceHandler`'s own independent resolution): correctness and "works regardless of caller" over shaving a redundant lookup. If this shows up in profiling, the fix would be a proper request-scoped cache with a clear, single owner and an explicit contract — not a same-request stash-and-hope-the-right-thing-ran-first, which is what this revision removes.

Everything else in `TenantPolicyService` from v3 is unchanged: `resolveNoPrincipalTenantId(Request, boolean strict)` (header → simulator → MDC, shared between the strict and lenient callers), `openPreAuthTenantScope` (promoted from `HttpServer`, unchanged in spirit), and the fail-closed behavior when a principal exists but carries no derivable tenant.

### 3.5 `payos-server-http`, `payos-server-tcp`, `payos-server-queue`

Unchanged from v3:

- **`HttpServer`**: overrides `supportsInteractiveLogin()` → `true`; `processInVirtualThread` collapsed to one plain call to `processRequest`. One accepted, minor side effect: a `TenantPolicyException`-derived error response now flows through `sendSuccessResponse`'s header/content-type logic instead of the old dedicated `sendErrorResponse` path — content-type for that one response shape changes from `text/plain` to `application/json`; not worth a special case to preserve byte-for-byte identical output there.
- **`TcpServer`**: `dispatch(...)` is overridden to delegate to a nullable `customHandler` (a real, classpath-discovered extensibility point — see `TcpServerProvider`, and `tcp/src/main/java/ma/s2m/server/MyHandler.java`) when present, else `super.dispatch(...)` — so a custom `TcpMessageHandler` still runs inside the tenant scope `Server.processRequest` opens, with no separate scope-opening call left in `TcpServer` at all.
- **`QueueServer`**: calls `processRequest(appId, request)` plainly; `TenantPolicyException` import kept for an unrelated, pre-existing use in `buildErrorResponseMessage`.

### 3.6 `payos` kernel — `resources/api/ApiResourceHandler.java` and `resources/vue/VueResourceHandler.java`

Unchanged from v3: both still resolve their own principal independently, deliberately not reading any contextData — nested endpoint-to-endpoint calls (`$Api.get/post/...`) build a fresh `Request` and bypass `Server.processRequest` entirely, so a stashed value would be empty there. Both gained an `interactiveLoginSupported` guard (`application.getServer().supportsInteractiveLogin()`): on a transport with no browser to redirect, a request with no principal now fails closed with a plain `403` instead of being handed whatever `ISecurityService.authenticate()`/`check()`'s internal fallback would have produced.

## 4. Downstream validation stays untouched

`validateKnownTenant`, `validateQuota`, and `mustRequireTenant` in `TenantPolicyService` are unchanged and still apply to whichever tenant won, regardless of source.

## 5. Verification performed

- `payos-foundation`, `payos`, `payos-server-http`, `payos-server-tcp`, `payos-server-queue` all compile cleanly (`mvn -o compile`/`install`) after every edit in this record, including the v4 correction.
- `payos`'s full test suite, re-run after the v4 fix: **557 tests, 0 failures, 0 errors**, including all 12 pre-existing `TenantPolicyServiceTest` cases passing unmodified — now genuinely exercising a fresh, independent `getCurrentPrincipal` call each time (visible in the test log: repeated "No session cookie found" debug lines from `PayOSSessionStore`), not a contextData short-circuit.
- **Still not done**: `payos-server-http`, `payos-server-tcp`, and `payos-server-queue` have zero existing tests; none of the scenarios in §6 were exercised by an automated test, only by compilation and by reasoning through the code paths.

## 6. Testing gap — recommended follow-up (not yet done)

- A `Server`-level test asserting: a bearer-credentialed request with no resolvable principal is rejected before `dispatch()` runs; the strict vs. lenient scope choice matches `supportsInteractiveLogin()`.
- A `TcpServer` test asserting a custom `TcpMessageHandler` still runs inside an open tenant scope.
- `TenantPolicyService` cases exercising `resolveTenantIdentity` with a real principal present (needs the test seam noted below, or a fake `ISecurityService`/session) — principal wins over a conflicting header; principal with no derivable tenant → `403`; no-principal MDC fallback.
- A test seam for `TenantPolicyService` (mirroring `quotaCounter`/`resetForTests()`) so principal-present scenarios can be unit-tested without a real OIDC session — genuinely more valuable now that resolution is direct rather than a stashed value a test could just inject via `contextData`.
- An `ApiResourceHandler`/`VueResourceHandler` case per transport confirming the `interactiveLoginSupported` gate.

## 7. Documentation to update

- [multi-tenancy.md](multi-tenancy.md) — resolution order and the ingress table both still describe per-transport scope-opening; update to `Server.processRequest` as the single call site, and the priority rule from §2.
- [ADR-0004](adr/0004-structural-multi-tenancy.md) — "every transport must implement scope opening" is listed as a cost under "Negative"; this change removes it.
- `payos-server-http/docs/request-lifecycle.md`, `architecture.md`, `security.md`, `api-contracts.md` — all describe `HttpServer` calling `TenantPolicyService`/`resolveAuthenticatedTenantId` directly; point at `Server.processRequest` instead.
- `payos-server-tcp`'s docs on `TcpMessageHandler`, if any, should note it now runs inside the shared tenant scope via `Server.dispatch()`.

## 8. Rollout

Implemented and verified in one pass; the v4 correction in §3.4 was folded into the same change rather than shipped separately, since it touches the same lines v3 had just written. Remaining before this is production-ready: the tests in §6, and the documentation in §7.

## 9. Related fix: `DynamicDataAccessService`'s ambient tenant context wasn't restore-safe

Found while explaining why `TenantScope.open()`/`close()` save and restore the *previous* MDC values instead of unconditionally clearing them — a question naturally leads to: is anything else thread-bound and request-scoped handled the same way? One thing is, and correctly: `DynamicDataAccessService`'s `requestScopeDepth`/`requestScopedSession` (`beginRequestScope()`/`endRequestScope()`), which uses a depth counter rather than save-and-restore, because it needs to support real reentrancy (nested `$Api.*` calls on the same thread), not just thread-pool-reuse safety. One thing wasn't: the same class's `CURRENT_TENANT` (a static `ThreadLocal<String>`, `setCurrentTenant`/`clearCurrentTenant`) — and `ApiResourceHandler.handle()` was calling it asymmetrically with its own `beginRequestScope`/`endRequestScope` pair right next to it:

```java
if (currentTenantId != null && !currentTenantId.isBlank()) {
    databaseService.setCurrentTenant(currentTenantId);
}
databaseService.beginRequestScope();   // depth-guarded
...
finally {
    databaseService.endRequestScope();    // depth-guarded
    databaseService.clearCurrentTenant(); // NOT depth-guarded — ran on every exit, nested or not
}
```

**The bug**: endpoint1 calls endpoint2 via `$Api.post(...)` (`ApiProxy.executeWithFallback` runs a fresh `ApiResourceHandler` synchronously on the same thread). When endpoint2's `handle()` returns, its `finally` unconditionally cleared `CURRENT_TENANT` — even though endpoint1 is still running and may call `$DB` again afterward. Not observable in practice, because `DynamicDataAccessService.resolveOptionalCurrentTenantId()` falls back to the MDC tenant when `CURRENT_TENANT` is empty, and the MDC tenant is untouched by nested calls (only `Server.processRequest`'s single `TenantScope` touches it) — so the bug was silently absorbed by an unrelated fallback path rather than actually fixed.

**Fix**: `IDatabaseService.beginRequestScope()`'s contract changed from `void` to `boolean`, returning whether *this* call is the depth 0→1 owner (`false` for a nested reentry). `DynamicDataAccessService.beginRequestScope()` implements this (session-backed mode, which has no depth tracking at all, always returns `true` — unchanged behavior for that mode). `ApiResourceHandler.handle()` now calls `beginRequestScope()` *before* `setCurrentTenant(...)` (reordered — see below) and only calls `setCurrentTenant`/`clearCurrentTenant` when it owns the scope:

```java
boolean isTenantContextOwner = false;
if (databaseService != null) {
    isTenantContextOwner = databaseService.beginRequestScope();
    if (isTenantContextOwner && currentTenantId != null && !currentTenantId.isBlank()) {
        databaseService.setCurrentTenant(currentTenantId);
    }
}
...
finally {
    if (databaseService != null) {
        databaseService.endRequestScope();
        if (isTenantContextOwner) {
            databaseService.clearCurrentTenant();
        }
    }
```

**Why reordering `beginRequestScope()` before `setCurrentTenant()` is safe**: `beginRequestScope()`'s own internal tenant resolution (to open the right tenant's session, on the owning call) goes through the same `resolveOptionalCurrentTenantId()` MDC fallback described above — and MDC is already correctly set by the time `ApiResourceHandler.handle()` runs at all, because `Server.processRequest` opens `TenantScope` (which sets MDC) before ever dispatching to a `ResourceHandler`. So the session still opens for the correct tenant even though `CURRENT_TENANT` itself isn't set until one line later.

**Deliberately not touched**: `HibernateNotificationStore.withTenant(tenantId, action)` (`payos-service-notification`), the other production caller of `setCurrentTenant`/`clearCurrentTenant`, keeps its own unconditional set-then-clear — it's a different, legitimate pattern (temporarily switch to an *explicitly given*, possibly different tenant for one sub-operation, always self-contained within its own `try`/`finally`, never spanning a nested `$Api.*` call), and gating it the same way `ApiResourceHandler` now is would have broken it: it needs to actually switch tenant even when called while another scope is already active on the thread. Changing `setCurrentTenant`/`clearCurrentTenant` themselves to be depth-aware — rather than fixing only `ApiResourceHandler`'s own call site — was considered and rejected for exactly this reason.

**Verification**: `payos-foundation`, `database-service`, `payos`, `payos-server-http`/`tcp`/`queue`, and `payos-service-notification` all rebuilt clean (`mvn -o clean ...`) and re-tested — `database-service`: 23/23, `payos`: 557/557, `payos-service-notification`: 158/158 (including `HibernateNotificationStoreTest`, confirming its untouched pattern still works). One operational note from this verification pass, unrelated to the fix itself: a stale/incorrectly-compiled `target/classes` (apparently left by a background IDE compiler using a classpath that didn't see the freshly-reinstalled `payos-foundation` jar) caused spurious "Unresolved compilation problem" test failures until `mvn clean` forced a real recompile — worth knowing if a similar false failure shows up after a foundation-interface change in the future.
