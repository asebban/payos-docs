# ADR: Task Scheduling System for Multi-Product Financial Platform

## Status
**<span style="color: orange;">Proposed</span>**

---

## Title
Adoption of a Centralized, Distributed Task Scheduling System for Operational and Business Workloads

---

## Context

The platform operates in a complex enterprise environment:

- Multi-product ecosystem (Gateway, Issuing, Acquiring, Switching, Analytics)
- Multi-tenant by design
- Serving Tier 1–3 financial institutions and SMB segments
- Deployable On-premise, PaaS, and Public Cloud
- Operating in regulated financial environments
- Experiencing rapid engineering team growth

The platform requires scheduled and asynchronous task execution for:

- Periodic reconciliation jobs
- Settlement processing
- Batch reporting
- Fraud model recalibration (if AI is used for fraud detection)
- Data retention and archival
- Compliance exports (periodic data exports / reports required by regulators / auditors)
- Retry and compensation workflows
- ...

A standardized task scheduling architecture is required.

---

## Scope

### Includes
- Scheduled tasks (cron-like)
- Periodic batch jobs
- Message-driven job triggering (a job execution triggered by a single message received on a queue destination, as an alternative or complement to a timer trigger — see the TSS base specification in payos-job-scheduler for the detailed mechanism)
- Distributed execution coordination
- Multi-tenant task isolation
- Retry policies
- Monitoring and observability
- Failover and high availability

### Excludes
- Real-time/continuous stream processing — windowed aggregation, stream-to-stream transformation, and stateful stream analytics remain the responsibility of a dedicated streaming engine or the message-oriented middleware itself. TSS treating a single message as a job trigger (one message in, one discrete, idempotent job execution) is job invocation, not stream processing, and stays in scope — see the TSS base specification for details.
- Workflow orchestration for complex BPM (covered separately)
- Infrastructure-level job schedulers (e.g., OS cron as primary mechanism)

> **Amendment (2026-09-25):** the Excludes entry on real-time stream processing was narrowed from a blanket exclusion to explicitly allow message-triggered job invocation, once it became clear that a job firing once per queue message is a different concern from continuous stream processing. Full rationale and the triggering mechanism are documented in the TSS base specification (payos-job-scheduler).

---

## Decision Drivers

1. **Regulatory Compliance**
   - Audit logs of job execution
   - Traceability of task runs
   - Deterministic retry policies

2. **Multi-Tenant Isolation**
   - Tenant-specific tasks
   - No cross-tenant interference
   - Tenant-scoped execution context

3. **High Availability**
   - No single-node scheduler
   - Failover support
   - Idempotent task execution

4. **Operational Observability**
   - Central monitoring
   - Execution history
   - SLA tracking
   - Alerting on failure

5. **Cloud / On-Prem Portability**
   - Must work in environments without Internet access
   - Must not depend on SaaS-only schedulers

6. **Scalability**
   - Support high task volume
   - Horizontal scaling

7. **Consistency Across Products**
   - Avoid per product scheduling silos
   - Unified execution model

8. **Engineering Governance**
   - Standard APIs
   - Controlled retry policies
   - Avoid duplicate cron logic

---

## Considered Options

### Option 1: Infrastructure-Level Scheduling (OS Cron / Kubernetes CronJobs)

Each product defines scheduled tasks at infrastructure level.

**Pros**
- Simple
- No custom scheduler needed
- Cloud-native option (K8s CronJob)

**Cons**
- No centralized governance
- Weak audit capabilities
- No built-in tenant awareness
- Hard to coordinate across cluster
- Not ideal for on-prem legacy environments

---

### Option 2: In-Memory Scheduler Per Service (e.g., Spring @Scheduled)

Each microservice runs its own scheduler.

**Pros**
- Easy to implement
- Low overhead
- No external dependency

**Cons**
- Not cluster-safe
- Risk of duplicate execution
- No failover coordination
- No global visibility
- Weak audit trail
- Siloed per product

---

### Option 3: Distributed Persistent Scheduler (Quartz Clustered Mode)

Centralized scheduler backed by database.

**Pros**
- Persistent jobs
- Cluster-aware
- Failover support
- Cron support
- Audit via DB
- Works on-prem

**Cons**
- Requires shared database
- Operational complexity
- Database contention at scale

---

### Option 4: Event-Driven Scheduling via Message Broker

Use a message broker (e.g., delayed queues) for scheduling and retries.

**Pros**
- Scalable
- Decoupled
- Natural fit for distributed systems
- Good retry semantics

**Cons**
- Cron semantics harder
- Requires orchestration logic
- More complex to manage periodic jobs

---

### Option 5: Dedicated Workflow / Job Orchestration Engine (e.g., Temporal / Camunda)

Use workflow engine to manage tasks.

**Pros**
- Rich orchestration
- State persistence
- Retry policies
- Visibility

**Cons**
- Heavyweight
- Overkill for simple scheduled tasks
- Operational overhead

---

## Evaluation Summary

| Criteria | Infra Cron | In-Memory | Quartz Cluster | Event-Driven | Workflow Engine |
|------------|------------|------------|----------------|--------------|----------------|
| Multi-Tenant Isolation | No | No | Yes | Yes | Yes |
| HA / Failover | Limited | No | Yes | Yes | Yes |
| Auditability | No | No | Yes | Should be implemented | Yes |
| Cron Support | Yes | Yes | Yes | Should be implemented | Yes |
| Retry Policies | No | Limited | Yes | Yes | Yes |
| On-Prem Compatibility | Yes | Yes | Yes | Yes | Yes |
| Operational Complexity | Low | Low | Medium | Medium | High |
| Governance | No | No | Yes | Should be implemented | Yes |

