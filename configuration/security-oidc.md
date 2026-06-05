# Security & OIDC configuration

The `security` block configures authentication (OIDC via pac4j or Nimbus), session
management, and CORS. The runtime model is in
[architecture/security-architecture.md](../architecture/security-architecture.md). Security
can be set globally and overridden per application.

## Provider selection

`SecurityServiceFactory.create()` resolves the provider in this order: **application config
→ session store → global config → default `pac4j`**.

| `provider` value | Implementation |
| --- | --- |
| `pac4j` (default) | `SecurityService` (pac4j OIDC). |
| `nimbus` | `NimbusSecurityService`. |

## OIDC keys

From `IConfigSpec.Security`:

```json
{
  "security": {
    "provider": "pac4j",
    "encryptionKey": "${PAYOS_SEC_KEY}",
    "oidcProviderBaseUrl": "https://idp.example.com",
    "keycloakBaseUrl": "https://idp.example.com",
    "realm": "payos",
    "discoveryUri": "https://idp.example.com/realms/payos/.well-known/openid-configuration",
    "clientId": "payos",
    "clientSecret": "${OIDC_CLIENT_SECRET}",
    "callBackUri": "https://app.example.com/callback",
    "scope": "openid profile email",
    "preferredJwsAlgorithm": "RS256",
    "logoutUrl": "https://idp.example.com/.../logout",
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
| `encryptionKey` | Key for session/value encryption. |
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
| `sessionMaxEntries` | Max concurrent sessions retained. |
| `sessionCookieSecure` | Mark the session cookie `Secure`. |

Sessions use `PayOSSessionStore`; the session cookie is `PAYOS_SESSION_ID` (HttpOnly,
SameSite=Lax, and Secure when configured/HTTPS). The authenticated principal is a map with
`id` (OIDC `sub`), `email`, `name`, `preferred_username`, and `roles` — exposed to scripts as
`$Principal`.

## CORS

`allowedOrigins` sets the permitted browser origins. CORS is resolved with a **three-level**
precedence: **application → tenant → global**, so an app or tenant can narrow or extend the
global policy. The HTTP transport handles the `OPTIONS` preflight automatically.

## Per-application overrides

An application's `security` object overrides the global block for that app — for example a
different `clientId`, `scope`, or `allowedOrigins`. See
[developer/application-model.md](../developer/application-model.md).

## Authorization in scripts

Resource access checks roles when the resource requires them; the authenticated principal's
`roles` drive authorization. From a script, read `$Principal.get("roles")`. See
[developer/scripting-bindings.md](../developer/scripting-bindings.md).

## Endpoints

The HTTP transport serves `/me` (current principal), `/callback` (OIDC redirect), and
`/logout`. See [reference/http-endpoints.md](../reference/http-endpoints.md).

## Next

- [Architecture: security](../architecture/security-architecture.md)
- [multi-tenancy.md](multi-tenancy.md)
