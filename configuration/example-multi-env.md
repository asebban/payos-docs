# Example: Multi-Environment Configuration

This example demonstrates a complete PayOS configuration using both **environment variables** and **config references** to create a maintainable, environment-agnostic setup.

## Scenario

- Single configuration file works across DEV, STAGING, and PRODUCTION
- Database credentials injected via environment variables
- Common values (ports, timeouts) reused with config references
- Secrets read from mounted files in production (Docker/K8s)

## Configuration File: `config/bootstrap.json`

```json
{
  "configDir": "config",
  
  "environment": {
    "name": "${ENV_NAME:-development}",
    "region": "${REGION:-eu-west-1}"
  },
  
  "network": {
    "internalHost": "${INTERNAL_HOST:-localhost}",
    "publicHost": "${PUBLIC_HOST:-localhost}",
    "httpPort": 8080,
    "httpsPort": 8443,
    "tcpPort": 9090
  },
  
  "database": {
    "host": "${DB_HOST:-localhost}",
    "port": 5432,
    "name": "${DB_NAME:-payos}",
    "username": "${DB_USER:-payos}",
    "password": "${file:${DB_PASSWORD_FILE:-/dev/null}}",
    "pool": {
      "minSize": 5,
      "maxSize": "${DB_POOL_MAX:-20}"
    }
  },
  
  "servers": {
    "http": {
      "enabled": true,
      "host": "${network.internalHost}",
      "port": "${network.httpPort}",
      "publicUrl": "https://${network.publicHost}"
    },
    "https": {
      "enabled": "${ENABLE_HTTPS:+true}",
      "host": "${network.internalHost}",
      "port": "${network.httpsPort}",
      "keystore": "${file:${KEYSTORE_PATH:-/etc/payos/keystore.jks}}",
      "keystorePassword": "${KEYSTORE_PASSWORD:?HTTPS enabled but keystore password not set}",
      "keystoreType": "JKS"
    },
    "tcp": {
      "enabled": true,
      "host": "${network.internalHost}",
      "port": "${network.tcpPort}"
    }
  },
  
  "database-service": {
    "url": "jdbc:postgresql://${database.host}:${database.port}/${database.name}",
    "username": "${database.username}",
    "password": "${database.password}",
    "driverClassName": "org.postgresql.Driver",
    "pool": {
      "minPoolSize": "${database.pool.minSize}",
      "maxPoolSize": "${database.pool.maxSize}",
      "acquireIncrement": 2,
      "maxIdleTime": 300
    }
  },
  
  "queue-service": {
    "enabled": "${ENABLE_QUEUE:+true}",
    "url": "${NATS_URL:-nats://localhost:4222}",
    "subject": "${NATS_SUBJECT:-payos.events}",
    "credentials": "${file:${NATS_CREDS_FILE:-/run/secrets/nats_creds}}"
  },
  
  "security": {
    "oidc": {
      "enabled": "${ENABLE_OIDC:+true}",
      "discoveryUri": "${OIDC_DISCOVERY_URL:?OIDC enabled but discovery URL not set}",
      "clientId": "${OIDC_CLIENT_ID}",
      "clientSecret": "${file:${OIDC_SECRET_FILE:-/run/secrets/oidc_secret}}",
      "callbackUrl": "${servers.http.publicUrl}/callback"
    }
  },
  
  "multitenancy": {
    "enabled": true,
    "policy": "header",
    "headerName": "X-Tenant-Id",
    "defaultTenantId": "default"
  },
  
  "applications": [
    {
      "id": "payments",
      "base.path": "../apps/payments",
      "active": true,
      "description": "Payment processing API"
    },
    {
      "id": "reports",
      "base.path": "../apps/reports",
      "active": "${ENABLE_REPORTS:+true}",
      "description": "Reporting and analytics"
    }
  ],
  
  "logging": {
    "level": "${LOG_LEVEL:-INFO}",
    "format": "${LOG_FORMAT:-json}"
  },
  
  "observability": {
    "metricsUrl": "${METRICS_URL:-http://prometheus:9090}",
    "tracingUrl": "${TRACING_URL:-http://jaeger:14268}",
    "environment": "${environment.name}",
    "region": "${environment.region}"
  }
}
```

## Environment Configurations

### Development (`.env.dev`)

```bash
# Development environment
ENV_NAME=development
REGION=local

# Local database
DB_HOST=localhost
DB_NAME=payos_dev
DB_USER=dev_user
DB_PASSWORD_FILE=/tmp/dev_db_password.txt

# Disable optional features
# ENABLE_HTTPS not set → HTTPS disabled
# ENABLE_QUEUE not set → Queue disabled
# ENABLE_OIDC not set → OIDC disabled

# Logging
LOG_LEVEL=DEBUG
LOG_FORMAT=text
```

### Staging (`.env.staging`)

