# Configuration References

**Purpose**: Reuse configuration values within the same configuration file using dot-notation paths.

## Overview

In addition to environment variables and file references, PayOS configuration supports **internal config references** — the ability to reference other keys already declared in the configuration using dot-separated paths.

This eliminates duplication and makes configuration more maintainable by defining values once and reusing them throughout the configuration.

## Syntax

### Explicit prefix

```json
{
  "database": {
    "host": "db.example.com",
    "port": 5432
  },
  "connection": {
    "url": "${config:database.host}:${config:database.port}"
  }
}
```

**Result**: `connection.url` → `"db.example.com:5432"`

### Implicit (auto-detection)

If a reference contains dots and is not found as an environment variable, it's automatically treated as a config path:

```json
{
  "database": {
    "host": "db.example.com"
  },
  "connection": {
    "url": "${database.host}:5432"
  }
}
```

**Result**: `connection.url` → `"db.example.com:5432"`

## Resolution Order

For a token like `${database.host}`:

1. **Environment variable** — checks `System.getenv("database.host")`
2. **Config path** (if contains `.`) — navigates `config.database.host`
3. **Empty string** — if not found anywhere, logs warning and substitutes `""`

For `${config:database.host}`:

1. **Config path** — directly navigates `config.database.host`
2. **Environment variable fallback** — attempts env var if not found in config
3. **Empty string** — if still not found

## Multi-Level Paths

Navigate nested structures using dots:

```json
{
  "server": {
    "http": {
      "host": "api.example.com",
      "port": 8080
    }
  },
  "client": {
    "endpoint": "${server.http.host}:${server.http.port}"
  }
}
```

**Result**: `client.endpoint` → `"api.example.com:8080"`

## Chained References

Config values can reference other values that themselves contain references:

```json
{
  "base": {
    "domain": "example.com"
  },
  "api": {
    "host": "api.${base.domain}"
  },
  "frontend": {
    "url": "https://${api.host}"
  }
}
```

**Result**: 
- `api.host` → `"api.example.com"`
- `frontend.url` → `"https://api.example.com"`

## Non-String Values

References to non-string values are automatically converted:

```json
{
  "database": {
    "port": 5432
  },
  "connection": {
    "url": "localhost:${database.port}"
  }
}
```

**Result**: `connection.url` → `"localhost:5432"` (port 5432 is converted to string)

## Combining with Defaults and Environment Variables

Mix config references with bash-style operators and environment variables:

```json
{
  "server": {
    "host": "localhost"
  },
  "app": {
    "url": "${server.host}:${PORT:-8080}/${API_PATH:-api}"
  }
}
```

**Result** (assuming `PORT` and `API_PATH` are unset):
- `app.url` → `"localhost:8080/api"`

## Use Cases

### Shared Database Configuration

Define database connection details once, reuse across multiple services:

```json
{
  "database": {
    "host": "${DB_HOST:-localhost}",
    "port": 5432,
    "name": "payos"
  },
  "applications": [
    {
      "id": "payments",
      "database-service": {
        "url": "jdbc:postgresql://${database.host}:${database.port}/${database.name}",
        "username": "${DB_USER}",
        "password": "${DB_PASSWORD}"
      }
    },
    {
      "id": "reports",
      "database-service": {
        "url": "jdbc:postgresql://${database.host}:${database.port}/${database.name}",
        "username": "${DB_USER}",
        "password": "${DB_PASSWORD}"
      }
    }
  ]
}
```

### Service Discovery

Use a single registry of service endpoints:

```json
{
  "services": {
    "auth": {
      "host": "auth.internal",
      "port": 9000
    },
    "payment": {
      "host": "payment.internal",
      "port": 9001
    }
  },
  "applications": [
    {
      "id": "gateway",
      "connectors": [
        {
          "type": "http-client",
          "auth-endpoint": "https://${services.auth.host}:${services.auth.port}/token",
          "payment-endpoint": "https://${services.payment.host}:${services.payment.port}/process"
        }
      ]
    }
  ]
}
```

### Environment-Specific Base URLs

Define base URLs once, compose specific endpoints:

```json
{
  "environment": {
    "apiBase": "${API_BASE_URL:-https://api.example.com}",
    "cdnBase": "${CDN_BASE_URL:-https://cdn.example.com}"
  },
  "applications": [
    {
      "id": "frontend",
      "endpoints": {
        "auth": "${environment.apiBase}/auth",
        "data": "${environment.apiBase}/data",
        "assets": "${environment.cdnBase}/assets"
      }
    }
  ]
}
```

## Important Notes

### Resolution Context

- References are resolved within the **root configuration map**
- All keys at any nesting level are accessible via dot notation
- Resolution happens **after** JSON parsing but **before** the config is stored in `PayOSConfig.settings`

### Circular References

Circular references will cause infinite recursion. Design your configuration to avoid cycles:

```json
// ❌ DON'T DO THIS
{
  "a": "${b}",
  "b": "${a}"
}
```

### Order Independence

References can point to keys declared anywhere in the configuration (before or after the referencing key). Resolution order is handled automatically.

```json
// ✅ This works
{
  "app": {
    "url": "${database.host}"
  },
  "database": {
    "host": "db.example.com"
  }
}
```

### Nested Maps Are Not Stringified

Only **leaf values** are referenced. You cannot reference an entire map:

```json
{
  "database": {
    "host": "localhost",
    "port": 5432
  },
  "copy": "${database}"  // ❌ Won't work as expected
}
```

To copy nested structures, reference individual keys or use JSON inheritance/composition at the application level.

## Related

- [Environment Variables](env-var-resolution.md) — syntax for environment variable resolution
- [File References](env-var-resolution.md#file-references) — reading secrets from files
- [Bootstrap Reference](bootstrap-reference.md) — full configuration schema
