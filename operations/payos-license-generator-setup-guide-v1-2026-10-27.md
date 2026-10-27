Created: 2026-10-27
Last updated: 2026-10-27
Version: 1

# payos-license-generator setup guide

A worked, verified walkthrough of building, configuring, and running `payos-license-generator` — the standalone service that generates cryptographically signed, machine-bound PayOS license files, per PRD Journey 1. Every command below was actually run against a real build, a real `secret-service-filesystem` provider, and a real HTTP request — this is not a hypothetical sequence. For the full configuration key reference see [configuration/payos-license-generator-v1-2026-10-27.md](../configuration/payos-license-generator-v1-2026-10-27.md); for the architecture and design decisions behind this service see its own repo's `_bmad-output/planning-artifacts/architecture.md`.

## What this service is, and isn't

`payos-license-generator` exposes one HTTP endpoint, `POST /v1/generate_license`: given client/entitlement/machine data, it generates an AES key, RSA-encrypts it against the editor's public key, computes per-machine SHA-256 fingerprints, assembles the complete license document, signs it with the editor's RSA private key, and writes the signed file to disk. It is the **issuing** side of PayOS licensing — it has no compile-time dependency on `payos-kernel` or any specific secret-provider module (Vault, filesystem, or a future one are all discovered via SPI at runtime), and it is not part of `payos-runtime`. The files it produces are meant to be read by the **validating** side already documented at [configuration/license-service.md](../configuration/license-service.md) and [developer/license-validation.md](../developer/license-validation.md) — `payos-kernel`'s `ILicenseValidator` extension point, where a `SignedLicenseValidator` implementation would verify the `signature` block this service produces against the editor's public key. No such validator ships yet; only `DevLicenseValidator` (no-op) and `LicenseValidator` (file-presence check only) exist today.

## 1. Build the service

```bash
cd payos-license-generator
mvn -o clean package
```

Produces a single executable shaded jar at `target/payos-license-generator-<version>.jar` — `payos-foundation`, Undertow, Jackson, SLF4J/Logback are all bundled; no secret-provider implementation is bundled (confirmed by `jar tf target/payos-license-generator-*.jar | grep -i secretprovider` returning nothing beyond `payos-foundation`'s own interface-naming constants).

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

Every key here is optional — see the configuration reference for defaults and the full system-property/environment-variable/config-file/default resolution order.

## 7. Run it

```bash
java -jar payos-license-generator-<version>.jar --generator-path <path to the bundle dir above>
```

`--generator-path` is optional; if omitted, every relative path in the config (the config file's own location, `secret-provider-dir`, `license-output-dir`) resolves against the process's working directory instead. On a clean start you should see:

```
INFO ma.s2m.payos.license.secret.SecretProviderLoader -- Secret-provider JAR registered: spm.jar
INFO io.undertow -- starting server: Undertow - 2.3.18.Final
...
payos-license-generator listening on port 8099 (version <version>)
```

## 8. Generate a license

```bash
curl -s -X POST http://localhost:8099/v1/generate_license \
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

Returns the complete, signed license document as the response body, and writes the identical JSON to `licenses/<license_id>.json` — verified in a real run that the two are byte-for-byte identical (`diff` against the HTTP response). Every field under `issuer`/`licensee`/`product`/`validity`/`entitlements`/`machine_bindings.identifiers` is carried through from the request unchanged; `format`, `format_version`, `license_id`, `issued_at`, `machine_bindings.fingerprints`, `key_envelope`, and `signature` are generated by the service.

## MVP limitations (by design, not oversight)

No authentication — the request above needs no `Authorization` header at all. This is a deliberate MVP decision (see the source PRD's Epic 5), not an omission: the HTTP layer has an auth seam ready, but nothing is wired to it yet, so this service must only be deployed on a trusted internal network for now. No payload validation either yet — a malformed request currently surfaces as a generic 500, not the field-level 4xx error the PRD specifies (Epic 2, not yet implemented). No request-rate limiting, no persistence of generation metadata, no metrics endpoint yet (Epic 4, not yet implemented).
