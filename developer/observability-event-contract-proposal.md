Created: 2026-07-03
Last updated: 2026-07-10

> **⚠️ SUPERSEDED (2026-07-10)** — This proposal's core design principle ("one stable envelope for all event types") has been superseded by [`event-category-payload-contracts-v2-2026-07-12.md`](event-category-payload-contracts-v2-2026-07-12.md), which instead defines a **separate abstraction (interface + swappable implementation) per event category** (audit, analytics, event-sourcing/state-reconstruction, observability/metrics, inter-service integration, connector retry/DLQ diagnostics). See that document for the current direction. This file is kept for historical context and because its field-level detail (source/actor/correlation/transport/resource blocks, the mapping table from existing PayOS emitters, the mandatory/conditional field governance table) remains a useful reference when fleshing out the individual per-category contracts — it should not be implemented as a single unified envelope.

# Observability Event Contract Proposal v1

## Goal

This proposal defines a single canonical event envelope for PayOS so that:

- audit events remain structured and machine-readable;
- technical logs remain separate and continue to use standard SLF4J logging;
- business events can carry dynamic payloads without freezing the schema too early;
- metrics and traces can be derived later from the same source of truth.

The contract is intentionally designed for PayOS runtime, not as a generic enterprise schema.

## Design Principles

1. Keep one stable envelope for all event types.
2. Put all observability correlation fields in the envelope, not in the dynamic payload.
3. Keep business payloads dynamic and versioned.
4. Treat audit as a structured event stream, not as free-text logs.
5. Keep technical logs outside the audit contract.
6. Avoid mixing sensitive data with observability data; mask at source.

## Event Shape

```json
{
  "schemaVersion": "1.0",
  "eventId": "uuid",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "API_EXEC",
  "category": "technical",
  "subsystem": "http",
  "severity": "INFO",
  "outcome": "SUCCESS",
  "source": {
    "service": "payos-runtime",
    "module": "api-resource-handler",
    "version": "1.0.0",
    "instanceId": "runtime-1",
    "environment": "prod",
    "nodeId": "node-a"
  },
  "actor": {
    "type": "user",
    "id": "john.doe",
    "tenantId": "acme",
    "sessionId": "sess-123"
  },
  "correlation": {
    "correlationId": "abc-123",
    "traceId": "trace-789",
    "spanId": "span-456",
    "requestId": "req-111"
  },
  "transport": {
    "protocol": "http",
    "direction": "inbound",
    "method": "POST",
    "path": "/api/payments",
    "destination": "payment-service",
    "topic": null,
    "queue": null
  },
  "resource": {
    "type": "api",
    "id": "/api/payments",
    "entityId": "pay_123"
  },
  "business": {
    "domain": "payments",
    "operation": "authorization",
    "entityType": "transaction",
    "entityId": "tx_123",
    "payload": {
      "amount": 1500,
      "currency": "MAD",
      "method": "card",
      "maskedPan": "4532 **** **** 1234",
      "merchantId": "m_456"
    }
  },
  "metrics": {
    "durationMs": 18,
    "count": 1,
    "statusCode": 200,
    "retryCount": 0,
    "nackCount": 0,
    "payloadSizeBytes": 842
  },
  "security": {
    "tenantScoped": true,
    "pii": false,
    "maskedFields": ["pan"],
    "sensitivity": "internal",
    "auditTrail": true
  },
  "result": {
    "code": "SUCCESS",
    "reason": null
  },
  "details": {
    "anyFutureField": "value"
  }
}
```

## Field Semantics

### Envelope fields

| Field | Required | Meaning |
|---|---:|---|
| `schemaVersion` | Yes | Contract version for forward/backward compatibility. |
| `eventId` | Yes | Unique identifier for deduplication and traceability. |
| `timestamp` | Yes | Event creation time in UTC. |
| `eventType` | Yes | Stable event name, for example `AUTH_SUCCESS`, `SECRET_READ`, `NOTIFICATION_LOOKUP`. |
| `category` | Yes | High-level family: `technical`, `security`, `business`, `capability`. |
| `subsystem` | Yes | Owning subsystem: `http`, `tcp`, `queue`, `oidc`, `secret`, `notification`, `capability`, etc. |
| `severity` | Yes | `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL`. |
| `outcome` | Yes | Normalized result: `SUCCESS`, `FAILURE`, `DENIED`, `NOT_FOUND`, `SKIPPED`, `ACTIVE`. |

### Correlation and source fields

