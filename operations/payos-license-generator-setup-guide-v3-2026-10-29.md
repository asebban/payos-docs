Created: 2026-10-27
Last updated: 2026-10-29
Version: 3 — supersedes v2: step 9 generalizes persistence beyond H2 to any JDBC database.

# payos-license-generator setup guide

A worked, verified walkthrough of building, configuring, and running `payos-license-generator` — the standalone service that generates cryptographically signed, machine-bound PayOS license files, per PRD Journey 1. Every command below was actually run against a real build, a real `secret-service-filesystem` provider, and a real HTTP request — this is not a hypothetical sequence. For the full configuration key reference see [configuration/payos-license-generator-v3-2026-10-29.md](../configuration/payos-license-generator-v3-2026-10-29.md); for the architecture and design decisions behind this service see its own repo's `_bmad-output/planning-artifacts/architecture.md`.

## What this service is, and isn't

`payos-license-generator` exposes one HTTP endpoint, `POST /v1/generate_license`: given client/entitlement/machine data, it validates the payload, generates an AES key, RSA-encrypts it against the editor's public key, computes per-machine SHA-256 fingerprints, assembles the complete license document, signs it with the editor's RSA private key, and writes the signed file to disk. It is the **issuing** side of PayOS licensing — it has no compile-time dependency on `payos-kernel` or any specific secret-provider module (Vault, filesystem, or a future one are all discovered via SPI at runtime), and it is not part of `payos-runtime`. The files it produces are meant to be read by the **validating** side already documented at [configuration/license-service.md](../configuration/license-service.md) and [developer/license-validation.md](../developer/license-validation.md) — `payos-kernel`'s `ILicenseValidator` extension point, where a `SignedLicenseValidator` implementation would verify the `signature` block this service produces against the editor's public key. No such validator ships yet; only `DevLicenseValidator` (no-op) and `LicenseValidator` (file-presence check only) exist today.

A caller-fault payload error returns 4xx naming the offending field; an infrastructure-fault failure (secret provider unreachable or misconfigured) returns 5xx; every request, successful or failed, is tagged with a correlation ID returned in the `X-Correlation-Id` response header and present on every log line produced while handling it. Every attempt is also recorded as one structured JSON log line, and optionally as a row in any JDBC-reachable database — see steps 8 and 9 below.

## 1. Build the service

```bash
cd payos-license-generator
mvn -o clean package
```

