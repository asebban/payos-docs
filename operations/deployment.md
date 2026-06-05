# Deployment

PayOS deploys as a single executable JAR (`payos-runtime`) run against a **bundle**. This
page describes how to lay out, configure, and launch a deployment, and how to place connector
and extension plugins. Configuration keys are in the
[configuration reference](../configuration/README.md).

## Artifacts

| Artifact | Role |
| --- | --- |
| `payos-runtime-<version>.jar` | The runnable server. Main class `ma.s2m.payos.BootServer`. Embeds the kernel, HTTP/TCP/queue transports, and the standard connectors (database, NATS queue, HTTP webhooks, filesystem secrets). |
| Connector JARs | Alternative/additional service backends placed in `connectors-dir`. |
| Extension JARs | Script-callable Java libraries placed in `extensions-dir`. |
| JDBC driver | Required by the database connector; placed in `connectors-dir`. |

See [build-and-release/module-map.md](../build-and-release/module-map.md) for what each
module contributes and the embedded versions.

## Bundle layout

```
/opt/payos/
├── payos-runtime-<version>.jar
└── bundle/
    ├── payos.json              # bundle entrypoint (usually only configDir)
    ├── config/                 # merged runtime config files, typically bootstrap.json
    ├── apps/                   # applications
    ├── connectors/             # connector JARs (+ JDBC driver)
    ├── extensions/             # extension JARs
    └── tcp-handlers/           # TCP codec/handler plugins (if using TCP)
```

## Launch

```bash
java -jar /opt/payos/payos-runtime-<version>.jar \
     --bundle-path /opt/payos/bundle
```

`--bundle-path` accepts a bundle directory or a direct path to `payos.json`. `BootServer`
loads `payos.json`, resolves `configDir`, merges the runtime configuration files from that
directory, initializes services, and starts the configured
[servers](../configuration/servers.md).

### As a service (systemd example)

```ini
[Unit]
Description=PayOS runtime
After=network.target

[Service]
ExecStart=/usr/bin/java -jar /opt/payos/payos-runtime-<version>.jar --bundle-path /opt/payos/bundle
Restart=on-failure
Environment=PAYOS_CONNECTORS_DIR=/opt/payos/bundle/connectors
Environment=PAYOS_EXTENSIONS_DIR=/opt/payos/bundle/extensions
# secrets master key (filesystem provider) — prefer Vault in production
# Environment=PAYOS_SECRET_MASTER_KEY=...

[Install]
WantedBy=multi-user.target
```

## Choosing service backends

Backends are selected by configuration `type`, and their JARs must be on the
[connectors path](../configuration/extensions-connectors.md):

| Service | Config block | Example type |
| --- | --- | --- |
| Database | `database-service` | (Hibernate) + JDBC driver |
| Queue | `queue-service` | `nats` |
| Secrets | `secret-service` | `filesystem` / `vault` |
| Webhooks | `webhooks` | `http` |

## TLS

For production HTTP, use the `https` protocol with a keystore — see
[configuration/servers.md](../configuration/servers.md). HSTS and `Secure` cookies are applied
automatically on HTTPS.

## Multi-tenancy in production

Ensure the [tenant simulator is disabled](../configuration/multi-tenancy.md) and
`requireTenantId` is `true`. Configure per-tenant schema/isolation/quotas as needed.

## Health and lifecycle endpoints

- `/health` — liveness/readiness probe target.
- `/stop` — graceful stop (restricted to localhost).

See [reference/http-endpoints.md](../reference/http-endpoints.md).

## Packing/encrypting bundles

For regulated delivery, bundles can be encrypted/packed with [`edc`](bundle-encryption.md)
before distribution.

## Zero-downtime config changes

Edits to watched configuration files are applied live by the config watcher — see
[hot-reload.md](hot-reload.md).

## Next

- [bundle-encryption.md](bundle-encryption.md)
- [secrets-management.md](secrets-management.md)
- [observability.md](observability.md)