| Field | Purpose |
|---|---|
| `source` | Runtime identity and deployment context. |
| `actor` | User, system, or service that caused the event. |
| `correlation` | Distributed tracing and incident correlation. |
| `transport` | Protocol, direction, and route metadata. |
| `resource` | Affected technical resource. |

### Business fields

The `business` block is the dynamic area for domain-specific information.

- `domain` is the business area, for example `payments`, `notifications`, `secrets`, `capabilities`.
- `operation` is the business action, for example `authorization`, `reversal`, `delivery`, `lookup`.
- `entityType` and `entityId` identify the business object.
- `payload` is a free-form object containing additional fields specific to that case.

This is where PayOS can keep flexibility without changing the envelope each time a new use case appears.

### Metrics and security fields

| Field | Purpose |
|---|---|
| `metrics.durationMs` | Latency of the operation. |
| `metrics.count` | Countable occurrence for aggregation. |
| `metrics.statusCode` | HTTP-like or execution status when applicable. |
| `metrics.retryCount` | Number of retries or attempts. |
| `metrics.nackCount` | Queue negative acknowledgements or similar failure signals. |
| `metrics.payloadSizeBytes` | Observability of message size. |
| `security.maskedFields` | Fields sanitized before emission. |
| `security.sensitivity` | Data sensitivity classification. |
| `security.auditTrail` | Indicates the event belongs to the immutable audit trail. |

## What This Covers For PayOS

This contract covers the categories already identified for PayOS observability:

- technical runtime events;
- security and compliance events;
- business events;
- queue and transport signals;
- observability metadata for traces and metrics derivation.

It is also compatible with the current PayOS direction where:

- `IAuditLogger` remains the structured audit entry point;
- normal technical logs remain SLF4J logs;
- Prometheus metrics are derived from counters and histograms, not from raw JSON logs;
- OpenTelemetry traces can reuse `correlation.traceId`, `correlation.spanId`, and `correlation.correlationId`.

## Example Event Types

### Technical

- `API_EXEC`
- `SYSTEM_STARTUP`
- `SYSTEM_SHUTDOWN`
- `DECRYPT_FAILURE`
- `QUEUE_PUBLISH`
- `QUEUE_CONSUME`
- `HTTP_REQUEST`

### Security

- `AUTH_SUCCESS`
- `AUTH_FAILURE`
- `AUTHZ_GRANTED`
- `AUTHZ_DENIED`
- `SESSION_CREATED`
- `SESSION_DESTROYED`
- `TENANT_SIMULATOR_ACTIVE`

### Business

- `PAYMENT_AUTHORIZED`
- `PAYMENT_REVERSED`
- `NOTIFICATION_SENT`
- `NOTIFICATION_LOOKUP`
- `SECRET_READ`
- `SECRET_WRITTEN`
- `CAPABILITY_ACTIVATED`

## Strict Contract Profile

This is the recommended v1 profile if the goal is to make the schema precise enough for audit, metrics extraction, and future exporters without making the event envelope rigid.

### Mandatory fields

These fields must always be present on every event:

- `schemaVersion`
- `eventId`
- `timestamp`
- `eventType`
- `category`
- `subsystem`
- `severity`
- `outcome`
- `source.service`
- `source.module`
- `source.environment`
- `actor.tenantId`
- `correlation.correlationId`
- `result.code`

### Conditionally mandatory fields

These fields are required depending on the event category:

- `actor.id` for user-caused events;
- `actor.sessionId` for session-related security events;
- `transport.protocol` and `transport.direction` for technical flow events;
- `transport.method` and `transport.path` for HTTP events;
- `transport.topic` for queue publish/consume events;
- `resource.type` and `resource.id` when a concrete runtime resource is involved;
- `business.domain`, `business.operation`, `business.entityType`, `business.entityId` for business events;
- `business.payload` for all business events and for any domain-specific extension;
- `metrics.durationMs` for all timed operations;
- `metrics.statusCode` for HTTP-oriented or API-oriented events;
- `metrics.nackCount` for queue-related failures or rejections;
- `security.maskedFields` for events that may contain sensitive data.

### Explicitly forbidden fields

These should not appear in the event payload:

- raw PAN, CVV, PIN, secrets, or credentials;
- free-form stack traces in the canonical payload;
- application-specific unstructured messages when a typed field exists;
- duplicate correlation identifiers outside the `correlation` block;
- transport payload bodies unless an explicit business event intentionally carries a redacted or tokenized representation.

### Versioning rule

