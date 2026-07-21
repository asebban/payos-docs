# Security & OIDC configuration

The `security` block configures authentication (OIDC via pac4j or Nimbus), session management, and CORS. The runtime model is in [architecture/security-architecture.md](../architecture/security-architecture.md). Security can be set globally and overridden per application.

## Provider selection

`SecurityServiceFactory.create()` resolves the provider in this order: **application config → session store → global config → default `nimbus`**.

| `provider` value | Implementation |
| --- | --- |
| `nimbus` (default) | `NimbusSecurityService` (Nimbus OIDC — primary). |
| `pac4j` (legacy) | `SecurityService` (pac4j OIDC). |

## OIDC keys

From `IConfigSpec.Security`:

```json
{
  "security": {
    "provider": "nimbus",
    "oidcProviderBaseUrl": "https://idp.example.com",
    "keycloakBaseUrl": "https://idp.example.com",
    "realm": "payos",
    "discoveryUri": "${config:security.oidcProviderBaseUrl}/realms/${config:security.realm}/.well-known/openid-configuration",
    "clientId": "payos",
    "clientSecret": "${OIDC_CLIENT_SECRET}",
    "callBackUri": "https://app.example.com/callback",
    "scope": "openid profile email",
    "preferredJwsAlgorithm": "RS256",
    "logoutUrl": "${config:security.oidcProviderBaseUrl}/.../logout",
    "postLogoutRedirectUri": "https://app.example.com/",
    "sessionTtlSeconds": 3600,
    "sessionMaxEntries": 10000,
    "sessionCookieSecure": true,
    "allowedOrigins": ["https://app.example.com"]
  }
}
```

| Key | Purpose |
| --- | --- |
| `provider` | `pac4j` or `nimbus`. |
| `oidcProviderBaseUrl` / `keycloakBaseUrl` | IdP base URL. |
| `realm` | IdP realm. |
| `discoveryUri` | OIDC discovery document URL. |
| `clientId` / `clientSecret` | OIDC client credentials. |
| `callBackUri` | Redirect URI registered with the IdP (served at `/callback`). |
| `scope` | Requested scopes. |
| `preferredJwsAlgorithm` | Expected token signing algorithm (e.g. `RS256`). |
| `logoutUrl` | IdP logout endpoint. |
| `postLogoutRedirectUri` | Where to send the user after logout. |

## Session management

| Key | Purpose |
| --- | --- |
| `sessionTtlSeconds` | Session lifetime. |
| `sessionMaxEntries` | Max concurrent sessions retained (only enforced by the in-memory backend — see below). |
| `sessionCookieSecure` | Mark the session cookie `Secure`. |
| `sessionStoreType` | Session storage backend: `memory` (default, zero external dependency) or `redis` (distributed, requires `session-service-redis` on the classpath — see [Distributed session storage](#distributed-session-storage) below). |

Sessions use `PayOSSessionStore`; the session cookie is `PAYOS_SESSION_ID` (HttpOnly, SameSite=Lax, and Secure when configured/HTTPS). The authenticated principal is a map with `id` (OIDC `sub`), `email`, `name`, `preferred_username`, and `roles` — exposed to scripts as `$Principal`.

### Distributed session storage

By default, sessions are held in an in-process map (`InMemorySessionStore`) — fine for a single node, but a login on one node is invisible to any other node behind the same load balancer without sticky sessions. Setting `sessionStoreType` to `redis` switches to a Redis-backed `ISessionStore` (module `session-service-redis`), so any node can read back a session created on another:

```json
{
  "security": {
    "sessionStoreType": "redis",
    "sessionStoreRedis": {
      "host": "127.0.0.1",
      "port": 6379,
      "password": "...",
      "database": 0,
      "tls": false,
      "keyPrefix": "payos:session:"
    }
  }
}
```

| `sessionStoreRedis.*` key | Default | Purpose |
| --- | --- | --- |
| `host` | `localhost` | Redis host. |
| `port` | `6379` | Redis port. |
| `password` | *(none)* | Redis `AUTH` password, omitted from the connection when blank. |
| `database` | `0` | Redis logical database index. |
| `tls` | `false` | Enables TLS on the Redis connection. |
| `keyPrefix` | `payos:session:` | Prefix applied to every session key, so the same Redis instance can later be shared with other distributed stores without collisions. |

Like the other pluggable backends (`queue-service-nats`, `secret-service-filesystem`), `session-service-redis` is resolved via `ServiceLoader` and must be on the runtime classpath (already the case for `payos-runtime`, which shades it in) — an unrecognized `sessionStoreType` fails explicitly at startup rather than silently falling back to memory. Backend selection is resolved once, at first access, and is not hot-reloaded — switching backends invalidates active sessions either way, so this is considered an acceptable restart-only change. Full design rationale in [`payos/docs/architects/session-store-redis-design.md`](../../payos/docs/architects/session-store-redis-design.md); module-level reference in [`session-service-redis/README.md`](../../session-service-redis/README.md).

## CORS

`allowedOrigins` sets the permitted browser origins. CORS is resolved with a **three-level** precedence: **application → tenant → global**, so an app or tenant can narrow or extend the global policy. The HTTP transport handles the `OPTIONS` preflight automatically.

## Per-application overrides

An application's `security` object overrides the global block for that app — for example a different `clientId`, `scope`, or `allowedOrigins`. See [developer/application-model.md](../developer/application-model.md).

## Authorization in scripts

Resource access checks roles when the resource requires them; the authenticated principal's `roles` drive authorization. From a script, read `$Principal.get("roles")`. See
[developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Endpoints

The HTTP transport serves `/me` (current principal), `/callback` (OIDC redirect), and `/logout`. See [reference/http-endpoints.md](../reference/http-endpoints.md).

## See further

For a fully more detailed explanation of security oidc functioning and configuration, see [oidc configuration guide](./oidc-configuration-guide.md)  

## Next

- [Architecture: security](../architecture/security-architecture.md)
- [multi-tenancy.md](multi-tenancy.md)