Produces a single executable shaded jar at `target/payos-license-generator-<version>.jar` — `payos-foundation`, Undertow, Jackson, SLF4J/Logback, `logstash-logback-encoder`, and the embedded H2 database engine are all bundled; no secret-provider implementation is bundled (confirmed by `jar tf target/payos-license-generator-*.jar | grep -i secretprovider` returning nothing beyond `payos-foundation`'s own interface-naming constants).

## 2. Build a secret-provider JAR

The service discovers its secret provider via SPI from an external, configured directory — it never compile-depends on one. For the filesystem provider, the shaded CLI jar (`spm.jar`) already bundles everything `ServiceLoader` needs (including its `META-INF/services/ma.s2m.payos.secret.api.ISecretProviderFactory` registration):

```bash
cd secret-service-filesystem
mvn -o clean package
# target/spm.jar is both the operator CLI and a valid SPI-discoverable provider JAR
```

For Vault, package `secret-service-vault` the same way and configure `secret-provider-type: "vault"` instead (see the configuration reference for Vault-specific keys).

## 3. Set up the bundle directory

Pick a base directory (the `--generator-path`, see below) and lay out:

```
<generator-path>/
  connectors/
    spm.jar                          # the SPI-discoverable provider JAR
  secrets/
    master.key                       # filesystem provider's AES-256 master key
    store/                           # filesystem provider's key storage root
  licenses/                          # where generated license files land
  license-generator-config.json      # this service's own config
```

```bash
mkdir -p connectors secrets/store licenses
cp /path/to/secret-service-filesystem/target/spm.jar connectors/
```

## 4. Generate the master key

```bash
java -jar connectors/spm.jar keygen --out secrets/master.key
```

Writes a random 32-byte AES-256 key. Treat it as a high-value secret (see [cli-tools/spm.md](../cli-tools/spm.md)'s Safety section) — anyone with it can decrypt every key the filesystem provider ever generates under this root.

## 5. Generate the editor's RSA key pair

```bash
java -jar connectors/spm.jar keypair-create \
  --root secrets/store --keyfile secrets/master.key \
  --tenant editor --key-name editor-signing-key --algorithm RSA_2048
```

Prints the public key PEM to stdout — the private key is generated, stored under `secrets/store`, and never printed or exported. `--tenant`/`--key-name` must match `editor-tenant-id`/`editor-key-name` in the service's own config (defaults: `editor`/`editor-signing-key`) — this is the one manual bootstrap step every fresh deployment needs before the service can sign anything; there is currently no automated first-run provisioning for it.

## 6. Write the config file

`license-generator-config.json`, at the bundle root:

```json
{
  "secret-provider-dir": "connectors",
  "secret-provider-type": "filesystem",
  "secret-provider-config": {
    "root": "secrets/store",
    "keyfile": "secrets/master.key"
  },
  "editor-tenant-id": "editor",
  "editor-key-name": "editor-signing-key",
  "license-output-dir": "licenses",
  "http-port": "8099"
}
```

Every key here is optional — see the configuration reference for defaults and the full system-property/environment-variable/config-file/default resolution order. Four more keys, `persistence-enabled` and `persistence-jdbc-url`/`persistence-jdbc-driver`/`persistence-jdbc-username`/`persistence-jdbc-password`, control the optional database persistence covered in step 9 below; they're omitted here because persistence is off by default.

## 7. Run it

```bash
java -jar payos-license-generator-<version>.jar --generator-path <path to the bundle dir above>
```

`--generator-path` is optional; if omitted, every relative path in the config (the config file's own location, `secret-provider-dir`, `license-output-dir`, and the default embedded-H2 `persistence-jdbc-url`) resolves against the process's working directory instead. On a clean start you should see one JSON object per log line (every log line the service emits is structured JSON — see the configuration reference's Logging section), followed by a final plain-text line once the HTTP listener is actually bound:

```
{"@timestamp":"2026-10-28T00:10:32.79+01:00","@version":"1","message":"Secret-provider JAR registered: spm.jar","logger_name":"ma.s2m.payos.license.secret.SecretProviderLoader","thread_name":"main","level":"INFO","level_value":20000}
{"@timestamp":"2026-10-28T00:10:33.08+01:00","@version":"1","message":"starting server: Undertow - 2.3.18.Final","logger_name":"io.undertow","thread_name":"main","level":"INFO","level_value":20000}
...
payos-license-generator listening on port 8099 (version <version>)
```

## 8. Generate a license

```bash
curl -si -X POST http://localhost:8099/v1/generate_license \
  -H "Content-Type: application/json" \
  -d '{
    "issuer": {"name": "S2M"},
    "licensee": {"client_id": "client_12345", "name": "Example Client Ltd"},
    "product": {"name": "PayOS", "edition": "Enterprise"},
    "validity": {"not_before": "2026-04-19T00:00:00Z", "expires_at": "2027-04-19T23:59:59Z"},
    "entitlements": {"services": ["payments", "webhooks"]},
    "machine_bindings": {
      "algorithm": "SHA-256",
      "identifiers": [
        {"machine_id": "machine_01", "mac_address": "AA:BB:CC:DD:EE:FF", "cpu_id": "CPU123", "motherboard_uuid": "UUID456"}
      ]
    }
  }'
```

The `-i` flag shows the response headers, including `X-Correlation-Id: <uuid>` — the same value that would appear in every log line for this request, and in the `correlation_id` field of an error body if the request had failed instead. The response body is the complete, signed license document, and the identical JSON is written to `licenses/<license_id>.json` — verified in a real run that the two are byte-for-byte identical (`diff` against the HTTP response). Every field under `issuer`/`licensee`/`product`/`validity`/`entitlements`/`machine_bindings.identifiers` is carried through from the request unchanged; `format`, `format_version`, `license_id`, `issued_at`, `machine_bindings.fingerprints`, `key_envelope`, and `signature` are generated by the service.

A malformed request (here, a missing `cpu_id`) is rejected before any cryptographic work or file write, naming the offending field:

```bash
curl -s -X POST http://localhost:8099/v1/generate_license -H "Content-Type: application/json" \
  -d '{"issuer":{"name":"S2M"},"licensee":{"client_id":"c"},"machine_bindings":{"identifiers":[{"machine_id":"m1","mac_address":"AA","motherboard_uuid":"U"}]}}'
# {"error":"machine_bindings.identifiers[0].cpu_id is required","correlation_id":"<uuid>"}
```

An infrastructure fault (the configured signing key missing or the secret provider unreachable) returns 503 instead of 500, with the same correlation-ID-tagged body shape — distinguishing "fix your request" from "the service or its secret provider has a problem" without needing to read logs first.

## 9. Enable optional persistence (any JDBC database)

By default every attempt is only logged (step 7's JSON lines already include one `"License generation attempt completed"` entry per request, successful or not). To additionally retain a queryable history, add to `license-generator-config.json`:

```json
{
  "persistence-enabled": "true",
  "persistence-jdbc-url": "jdbc:h2:file:db/license-generator"
}
```

Persistence is a plain JDBC connection, not an H2-specific feature — `persistence-jdbc-url` can point at any database, and `persistence-jdbc-driver`/`persistence-jdbc-username`/`persistence-jdbc-password` go with it (see the configuration reference's Persistence section for the PostgreSQL-style example). The example above keeps the bundled, zero-setup default: an embedded H2 file, needing no separate server or driver JAR, which is what the rest of this step walks through.

Restart the service and generate a license as in step 8 — the directory `db/` and the file `db/license-generator.mv.db` are created automatically on first use, no migration step needed. Stop the service, then inspect the database directly with H2's own shell tool (the jar is already on the classpath inside the service's own shaded jar, or fetch `com.h2database:h2:2.3.232` separately):

```bash
java -cp h2-2.3.232.jar org.h2.tools.Shell \
  -url "jdbc:h2:file:/absolute/path/to/db/license-generator" -user sa -password "" \
  -sql "SELECT client_id, license_id, generated_at, outcome, error_category FROM license_generation_attempts;"
```

This was verified against a real run: a successful request produced a row with `outcome = SUCCESS` and `error_category = null`, and a validation failure produced a row with `license_id = null`, `outcome = VALIDATION_FAILURE`, `error_category = VALIDATION_FAILURE` — both still present after forcibly killing the server process, confirming H2 durably committed them rather than only buffering in memory. One real gotcha hit while verifying this: the JDBC URL needs a genuine Windows-style path (`C:/...`) on Windows — a Git Bash `$(pwd)`-style POSIX path (`/c/...`) is silently reinterpreted by the JVM as a path from the current drive's root and points at the wrong file entirely, so always pass an absolute path with a drive letter when running the Shell tool manually from Git Bash.

Switching to another vendor (PostgreSQL, MySQL, etc.) reuses this exact mechanism — set `persistence-jdbc-url`/`persistence-jdbc-driver`/`persistence-jdbc-username`/`persistence-jdbc-password` and put that vendor's driver JAR on the classpath yourself, since only H2's driver is bundled. This has not been re-verified here against a live non-H2 server; the code path is identical to the H2 case above (same `CREATE TABLE IF NOT EXISTS`/insert statements over a standard `java.sql.Connection`), and `CREATE TABLE IF NOT EXISTS` is supported by PostgreSQL and MySQL/MariaDB the same way it is by H2. A vendor without that syntax (Oracle, SQL Server) needs the table pre-created with the schema in the configuration reference before you enable persistence against it.

## MVP limitations (by design, not oversight)

No authentication — the request above needs no `Authorization` header at all. This is a deliberate MVP decision (see the source PRD's Epic 5), not an omission: the HTTP layer has an auth seam ready, but nothing is wired to it yet, so this service must only be deployed on a trusted internal network for now. No Prometheus metrics endpoint yet either (Epic 4's Story 4.3, not yet implemented) — for now, operational visibility comes from the structured JSON logs (step 7) and, if enabled, the persistence table (step 9).