- `schemaVersion` changes only when a backward-incompatible change is introduced.
- Additive fields go into `details` or new nested blocks without changing the major schema version.
- Existing consumers must be able to ignore unknown fields safely.

## Mapping From Existing PayOS Events

This section shows how the current PayOS event sources map into the proposed schema.

| Current source | Existing event / behavior | Proposed `eventType` | `category` | `subsystem` | Notes |
|---|---|---|---|---|---|
| `payos.security.oidc.nimbus.NimbusSecurityService` | auth success | `AUTH_SUCCESS` | `security` | `oidc` | Keep user, tenant, app, correlation. |
| `payos.security.oidc.nimbus.NimbusSecurityService` | auth failure | `AUTH_FAILURE` | `security` | `oidc` | Put reason in `details.reason` or `result.reason`. |
| `payos.security.oidc.nimbus.NimbusSecurityService` | authz granted | `AUTHZ_GRANTED` | `security` | `oidc` | Include granted roles in `details.grantedRoles`. |
| `payos.security.oidc.nimbus.NimbusSecurityService` | authz denied | `AUTHZ_DENIED` | `security` | `oidc` | Include required roles in `details.requiredRoles`. |
| `payos.security.oidc.nimbus.NimbusSecurityService` | logout | `SESSION_DESTROYED` or `LOGOUT` | `security` | `oidc` | Prefer `SESSION_DESTROYED` when session semantics matter. |
| `payos.security.oidc.pac4j.SecurityService` | auth / logout / authz | same as above | `security` | `oidc` | Same normalization as Nimbus path. |
| `payos.security.CryptoService` | decryption failure | `DECRYPT_FAILURE` | `technical` | `crypto` | Add algorithm/context in `details`. |
| `payos.resources.api.ApiResourceHandler` | script execution / API execution | `API_EXEC` | `technical` | `http` | Put route, status, duration, appId in envelope. |
| `payos.multitenancy.TenantPolicyService` | tenant simulator activated | `TENANT_SIMULATOR_ACTIVE` | `security` | `multitenancy` | Mark as privileged/admin event. |
| `payos.capabilities.EventLog` / `CapabilityManager` | INSTALL / ACTIVATE / DEACTIVATE / UNINSTALL | `CAPABILITY_INSTALLED`, `CAPABILITY_ACTIVATED`, `CAPABILITY_DEACTIVATED`, `CAPABILITY_UNINSTALLED` | `capability` | `capability` | Keep capability id in `resource.entityId` and `business.entityId`. |
| `payos-secret-api.AbstractSecretProvider` | secret read success | `SECRET_READ` | `security` | `secret` | Include operation result and masked secret metadata only. |
| `payos-secret-api.AbstractSecretProvider` | secret not found | `SECRET_NOT_FOUND` | `security` | `secret` | Use `outcome = NOT_FOUND`. |
| `payos-secret-api.AbstractSecretProvider` | secret denied | `SECRET_ACCESS_DENIED` | `security` | `secret` | Use `outcome = DENIED`. |
| `payos-secret-api.AbstractSecretProvider` | secret written | `SECRET_WRITTEN` | `security` | `secret` | Keep caller and tenant. |
| `payos-secret-api.AbstractSecretProvider` | secret deleted | `SECRET_DELETED` | `security` | `secret` | Keep caller and tenant. |
| `payos-secret-api.AbstractSecretProvider` | secret listed | `SECRET_LISTED` | `security` | `secret` | Useful for access auditing and counts. |
| `payos-service-notification.NotificationQueryService` | lookup found | `NOTIFICATION_LOOKUP` | `business` | `notification` | Put lookup result in `business.payload.result = FOUND`. |
| `payos-service-notification.NotificationQueryService` | lookup not found | `NOTIFICATION_LOOKUP` | `business` | `notification` | Put lookup result in `business.payload.result = NOT_FOUND`. |
| Future queue producers/consumers | publish / consume / nack | `QUEUE_PUBLISH`, `QUEUE_CONSUME`, `QUEUE_NACK` | `technical` | `queue` | Use `metrics.nackCount` and `transport.topic`. |

## Normalization Rules

1. Security and technical events should use the envelope directly.
2. Business events should keep the stable business metadata in `business.domain`, `business.operation`, `business.entityType`, and `business.entityId`.
3. Free-form business variation must live in `business.payload`.
4. Large or unstable computed fields should stay in `details`, not in the stable envelope.
5. Any field needed for dashboards or alerting should be promoted to a first-class field only when it is reused across multiple event types.

