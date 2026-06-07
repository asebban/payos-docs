# Environment Variable Resolution

**Purpose**: Inject runtime secrets and environment-specific values into configuration without storing them in files.

## Overview

PayOS configuration supports **bash-style environment variable expansion** using `${...}` tokens. This allows you to keep sensitive credentials out of version control and adapt configurations across deployment environments without editing files.

Resolution happens **after JSON parsing** but **before** values are stored in `PayOSConfig.settings`, ensuring passwords and API keys are never written to disk.

## Basic Syntax

### Simple substitution: `${VAR}`

Replaces the token with the value of the environment variable `VAR`. If unset, logs a warning and substitutes an empty string.

```json
{
  "database-service": {
    "username": "${DB_USER}",
    "password": "${DB_PASSWORD}"
  }
}
```

**Environment**:
```bash
export DB_USER=payos_app
export DB_PASSWORD=secret123
```

**Result**:
```json
{
  "database-service": {
    "username": "payos_app",
    "password": "secret123"
  }
}
```

### Multiple tokens in one value

A single string can contain multiple tokens:

```json
{
  "servers": {
    "http": {
      "listen": "${HOST}:${PORT}"
    }
  }
}
```

**Environment**:
```bash
export HOST=0.0.0.0
export PORT=8080
```

**Result**: `listen` → `"0.0.0.0:8080"`

## Operators

PayOS supports bash-like parameter expansion operators for defaults, alternates, and required variables.

### `:−` (colon-dash) — Default if unset or empty

```json
{
  "host": "${DB_HOST:-localhost}"
}
```

- If `DB_HOST` is **set and non-empty**, use its value
- If `DB_HOST` is **unset or empty**, use `"localhost"`

### `:=` (colon-equals) — Alias for `:-`

Identical behavior to `:-`. Use whichever you prefer:

```json
{
  "port": "${PORT:=8080}"
}
```

### `:+` (colon-plus) — Alternate if set and non-empty

```json
{
  "profile": "${ENV:+production}"
}
```

- If `ENV` is **set and non-empty**, use `"production"`
- If `ENV` is **unset or empty**, use `""`

Useful for conditional flags:

```json
{
  "debug": "${DEBUG_MODE:+true}"
}
```

If `DEBUG_MODE` is set (even to an empty value like `export DEBUG_MODE=`), evaluates to `"true"`. Otherwise empty string.

### `:?` (colon-question) — Throw error if unset or empty

```json
{
  "apiKey": "${API_KEY:?API_KEY environment variable is required}"
}
```

- If `API_KEY` is **set and non-empty**, use its value
- If `API_KEY` is **unset or empty**, throw `IllegalStateException` with the message

Use this to **enforce required configuration** and fail fast at startup.

### `-` (dash, no colon) — Default only if unset (empty is preserved)

```json
{
  "value": "${VAR-default}"
}
```

- If `VAR` is **unset**, use `"default"`
- If `VAR` is **set** (even to empty string), use the value (including `""`)

Difference from `:-`:
- `:-` treats **empty** as unset
- `-` only checks **existence**

### `=` (equals, no colon) — Alias for `-`

```json
{
  "value": "${VAR=default}"
}
```

### `+` (plus, no colon) — Alternate if set (even if empty)

```json
{
  "flag": "${VAR+yes}"
}
```

- If `VAR` is **set** (even to `""`), use `"yes"`
- If `VAR` is **unset**, use `""`

### `?` (question, no colon) — Throw error only if unset

```json
{
  "apiKey": "${API_KEY?API_KEY must be set}"
}
```

- If `API_KEY` is **set** (even if empty), use the value
- If `API_KEY` is **unset**, throw `IllegalStateException`

## File References: `${file:/path}`

Read the content of a file (useful for Docker secrets):

```json
{
  "database-service": {
    "password": "${file:/run/secrets/db_password}"
  }
}
```

- Reads the file at `/run/secrets/db_password`
- Strips leading and trailing whitespace
- Throws `IllegalStateException` if file cannot be read

