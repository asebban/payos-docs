# Operations guide

For **operators** running PayOS in production: deployment, bundle encryption, secrets management, observability, and hot reload. This section assumes familiarity with the [architecture](../architecture/README.md) and the [configuration reference](../configuration/README.md).

## Contents

| Topic | Document |
| --- | --- |
| Deploying the runtime and bundles | [deployment.md](deployment.md) |
| Running the notification service daemon | [notification-service.md](notification-service.md) |
| Encrypting/packing bundles (`edc`) | [bundle-encryption.md](bundle-encryption.md) |
| Managing secrets (Vault, rotation, `spm`) | [secrets-management.md](secrets-management.md) |
| Logging, correlation, tenancy, audit | [observability.md](observability.md) |
| Zero-downtime configuration reload | [hot-reload.md](hot-reload.md) |
| Guide of all CLI tools | [cli-tools-guide.md](./cli-tools-guide.md) |

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