## Recommended Implementation Boundary

If this proposal is approved, the most practical implementation boundary is:

1. keep `AuditLogger` as the structured entry point for the kernel;
2. adapt existing audit emitters in secret and notification modules to the same canonical envelope when they are ready;
3. keep SLF4J technical logs unchanged outside the audit stream;
4. add metric exporters separately from the audit logger, using the canonical events as the semantic source.

## Field Governance Table

This table is the practical rule set for authors and implementers.

| Field / block | Status | Rule |
|---|---|---|
| `schemaVersion` | Mandatory | Bump only for backward-incompatible changes. |
| `eventId` | Mandatory | Must be globally unique. |
| `timestamp` | Mandatory | UTC, ISO-8601. |
| `eventType` | Mandatory | Stable, machine-readable event name. |
| `category` | Mandatory | One of `technical`, `security`, `business`, `capability`. |
| `subsystem` | Mandatory | Owning subsystem or protocol family. |
| `severity` | Mandatory | Logging severity classification. |
| `outcome` | Mandatory | Normalized result code. |
| `source` | Mandatory | Runtime and deployment identity. |
| `actor.tenantId` | Mandatory | Always present for tenant-scoped flows. |
| `correlation.correlationId` | Mandatory | Always propagated end to end. |
| `actor.id` | Conditional | Required when an actor can be identified. |
| `actor.sessionId` | Conditional | Required for session-bound security events. |
| `transport` | Conditional | Required for transport-driven events. |
| `resource` | Conditional | Required when a concrete runtime resource is involved. |
| `business` | Conditional | Required for business events. |
| `business.payload` | Conditional | Dynamic object for case-specific fields. |
| `metrics` | Conditional | Required when the event should feed metrics derivation. |
| `security` | Conditional | Required when sensitivity or masking matters. |
| `details` | Optional | Catch-all for additive, low-stability fields. |

### Payload rules by category

#### Technical events

- Use `transport`, `resource`, and `metrics` as first-class fields.
- Keep `business` empty unless the event also carries business meaning.
- Keep `details` small and descriptive.

#### Security events

- Use `actor`, `correlation`, `result`, and `security` as first-class fields.
- Put reasons, role lists, and policy identifiers in `details`.
- Never include sensitive raw data.

#### Business events

- Use `business.domain`, `business.operation`, `business.entityType`, and `business.entityId`.
- Put variable business attributes in `business.payload`.
- Promote a field to the stable envelope only if it will be reused across multiple event types or metrics.

#### Capability events

- Use `resource.entityId` for the capability identifier.
- Use `business.payload` for operation-specific capability metadata.
- Keep lifecycle operations normalized as `INSTALL`, `ACTIVATE`, `DEACTIVATE`, `UNINSTALL` or the equivalent capability event names.

## Concrete Examples

### 1. Authentication success

```json
{
  "schemaVersion": "1.0",
  "eventId": "c1a0d4b3-7f42-4f9e-8e5c-4a4f20d6a111",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "AUTH_SUCCESS",
  "category": "security",
  "subsystem": "oidc",
  "severity": "INFO",
  "outcome": "SUCCESS",
  "source": {"service": "payos-runtime", "module": "NimbusSecurityService", "version": "1.0.0", "instanceId": "runtime-1", "environment": "prod", "nodeId": "node-a"},
  "actor": {"type": "user", "id": "john.doe", "tenantId": "acme", "sessionId": "sess-123"},
  "correlation": {"correlationId": "abc-123", "traceId": "trace-789", "spanId": "span-456", "requestId": "req-111"},
  "resource": {"type": "auth", "id": "oidc-login"},
  "metrics": {"count": 1, "durationMs": 18},
  "result": {"code": "SUCCESS"},
  "details": {"appId": "payment-app"}
}
```

### 2. API execution

```json
{
  "schemaVersion": "1.0",
  "eventId": "2d6e8e6b-4a8c-4b5c-9c19-0cfebdf5a222",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "API_EXEC",
  "category": "technical",
  "subsystem": "http",
  "severity": "INFO",
  "outcome": "SUCCESS",
  "source": {"service": "payos-runtime", "module": "ApiResourceHandler", "version": "1.0.0", "instanceId": "runtime-1", "environment": "prod", "nodeId": "node-a"},
  "actor": {"type": "user", "id": "john.doe", "tenantId": "acme"},
  "correlation": {"correlationId": "abc-123", "traceId": "trace-789", "spanId": "span-456", "requestId": "req-111"},
  "transport": {"protocol": "http", "direction": "inbound", "method": "POST", "path": "/api/payments", "destination": "payment-service"},
  "resource": {"type": "api", "id": "/api/payments/authorize"},
  "metrics": {"count": 1, "durationMs": 24, "statusCode": 200, "payloadSizeBytes": 842},
  "result": {"code": "SUCCESS"},
  "details": {"appId": "payment-app"}
}
```

