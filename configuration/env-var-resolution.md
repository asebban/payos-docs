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

## Configuration Variable Resolution: `${config:path}`

In addition to environment variables, PayOS supports **configuration key references** using the `${config:...}` prefix. This allows you to reuse values defined elsewhere in the configuration without duplication.

### Basic Syntax

```json
{
  "shared": {
    "host": "api.example.com",
    "port": 8443
  },
  "service-a": {
    "endpoint": "${config:shared.host}"
  },
  "service-b": {
    "url": "https://${config:shared.host}:${config:shared.port}/api"
  }
}
```

**Result**:
```json
{
  "shared": {
    "host": "api.example.com",
    "port": 8443
  },
  "service-a": {
    "endpoint": "api.example.com"
  },
  "service-b": {
    "url": "https://api.example.com:8443/api"
  }
}
```

### Nested Object Navigation

Use dot notation to access nested values:

```json
{
  "database": {
    "primary": {
      "host": "db1.example.com",
      "port": 5432,
      "credentials": {
        "username": "payos_user",
        "password": "${DB_PASSWORD}"
      }
    }
  },
  "backup-service": {
    "target": "${config:database.primary.host}",
    "port": "${config:database.primary.port}",
    "user": "${config:database.primary.credentials.username}"
  }
}
```

**Result** (assuming `DB_PASSWORD=secret123`):
```json
{
  "backup-service": {
    "target": "db1.example.com",
    "port": 5432,
    "user": "payos_user"
  }
}
```

### Array Access by Index

Access array elements using bracket notation:

```json
{
  "servers": ["server1.example.com", "server2.example.com", "server3.example.com"],
  "primary": {
    "host": "${config:servers[0]}"
  },
  "failover": {
    "host": "${config:servers[1]}"
  }
}
```

**Result**:
```json
{
  "primary": {
    "host": "server1.example.com"
  },
  "failover": {
    "host": "server2.example.com"
  }
}
```

### Combining with Environment Variables

Config references can be mixed with environment variable resolution:

```json
{
  "common": {
    "apiVersion": "v2",
    "basePath": "/api"
  },
  "services": {
    "payment": {
      "url": "${API_HOST:-https://localhost}${config:common.basePath}/${config:common.apiVersion}/payments",
      "apiKey": "${PAYMENT_API_KEY:?Payment API key required}"
    }
  }
}
```

**Environment**:
```bash
export API_HOST=https://prod.example.com
export PAYMENT_API_KEY=pk_live_abc123
```

**Result**:
```json
{
  "services": {
    "payment": {
      "url": "https://prod.example.com/api/v2/payments",
      "apiKey": "pk_live_abc123"
    }
  }
}
```

### Default Values for Config References

Use the `:-` operator to provide defaults when a config path doesn't exist:

```json
{
  "app": {
    "theme": "dark"
  },
  "ui": {
    "colorScheme": "${config:app.theme:-light}",
    "language": "${config:app.locale:-en}"
  }
}
```

**Result**:
```json
{
  "ui": {
    "colorScheme": "dark",
    "language": "en"
  }
}
```

### Cross-Application Configuration Sharing

Configuration references are particularly useful in application inheritance scenarios:

```json
// parent-app/config/application.json
{
  "shared": {
    "database": {
      "host": "shared-db.example.com",
      "pool": {
        "min": 5,
        "max": 20
      }
    }
  }
}

// child-app/config/application.json
{
  "extends": ["parent-app"],
  "database-service": {
    "url": "jdbc:postgresql://${config:shared.database.host}:5432/child_db",
    "pool": {
      "maxSize": "${config:shared.database.pool.max}"
    }
  }
}
```

### Resolution Order and Precedence

When resolving a `${...}` token, PayOS follows this order:

1. **Check for explicit prefix**:
   - `${config:path}` → Always resolve as configuration reference
   - `${file:path}` → Always resolve as file read
   - `${env:VAR}` → Always resolve as environment variable

2. **If no prefix** (e.g., `${VAR}`):
   - First try as environment variable `System.getenv("VAR")`
   - If not found and contains dots (e.g., `${a.b.c}`), try as config path
   - If still not found, use default value or throw error per operator

### Circular Reference Detection

PayOS detects circular references and fails fast:

```json
{
  "a": "${config:b}",
  "b": "${config:c}",
  "c": "${config:a}"
}
```

**Result**: `IllegalStateException` thrown at startup with message indicating circular dependency.

### Best Practices

1. **Use explicit prefix for clarity**:
   ```json
   // ✅ GOOD - clear intent
   { "value": "${config:shared.setting}" }
   
   // ⚠️ AMBIGUOUS - could be env var or config
   { "value": "${shared.setting}" }
   ```

2. **Define shared configuration in a dedicated section**:
   ```json
   {
     "shared": {
       "apiVersion": "v2",
       "timeout": 30000
     },
     "service-a": {
       "version": "${config:shared.apiVersion}"
     }
   }
   ```

3. **Use config references for DRY configuration**:
   ```json
   // ❌ BAD - duplication
   {
     "service-a": { "host": "api.example.com" },
     "service-b": { "host": "api.example.com" },
     "service-c": { "host": "api.example.com" }
   }
   
   // ✅ GOOD - single source of truth
   {
     "shared": { "host": "api.example.com" },
     "service-a": { "host": "${config:shared.host}" },
     "service-b": { "host": "${config:shared.host}" },
     "service-c": { "host": "${config:shared.host}" }
   }
   ```

4. **Combine with environment variables for flexibility**:
   ```json
   {
     "defaults": {
       "port": 8080
     },
     "server": {
       "port": "${PORT:-${config:defaults.port}}"
     }
   }
   ```

### Limitations

- Config references are resolved **left-to-right, top-to-bottom** in the JSON structure
- Cannot reference values defined later in the file (forward references not supported)
- Cannot reference into arrays using dynamic indices (only literal numbers)
- Resolution depth is limited to prevent infinite loops

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