```bash
# Staging environment
ENV_NAME=staging
REGION=eu-west-1

# Network
INTERNAL_HOST=0.0.0.0
PUBLIC_HOST=staging-api.example.com

# Database
DB_HOST=staging-db.example.internal
DB_NAME=payos
DB_USER=payos_staging
DB_PASSWORD_FILE=/run/secrets/db_password
DB_POOL_MAX=50

# Enable HTTPS
ENABLE_HTTPS=1
KEYSTORE_PATH=/etc/payos/staging-keystore.jks
KEYSTORE_PASSWORD_FILE=/run/secrets/keystore_password

# Enable Queue
ENABLE_QUEUE=1
NATS_URL=nats://nats.staging.internal:4222
NATS_SUBJECT=payos.staging.events
NATS_CREDS_FILE=/run/secrets/nats_creds

# Enable OIDC
ENABLE_OIDC=1
OIDC_DISCOVERY_URL=https://auth.staging.example.com/.well-known/openid-configuration
OIDC_CLIENT_ID=payos-staging
OIDC_SECRET_FILE=/run/secrets/oidc_secret

# Enable reports
ENABLE_REPORTS=1

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

### Production (Kubernetes ConfigMap + Secrets)

**ConfigMap**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payos-config
data:
  ENV_NAME: "production"
  REGION: "eu-west-1"
  INTERNAL_HOST: "0.0.0.0"
  PUBLIC_HOST: "api.example.com"
  DB_HOST: "prod-db.example.internal"
  DB_NAME: "payos"
  DB_USER: "payos_prod"
  DB_POOL_MAX: "100"
  ENABLE_HTTPS: "1"
  ENABLE_QUEUE: "1"
  NATS_URL: "nats://nats.prod.internal:4222"
  NATS_SUBJECT: "payos.prod.events"
  ENABLE_OIDC: "1"
  OIDC_DISCOVERY_URL: "https://auth.example.com/.well-known/openid-configuration"
  OIDC_CLIENT_ID: "payos-production"
  ENABLE_REPORTS: "1"
  LOG_LEVEL: "WARN"
  LOG_FORMAT: "json"
  METRICS_URL: "http://prometheus.monitoring:9090"
  TRACING_URL: "http://jaeger.monitoring:14268"
```

**Secrets** (mounted as files):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: payos-secrets
type: Opaque
data:
  db_password: <base64-encoded>
  keystore_password: <base64-encoded>
  nats_creds: <base64-encoded>
  oidc_secret: <base64-encoded>
```

**Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payos
spec:
  template:
    spec:
      containers:
      - name: payos
        image: payos:latest
        envFrom:
        - configMapRef:
            name: payos-config
        env:
        - name: DB_PASSWORD_FILE
          value: /run/secrets/db_password
        - name: KEYSTORE_PATH
          value: /run/secrets/keystore.jks
        - name: KEYSTORE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: payos-secrets
              key: keystore_password
        - name: NATS_CREDS_FILE
          value: /run/secrets/nats_creds
        - name: OIDC_SECRET_FILE
          value: /run/secrets/oidc_secret
        volumeMounts:
        - name: secrets
          mountPath: /run/secrets
          readOnly: true
      volumes:
      - name: secrets
        secret:
          secretName: payos-secrets
```

## Resolution Flow

### Example 1: Database URL

**Config**:
```json
{
  "database": {
    "host": "${DB_HOST:-localhost}",
    "port": 5432,
    "name": "${DB_NAME:-payos}"
  },
  "database-service": {
    "url": "jdbc:postgresql://${database.host}:${database.port}/${database.name}"
  }
}
```

**Development** (no env vars):
1. `DB_HOST` → unset → default `"localhost"`
2. `DB_NAME` → unset → default `"payos"`
3. `database.host` → `"localhost"`
4. `database.port` → `5432`
5. `database.name` → `"payos"`
6. `database-service.url` → `"jdbc:postgresql://localhost:5432/payos"`

**Production** (`DB_HOST=prod-db.internal`, `DB_NAME=payos_prod`):
1. `DB_HOST` → `"prod-db.internal"`
2. `DB_NAME` → `"payos_prod"`
3. `database.host` → `"prod-db.internal"`
4. `database-service.url` → `"jdbc:postgresql://prod-db.internal:5432/payos_prod"`

### Example 2: OIDC Callback URL

**Config**:
```json
{
  "network": {
    "publicHost": "${PUBLIC_HOST:-localhost}"
  },
  "servers": {
    "http": {
      "publicUrl": "https://${network.publicHost}"
    }
  },
  "security": {
    "oidc": {
      "callbackUrl": "${servers.http.publicUrl}/callback"
    }
  }
}
```

**Development** (no env vars):
- `PUBLIC_HOST` → `"localhost"`
- `network.publicHost` → `"localhost"`
- `servers.http.publicUrl` → `"https://localhost"`
- `security.oidc.callbackUrl` → `"https://localhost/callback"`

**Production** (`PUBLIC_HOST=api.example.com`):
- `PUBLIC_HOST` → `"api.example.com"`
- `network.publicHost` → `"api.example.com"`
- `servers.http.publicUrl` → `"https://api.example.com"`
- `security.oidc.callbackUrl` → `"https://api.example.com/callback"`

### Example 3: Optional Features

**Config**:
```json
{
  "queue-service": {
    "enabled": "${ENABLE_QUEUE:+true}"
  },
  "applications": [
    {
      "id": "reports",
      "active": "${ENABLE_REPORTS:+true}"
    }
  ]
}
```

**Development** (features disabled):
- `ENABLE_QUEUE` → unset → `queue-service.enabled` → `""`
- `ENABLE_REPORTS` → unset → `applications[1].active` → `""`

**Production** (`ENABLE_QUEUE=1`, `ENABLE_REPORTS=1`):
- `ENABLE_QUEUE` → set → `queue-service.enabled` → `"true"`
- `ENABLE_REPORTS` → set → `applications[1].active` → `"true"`

## Benefits of This Approach

1. **Single Configuration File**: Same `bootstrap.json` for all environments
2. **Secrets Never Committed**: All sensitive values injected at runtime
3. **No Duplication**: Common values defined once, referenced everywhere
4. **Environment Flexibility**: Override only what changes per environment
5. **Fail-Fast Validation**: Required secrets (`:?`) cause startup failure if missing
6. **Cloud-Native**: Works seamlessly with Kubernetes ConfigMaps and Secrets

## Related

- [Environment Variable Resolution](env-var-resolution.md)
- [Configuration References](config-references.md)
- [Bootstrap Reference](bootstrap-reference.md)
- [Deployment Guide](../operations/deployment.md)