### 3. Business payment event

```json
{
  "schemaVersion": "1.0",
  "eventId": "8b7b6a0f-0a55-4f78-9d70-4f4b5c3a3333",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "PAYMENT_AUTHORIZED",
  "category": "business",
  "subsystem": "payments",
  "severity": "INFO",
  "outcome": "SUCCESS",
  "source": {"service": "payos-runtime", "module": "payment-flow", "version": "1.0.0", "instanceId": "runtime-1", "environment": "prod", "nodeId": "node-a"},
  "actor": {"type": "user", "id": "john.doe", "tenantId": "acme"},
  "correlation": {"correlationId": "abc-123", "traceId": "trace-789", "spanId": "span-456", "requestId": "req-111"},
  "resource": {"type": "transaction", "id": "tx_123"},
  "business": {
    "domain": "payments",
    "operation": "authorization",
    "entityType": "transaction",
    "entityId": "tx_123",
    "payload": {
      "amount": 1500,
      "currency": "MAD",
      "channel": "card",
      "merchantId": "m_456",
      "maskedPan": "4532 **** **** 1234"
    }
  },
  "metrics": {"count": 1, "durationMs": 31, "statusCode": 200},
  "result": {"code": "SUCCESS"},
  "security": {"tenantScoped": true, "pii": false, "maskedFields": ["pan"], "sensitivity": "internal", "auditTrail": true}
}
```

### 4. Secret access denied

```json
{
  "schemaVersion": "1.0",
  "eventId": "f2f1e1b2-c3d4-4e5f-8a9b-012345678444",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "SECRET_ACCESS_DENIED",
  "category": "security",
  "subsystem": "secret",
  "severity": "WARN",
  "outcome": "DENIED",
  "source": {"service": "payos-runtime", "module": "AbstractSecretProvider", "version": "1.0.0", "instanceId": "runtime-1", "environment": "prod", "nodeId": "node-a"},
  "actor": {"type": "user", "id": "john.doe", "tenantId": "acme"},
  "correlation": {"correlationId": "abc-123", "traceId": "trace-789", "spanId": "span-456", "requestId": "req-111"},
  "resource": {"type": "secret", "id": "payments/private-key"},
  "metrics": {"count": 1},
  "result": {"code": "DENIED", "reason": "tenant policy rejected access"},
  "details": {"callerId": "api-client"},
  "security": {"tenantScoped": true, "pii": false, "maskedFields": ["secretName"], "sensitivity": "restricted", "auditTrail": true}
}
```

### 5. Capability lifecycle event

```json
{
  "schemaVersion": "1.0",
  "eventId": "aa11bb22-cc33-44dd-88ee-ff0011225544",
  "timestamp": "2026-07-02T10:15:30.123Z",
  "eventType": "CAPABILITY_ACTIVATED",
  "category": "capability",
  "subsystem": "capability",
  "severity": "INFO",
  "outcome": "SUCCESS",
  "source": {"service": "payos-pm", "module": "CapabilityManager", "version": "1.0.0", "instanceId": "pm-1", "environment": "prod", "nodeId": "node-a"},
  "actor": {"type": "system", "id": "capability-manager", "tenantId": "acme"},
  "correlation": {"correlationId": "abc-123", "traceId": "trace-789", "spanId": "span-456", "requestId": "req-111"},
  "resource": {"type": "capability", "id": "notification-service", "entityId": "notification-service"},
  "business": {
    "domain": "capability-management",
    "operation": "activate",
    "entityType": "capability",
    "entityId": "notification-service",
    "payload": {
      "version": "1.2.0",
      "scope": "tenant",
      "installMode": "runtime"
    }
  },
  "metrics": {"count": 1},
  "result": {"code": "SUCCESS"}
}
```

## Recommended Next Step

If this proposal is accepted, the implementation should likely happen in two layers:

1. evolve `AuditEvent` into this canonical envelope while keeping compatibility helpers;
2. map existing specific events into it without losing their current semantics.

That keeps the change controlled and avoids breaking current audit producers.