---

## Decision / Choice

### Adopt a Centralized, Distributed Persistent Scheduler (Clustered Mode)

We recommend the platform to implement a centralized task scheduling service based on a distributed, persistent scheduler. We recommend a Java based centralized clustered scheduling system like Quartz, with:

- Database-backed job persistence (very important for a financial system like S2M's)
- Cluster-aware execution (Means that Jobs can run on multiple nodes but only one node executes a given job. All jobs are connected to the same job store, no SPOF)
- Tenant-scoped job metadata
- Centralized monitoring (All products or modules use the same scheduling system for unified governance, unified audit trail, ...)
- Standardized retry policies
- Execution history tracking (All execution history stored in the database)
- Independance of the OS (does not rely on Linux Cron or Windows Task Scheduler)

Quartz is a self-contained, open-source library (primarily for Java and .NET) that manages its own scheduling logic internally. While it uses "cron-like" syntax to define schedules, it operates independently of the operating system's built-in scheduling tools.

### Architectural Principles

1. Scheduler runs as dedicated service or cluster component.
2. Jobs are persistent.
3. Execution is coordinated across nodes.
4. All jobs include tenant context.
5. Jobs are observable and auditable.
6. Standard API for job registration.

#### Idempotence

Jobs are not magically idempotent (running the job twice produces the same result as running it once). The job developer should design it properly to be so. You can find below some basic patterns of idempotency in financial systems:

##### Pattern 1: Status-Based Processing

Only process records based on a specific state (PENDING for example)

##### Pattern 2: Business Idempotency Key

Each job execution has a unique ID (specific to the current execution). Before applying changes, check if this execution ID was already applied. If yes → skip

##### Pattern 3: Transactional Boundary

Wrap processing in a database transaction. Either all steps succeed, or nothing is committed. This is ideal but not always feasible for large batch jobs.

##### Pattern 4: Checkpointing

Store job progress (for example Last processed transaction ID). On restart, resume from checkpoint.


In Quartz, if a node crashes:

- Quartz sees missing heartbeat.
- Trigger becomes available.
- Another node acquires it.
- Job executes again.

Quartz assumes job logic is safe. It does not rollback your business data and does not know what part was completed and what part was not. Quartz garantees at least once, but does not garantee business data consistency.

#### Multi-tenancy

Quartz does NOT natively provide multi-tenancy. multi-tenancy must be designed on top of Quartz (it's a design concern).

Quartz Scheduler provides Job definitions, triggers, clustering, persistence, thread pools, at-least-once execution but does not provide idempotence, tenant isolation, tenant-aware authorization, tenant resource limits, data segregation and tenant-based job filtering.

Multi-tenancy could be provided with the following strategies:

##### Strategy 1: Logical multi-tenancy (Single scheduler + tenant metadata)

This is the most common strategy: 

- Use one Quartz cluster.
- Add tenantId in JobDataMap (A key-value structure (like a Map) attached to a Job or a trigger to pass parameters to it).
- Use JobGroup per tenant (A logical mechanism for grouping jobs, the froup could be the tenant ID).
- Prefix job keys with tenant (explicitly include the tenant id in the job name).

##### Strategy 2: Separate Scheduler per Tenant

Each tenant gets:

- Dedicated Quartz instance
- Dedicated DB schema (or DB)
- Dedicated thread pool

This strategy ensures physical strong resource and fault isolation but is more costly in terms of infrastructure. This strategy is usually used for very large Tier 1 tenants.

##### Strategy 3: Hybrid Model (Recommended for our case)

- Shared scheduler cluster
- Shared DB
- Tenant metadata enforced at registration layer (to make Jobs not rely on the developer for tenant isolation)
- Per-tenant quotas (i.e. max number of scheduled jobs, max execution frequency, ...)
- Per-tenant job groups
- Possibly separate thread pools for heavy tenants
---

## Consequences

### Positive

- Unified scheduling across products
- Regulatory-grade audit trail
- HA and failover support
- Multi-tenant safe
- Consistent retry semantics
- On-prem and cloud compatible
- Portability and independence of the OS

### Negative

- Additional infrastructure component
- Requires database availability
- Requires governance of job definitions
- Operational overhead compared to simple cron

---

## Risks and Mitigations

### Risk 1: Duplicate Job Execution

Cluster misconfiguration may trigger multiple runs.

**Mitigation:**
- Cluster mode with DB locking
- Idempotent job design
- Unique job keys per tenant

---

### Risk 2: Database Bottleneck

Scheduler relies on shared DB.

**Mitigation:**
- Dedicated scheduler schema
- Connection pooling
- Monitoring DB performance
- Horizontal scaling of worker nodes

---

### Risk 3: Long-Running Job Blocking

Jobs may block scheduler threads.

**Mitigation:**
- Separate execution thread pool
- Delegate heavy jobs to async workers
- Timeout enforcement

---

### Risk 4: Tenant Misconfiguration

Tenant-specific jobs may overload system.

**Mitigation:**
- Tenant quotas
- Job rate limiting
- Governance approval for high-frequency jobs

---

## Summary

Given:

- Multi-product architecture
- Multi-tenant requirements
- Regulatory constraints
- Hybrid deployment model
- Need for high availability and auditability

A centralized, clustered, persistent scheduling system provides the best balance between governance, scalability, compliance, and operational control.

This approach eliminates scheduling silos and ensures consistent execution across the platform.