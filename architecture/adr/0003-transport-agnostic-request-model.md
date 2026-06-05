# ADR-0003 — Transport-agnostic request/response model

**Status:** Accepted

## Context

PayOS applications must be reachable over multiple protocols — HTTP/HTTPS for web and API
clients, raw TCP for payment terminals and ISO 8583 traffic, and message queues for
asynchronous integration. Coupling business logic to any one protocol would force
re-implementation per transport and leak protocol concerns into application code.

## Decision

Introduce two transport-neutral exchange objects, `Request` and `Response`
(`ma.s2m.payos.servers.exchanges`), and a transport abstraction `IServer` / `Server` /
`ServerProvider`. Each transport adapter converts its wire format into a `Request`, calls
`Server.processRequest(appId, request)`, and serializes the returned `Response`. The
resource, scripting, and service layers never see the underlying protocol.

Routing is unified through `Application.getAppIdFromUri(uri)` (first path segment), and the
cross-cutting metadata `X-Tenant-Id` / `X-Correlation-Id` is carried in `Request.contextData`
regardless of transport.

## Consequences

- **Positive:** application logic is written once and served over HTTP, TCP, and queues
  simultaneously; new protocols are added as `ServerProvider` implementations.
- **Positive:** consistent tenant/correlation propagation across all transports.
- **Negative:** transport-specific features (e.g. HTTP streaming, TCP framing) must be
  expressed within the neutral model or via transport plugins (e.g. the TCP codec plugins).
- See [request-processing.md](../request-processing.md) and [extensibility.md](../extensibility.md).