### Kubernetes / Docker Secrets Example

```yaml
# docker-compose.yml
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```json
{
  "database-service": {
    "url": "jdbc:postgresql://db:5432/payos",
    "username": "payos_user",
    "password": "${file:/run/secrets/db_password}"
  }
}
```

At runtime, PayOS reads the mounted secret file and injects the password.

## Default Values with Complex Strings

Operand values (defaults, alternates) are themselves resolved for embedded tokens:

```json
{
  "url": "${DATABASE_URL:-jdbc:postgresql://${DB_HOST:-localhost}:5432/payos}"
}
```

**Resolution**:
1. Check `DATABASE_URL` environment variable
2. If unset, evaluate the default: `jdbc:postgresql://${DB_HOST:-localhost}:5432/payos`
3. Within the default, resolve `${DB_HOST:-localhost}`
4. If `DB_HOST` is unset, use `"localhost"`

**Final result** (if both unset): `"jdbc:postgresql://localhost:5432/payos"`

## Precedence and Evaluation Order

For a token like `${VAR:-default}`:

1. **Environment variable**: Check `System.getenv("VAR")`
2. **Default evaluation**: If env var doesn't satisfy the condition, evaluate operand (which may itself contain tokens)
3. **Config references**: If the expression contains dots and is not found as an env var, attempt config path resolution (see [config-references.md](config-references.md))

## Common Patterns

### Database connection with fallbacks

```json
{
  "database-service": {
    "url": "${DATABASE_URL:-jdbc:postgresql://localhost:5432/payos}",
    "username": "${DB_USER:-payos}",
    "password": "${DB_PASSWORD:?Database password is required}",
    "pool": {
      "maxSize": "${DB_POOL_SIZE:-10}"
    }
  }
}
```

### Multi-environment base URL

```json
{
  "api": {
    "baseUrl": "${API_BASE_URL:-https://api.example.com}",
    "timeout": "${API_TIMEOUT:-30000}"
  }
}
```

### TLS certificate paths

```json
{
  "servers": {
    "https": {
      "enabled": true,
      "keystore": "${file:${KEYSTORE_PATH:-/etc/payos/keystore.jks}}",
      "keystorePassword": "${KEYSTORE_PASSWORD:?Keystore password required}"
    }
  }
}
```

### Feature flags

```json
{
  "features": {
    "experimental": "${ENABLE_EXPERIMENTAL:+true}",
    "debug": "${DEBUG_MODE:+true}"
  }
}
```

## Important Notes

### Environment Variable Naming

- Use uppercase with underscores: `DB_HOST`, `API_KEY`
- Avoid dots in env var names (dots are for config path references)
- Do not use spaces or special characters

### Security Best Practices

1. **Never commit secrets to version control**:
   ```json
   // ❌ BAD
   { "password": "hardcoded_secret" }
   
   // ✅ GOOD
   { "password": "${DB_PASSWORD}" }
   ```

2. **Use required operators (`:?`) for critical secrets**:
   ```json
   {
     "apiKey": "${API_KEY:?API key must be provided}"
   }
   ```

3. **Use file references for containerized deployments**:
   ```json
   {
     "password": "${file:/run/secrets/db_password}"
   }
   ```

### Resolution Timing

- Happens **once at startup** during `ConfigLoader.loadConfig()`
- Values are resolved **before** they reach application code
- Hot-reload via `ConfigWatcher` re-applies resolution

### Limitations

- Cannot reference environment variables from within JavaScript API scripts (resolution happens before scripts run)
- No shell command substitution (e.g., `$(date)` won't work)
- No arithmetic expansion

## Related

- [Configuration References](config-references.md) — reuse config keys with dot notation
- [Bootstrap Reference](bootstrap-reference.md) — full configuration schema
- [Bundle Encryption](../operations/bundle-encryption.md) — encrypt sensitive config files
- [Deployment Guide](../operations/deployment.md) — environment variable setup in production
