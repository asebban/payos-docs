# Operations guide

For **operators** running PayOS in production: deployment, bundle encryption, secrets management, observability, and hot reload. This section assumes familiarity with the [architecture](../architecture/README.md) and the [configuration reference](../configuration/README.md).

## Contents

| Topic | Document |
| --- | --- |
| Deploying the runtime and bundles | [deployment.md](deployment.md) |
| Running the notification service daemon | [notification-service.md](notification-service.md) |
| Running the Task Scheduling System (TSS) — standalone or embedded, applying `jobs.json` changes without a restart (reconciliation, hot reload, `sync`), manual trigger from the command line or the secured admin HTTP endpoint, event-driven triggers (`queue`/`dependency`) started per node and their cluster caveats | [task-scheduling-system-v21-2026-11-15.md](task-scheduling-system-v21-2026-11-15.md) |
| Starting TSS from A to Z (French) — every prerequisite, the TSS database scripts per vendor, the exact start command (Linux, Windows, extra JDBC driver), expected start-up log lines, and every command-line option, output and exit code | [tss-startup-and-cli-guide-v2-2026-11-13.md](tss-startup-and-cli-guide-v2-2026-11-13.md) |
| TSS standalone smoke-test walkthrough — one worked example against a real bundle, plus 3 findings a real run surfaced (one fixed) | [tss-standalone-smoke-test-walkthrough-v2-2026-10-08.md](tss-standalone-smoke-test-walkthrough-v2-2026-10-08.md) |
| Encrypting/packing bundles (`edc`) | [bundle-encryption.md](bundle-encryption.md) |
| Managing secrets (Vault, rotation, `spm`) | [secrets-management.md](secrets-management.md) |
| Logging, correlation, tenancy, audit | [observability.md](observability.md) |
| Pluggable, durable audit-trail capability | [audit-trail.md](audit-trail.md) |
| Exhaustive Micrometer/Prometheus metrics catalog | [metrics-catalog-v2-2026-09-11.md](metrics-catalog-v2-2026-09-11.md) |
| Zero-downtime configuration reload | [hot-reload.md](hot-reload.md) |
| Guide of all CLI tools | [cli-tools-guide.md](./cli-tools-guide.md) |
| `payos-license-generator` setup guide — build, configure, run, generate a signed license, scrape its Prometheus `/metrics`, and optionally enable persistence to any JDBC database (H2 by default), verified against a real build and real HTTP requests | [payos-license-generator-setup-guide-v7-2026-11-01.md](payos-license-generator-setup-guide-v7-2026-11-01.md) |

## Related references

- [Configuration reference](../configuration/README.md) — every key.
- [CLI tools](../cli-tools/README.md) — `apm`, `cpm`, `ppm`, `spm`, `edc`, `pdoc`.
- [Build & release](../build-and-release/README.md) — artifacts and versions.

## Operating model at a glance

- The runtime is a single fat JAR (`payos-runtime`) launched with `--bundle-path`.
- A **bundle** is rooted at `payos.json`; that entrypoint points to `configDir`, where the
  runtime configuration files (typically `bootstrap.json`) live alongside applications and
  plugin dirs.
- Services (DB, queue, secrets, webhooks) are **service-adapter JARs** chosen by configuration.
- Configuration changes are picked up live by the [config watcher](hot-reload.md).
- Every request is tenant- and correlation-scoped and audited
  ([observability](observability.md)).
