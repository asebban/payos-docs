# PayOS Integrator Workflow — From Encrypted Bundle Reception to Customer Delivery

> **Purpose:** This document covers **all operational and project-management steps** of an integration mission — from the moment you receive an encrypted PayOS bundle from the editor, through development, testing, and re-encryption, to the final delivery of a customized bundle to the customer.
> The extensibility mechanisms themselves (overlays, hooks, capabilities, webhooks, secrets, etc.) are documented in [README.md](./README.md). Read that guide alongside this one.

---

## Table of contents

1. [Prerequisites and access checklist](#1-prerequisites-and-access-checklist)
2. [Receiving and unpacking the editor's bundle](#2-receiving-and-unpacking-the-editors-bundle)
3. [Inspecting the delivered bundle](#3-inspecting-the-delivered-bundle)
4. [Setting up your integration workspace](#4-setting-up-your-integration-workspace)
5. [Source-control strategy for the overlay](#5-source-control-strategy-for-the-overlay)
6. [Local development environment](#6-local-development-environment)
7. [Iterative development loop](#7-iterative-development-loop)
8. [Testing strategy](#8-testing-strategy)
9. [Managing multiple environments (dev → staging → production)](#9-managing-multiple-environments-dev--staging--production)
10. [Generating a delivery encryption key](#10-generating-a-delivery-encryption-key)
11. [Assembling and encrypting the final customer bundle](#11-assembling-and-encrypting-the-final-customer-bundle)
12. [Delivering the bundle to the customer](#12-delivering-the-bundle-to-the-customer)
13. [Handling editor version upgrades](#13-handling-editor-version-upgrades)
14. [Ongoing maintenance and patches](#14-ongoing-maintenance-and-patches)
15. [Handover documentation](#15-handover-documentation)

---

## 1. Prerequisites and access checklist

Before starting the mission, confirm the following with the editor and with the customer:

### From the editor

- [ ] Received the **encrypted bundle archive** and the mechanism to decrypt the encrypted files of the bundle (see [Decryption mechanism](./decryption%20mechanism.md)).
- [ ] Received the `payos-runtime-<version>.jar` matching the bundle's applications versions, pre-configured with a vault access token for the integrator.
- [ ] Received the list of **connector JARs** required by the product (secret provider, webhook dispatcher, ... — these are not bundled in the runtime JAR in the integrator packaging but are all in the connectors directory).
- [ ] Optionnaly, either a preconfigured vault instance (secret provider) entirely editor controlled (no access to it by the integrator) or an access to the central editor's vault
- [ ] Received the necessary editor's CLI tools (receive a payos-pm folder containing the Jars and the install scripts of the CLI tools)
- [ ] Confirmed the exact **product identifiers** and their version (e.g. `payment-gateway:2.1.0`).
- [ ] Obtained the **API contract** and data model documentation for the delivered applications.
- [ ] Received the integrator technical user guide for the product
- [ ] Confirmed **kernel version constraints** (Java 21 minimum; GraalVM polyglot for JS).

### From the customer

- [ ] Collected the customer's **technical requirements**: mandatory custom fields, business rules, languages, branding assets, IdP coordinates, webhook target URLs, ...
- [ ] Agreed on the **tenant identifier** (`X-Tenant-Id`) used in all flows.
- [ ] Confirmed the **isolation model** (`shared-schema`, `dedicated-schema`, or `dedicated-database`).
- [ ] Agreed on the **secret management model** (`filesystem` for on-prem / air-gapped; `vault` for centralized production).
- [ ] Agreed on the **delivery format**: containered? orchestrated (Kubernetes)? installation on VM ? Bear-metal ?

### Tools installed on your workstation

- Install Java 21+
- Install maven 3.9.9+
- Install the CLI tools from `payos-pm` module (see [cli-tools/README.md](../cli-tools/README.md)):

```bash
$ cd payos-pm && ./install-payos-tools.sh     # Linux/macOS
$ cd payos-pm && .\install-payos-tools.ps1    # Windows
```

```bash
# Verify tool versions
java -version          # must be 21+
mvn -version           # for building custom connectors / capabilities (if any)
cpm --help             # capability manager to install custom capabilities
apm --help             # application manager to install custom applications
spm --help             # secret manager (filesystem secret manager if required)
```
---

## 2. Receiving and unpacking the editor's bundle

### 2.1 Verifying the bundle artifact

The bundle is received as an archive (ZIP, tar, ...). 

- Copy it in the destination bundle folder where you want to run the application and unzip it there. This folder will be the main folder of the bundle.
- Copy the product licence of the integrator in the main folder of the bundle

PayOS encrypted bundles carry the magic header `P8OS`. Before unpacking, verify integrity:

```bash
# Confirm the magic header is present (first 4 bytes)
# Choose an encrypted file for the test. This case is when payos.json is encrypted
head -c 4 payos.json | xxd
# Expected: 50 38 4f 53   (P 8 O S)
```

If the header is absent or corrupt, contact the editor — do not proceed.

### 2.2 The editor-provisioned Vault for key-blind decryption or an access to the central editor's vault

The editor **does not transmit the raw decryption key**. Instead, the editor delivers:

- **A PayOS runtime pre-built with an embedded shared obfuscated secret key, and an access token** whose Vault policy grants access only to the specific secret path of the AES key.
- Either **A vault local instance pre-configured with the AES symmetric encryption key** or an **access to the editor's central vault**.

The Vault instance hosts the product encryption / decryption AES key. It will be used by the runtime to get the key and decrypt on the fly the encrypted files to use them in the application.

#### Vault policy shipped by the editor (example)

```hcl
# payos-unpack-policy.hcl
path "secret/data/default/encryptionKey" {
  capabilities = ["read"]
}
path "secret/*" {
  capabilities = ["deny"]
}
```

#### Starting the editor-provisioned Vault (in case of a local vault instance)

```bash
# Example: editor ships a Docker image containing the pre-seeded Vault data
docker run -d --name payos-editor-vault \
    -p 8200:8200 \
    -v /path/to/vault-data:/vault/file \
    vault-image-from-editor:2.1.0

export VAULT_ADDR="http://localhost:8200"
```

If the editor configured **Vault auto-unseal** (AWS KMS, Azure Key Vault, GCP KMS, or a Transit seal), the instance unseals automatically on startup — no human involvement needed. Otherwise the editor provides the unseal key (or Shamir shares) through a separate channel.

**The objective of the vault instance is for the runtime to be able to get the encryption key from vault and decrypt the bundle files when needed**.

## 3. Inspecting the delivered bundle

Before writing a single line of customization code, spend time understanding exactly what you have received.

### 3.1 Read the documentation of the delivered bundle

- Installed products and applications with IDs and versions
- Installed / configured servers (https, tcp, ...)
- Installed capabilities
- Extensibility mechanisms ([Extensibility mechanisms](./extensibility-mechanisms.md))
- JPA entity classes by application
- The integrator technical user guide

### 3.2 Read the editor's API contract

Obtain the OpenAPI specification for each delivered application. Normally it is already available in the bundle as a static file, but if not, generate it using `pdoc`:

```bash
pdoc --bundle-path . --app payment-gateway --output ./docs/payment-gateway-api.yaml
```
In general, swagger comes pre-configured with the product. But for information, see [Swagger](../developer/swagger.md).

Study the exposed endpoints, their required roles, path variables, request/response schemas, and any documented hook points (in the technical user guide). This is the contract you must **not break** when overriding endpoints.

### 3.4 Understand the data model

Review the JPA entity classes (if provided by the editor as a separate artifact) or infer the schema from the API contract. Identify:

- Tables owned by the editor's application (you must **never** alter these columns).
- Extension tables you will need to create (prefixed with your client namespace).
- The tenant isolation column pattern (`tenant_id`) used in shared-schema mode.

---

## 4. Setting up your integration workspace

### 4.1 Directory layout recommendation

Maintain a clear separation between the **received bundle** (read-only), your **overlay source**, and **environment-specific configuration**. In general, preferably inherit all delivered applications in the editor's bundle into your custom application. Maintain just one `bootstrap.json` delievered by the editor in which you will add your custom application. To override `bootstrap.json` or `payos.json` entries, create a custom file and name it exactly `custom.json`. You can declare there all `bootstrap.json` or `payos.json` entries you want to override.

So to add your custom application, you can add the `custom.json` file in the editor's bundle directory and declare all customization configurations you want to use in this config file.

> `custom.json` is loaded **after** all other files in the `config/` directory, so any key it declares (including `apps`, server settings, or feature flags) **overrides** the editor's delivered configuration. Keep it focused on your additions and overrides only.

```
integration-workspace/
├── received-bundle/            # encrypted whole editor bundle — READ ONLY, never commit
│   ├── payos.json
│   ├── bootstrap.json
│   ├── custom.json             # Not encrypted (added by the integrator). Will be encrypted upon delivery to the client
│   ├── .capabilities/
│   └── apps/
│      ├── payment-gateway/
│   ├── custom-capabilities/       # optional: custom capabilities you develop
│   │  └── loyalty-points/
├── overlay/                    # YOUR source code — version-controlled
│   └── atlas-payment-gateway/
│       ├── manifest.json
│       ├── api/
│       ├── hooks/
│       └── ...
├── environments/               # env-specific config — version-controlled (no secrets)
│   ├── dev/
│   │   └── bootstrap-overrides.json
│   ├── staging/
│   │   └── bootstrap-overrides.json
│   └── prod/
│       └── bootstrap-overrides.json
├── secrets/                    # local dev secrets only — NEVER commit
│   └── .gitignore              # ensures this directory is excluded
├── build/                      # assembled bundles ready for testing or delivery
│   └── .gitignore
└── docs/                       # integration changelog and customization register
    ├── customization-register.md
    └── delivery-notes.md
```

```json
// received-bundle/custom.json
// This adds 
{
  "applications": [
    {
      "id": "atlas-payment-gateway",
      "name": "Payment Gateway — Atlas",
      "base.path": "../overlay/atlas-payment-gateway",
      "version": "1.2.3",
      "extends": ["payment-gateway"],
      "authorized-tenants": ["atlas"]
    }
  ]
}
```

### 4.2 Protect the received bundle from accidental modification

```bash
# Linux/macOS — make all editor files read-only
chmod -R a-w received-bundle/

# Windows
attrib +R /S /D received-bundle\*
```

Add a top-level `.gitignore` to prevent committing the editor's bundle (except the `custom.json` file):

```
# .gitignore at integration-workspace root
received-bundle/
!received-bundle/custom.json
build/
secrets/
*.enc
```

> The negation rule `!received-bundle/custom.json` re-includes that single file after the broad `received-bundle/` exclusion. Git requires the parent directory itself to not be excluded for negations to work — if your Git version still ignores the file, replace the two lines with a more explicit pattern:
> ```
> received-bundle/**
> !received-bundle/custom.json
> ```

---

## 5. Source-control strategy for the overlay

### 5.1 What to version-control

| Item | Commit? | Notes |
| --- | --- | --- |
| Overlay application source (`overlay/`) | ✅ Yes | Core of your work |
| Custom capability source (`custom-capabilities/`) | ✅ Yes | |
| Environment config skeletons (`environments/`) | ✅ Yes | Without secret values |
| `docs/customization-register.md` | ✅ Yes | Critical for upgrade traceability |
| Received editor bundle (`received-bundle/`) | ❌ No | Proprietary, not yours |
| Secrets (any `.enc`, keys, passwords) | ❌ No | Use Vault or `spm` |
| Assembled build artifacts (`build/`) | ❌ No | Reproducible from source |
| Packed/encrypted delivery bundles | ❌ No | Artifacts, not source |

### 5.2 Tagging against editor versions

Tag every commit that was validated against a specific editor version:

```
git tag -a "editor-payment-gateway-2.1.0" \
    -m "Overlay validated against payment-gateway:2.1.0"
```

This tag is the anchor for the upgrade workflow (see [§13](#13-handling-editor-version-upgrades)).

### 5.3 Branch model

A lightweight branching model for integration work (You may use Gitflow or other similar processes for your development workflow):

```
main            ← stable, validated state; tagged per editor version
develop         ← current integration development
feature/<name>  ← individual feature branches (one per customer requirement)
hotfix/<name>   ← emergency fixes on top of a delivered state
```

---

## 6. Local development environment

### 6.1 Assembling a local working bundle

If you want to structure your development environment in a single bundle directory, you can assemble a **combined bundle** from the received bundle plus your
overlay, without modifying the received bundle:

```bash
# 1. Copy the received bundle to a local working directory
cp -r received-bundle/ build/local-bundle/

# 2. Copy your overlay application into the bundle's apps directory
cp -r overlay/atlas-payment-gateway/ build/local-bundle/apps/

# 3. If you have custom capabilities, copy them too
cp -r custom-capabilities/loyalty-points/ build/local-bundle/.capabilities

# 4. Copy local dev connector JARs (database, secret provider, etc.)
cp connectors/*.jar build/local-bundle/connectors/
```
make sure relative paths in configuration files remain correct.

> The assembly step is typically scripted (see the shell script template in [§11](#11-assembling-and-encrypting-the-final-customer-bundle)).

### 6.2 Bootstrap configuration for local development

Create a `custom.json` overlay file in `build/local-bundle/` that applies development-only settings without modifying the editor's `bootstrap.json`:

```json
// build/local-bundle/custom.json
{
  "multitenancy": {
    "tenantSimulator": { "enabled": true, "tenantId": "atlas" }
  },
  "applications": [
    {
      "id": "atlas-payment-gateway",
      "name": "Payment Gateway — Atlas (DEV)",
      "base.path": "./overlay/atlas-payment-gateway",
      "version": "0.0.0-dev",
      "extends": ["payment-gateway"],
      "authorized-tenants": ["atlas"]
    }
  ]
}
```

The runtime merges all `*.json` files in `configDir` — your dev overrides layer on top of the editor's are in `custom.json`.

### 6.3 Starting the runtime

```bash
java -jar payos-runtime-<version>.jar --bundle-path ./build/local-bundle
```

With hot-reload active, changes to your overlay files (`api/`, `hooks/`, `webhooks.json`, `menu/`, `i18n/`) take effect immediately. Changes to connector JARs and new server / ports require a restart.

### 6.4 Provisioning local development secrets

The filesystem secret provider Jar should exist in connectors/ folder as a prerequisite.

```bash
spm set --root ./secrets/dev \
        --tenant atlas \
        --connectors-dir ./build/local-bundle/connectors \
        --name atlas-psp-api-key \
        --value "sk_test_xxx"
```

The corresponding config for the runtime to be able to read the created keys:

```json
{
  "secret-service": {
    "configuration": {
      "enabled": true,
      "type": "filesystem",
      "root": "./secrets/dev",
      "keyfile": "${config:secret-service.configuration.root}/master.key"
    }
  }
}
```

---

## 7. Iterative development loop

```
┌─────────────────────────────────────────────────────────┐
│ 1. Write/edit overlay files (api/, hooks/, page/, etc.) │
│    (hot-reload — no restart needed)                     │
│                                                         │
│ 2. Send a test request                                  │
│    curl -H "X-Tenant-Id: atlas" http://localhost:8080/  │
│    atlas-payment-gateway/api/payments                   │
│                                                         │
│ 3. Inspect logs                                         │
│    tail -f runtime.log | grep "X-Correlation-Id: ..."   │
│                                                         │
│ 4. If a connector JAR changed → restart runtime         │
│                                                         │
│ 5. Commit to feature branch when unit of work is done   │
└─────────────────────────────────────────────────────────┘
```
See [Extensibility mechanisms](./extensibility-mechanisms.md) for development mechanisms that can be used to extend the application.

### Useful development commands

```bash
# Watch live logs filtered to the atlas tenant
tail -f runtime.log | grep '"tenantId":"atlas"'

# Test an overridden endpoint (with tenant simulator, X-Tenant-Id header is optional)
curl -s http://localhost:8080/atlas-payment-gateway/api/payments \
     -H "Content-Type: application/json" \
     -d '{"amount":100,"currency":"USD","agencyCode":"AT-0042"}' | jq .

# Verify that the overlay script is being served (not the editor's)
curl -s http://localhost:8080/atlas-payment-gateway/api/payments/test-receipt \
     -H "X-Tenant-Id: atlas" | jq .

# Check capability status
cpm --status --all --bundle-path ./build/local-bundle

# Reload a capability activation change without restart (automatic via hot-reload)
cpm --activate --id loyalty-points \
    --app atlas-payment-gateway --tenant atlas \
    --bundle-path ./build/local-bundle
```

---

## 8. Testing strategy

### 8.1 Levels of testing

| Level | What to test | Tool | Run when |
| --- | --- | --- | --- |
| **Unit** | Individual JS scripts in isolation | curl / Postman / Java Unit tests | On every commit |
| **Integration** | Full HTTP pipeline against a running runtime | curl / Postman / Rest-Assured | Before merge to `develop` |
| **Regression** | All editor endpoints still work through `extends` | Same as integration | Before any delivery |
| **Tenant isolation** | Requests with wrong `X-Tenant-Id` are rejected | curl | Before every delivery |

### 8.2 Required integration test cases (minimum)

For each endpoint your overlay **overrides**:
1. Call it with a valid `X-Tenant-Id: atlas` request — verify your override responds.
2. Call the same path on the base application (`/payment-gateway/...`) — verify the editor's script still responds correctly (the overlay must not have side-effected the base app).

For each endpoint your overlay **inherits** (no override):
3. Call it via the overlay path (`/atlas-payment-gateway/...`) — verify it resolves to the editor's script and returns the same response.

For hooks:
4. Trigger a success scenario — verify `$HookChain.proceed()` was reached and the chain ran.
5. Trigger the rejection scenario — verify `$HookChain.stop()` was invoked and the API script did not execute.

For capabilities:
6. Call a capability API endpoint while the capability is **active** — verify 200.
7. Call the same endpoint while the capability is **inactive** — verify 404.
8. Verify that menu entries appear/disappear with capability activation state.

For tenant isolation:
9. Send a request to the overlay with `X-Tenant-Id: unknown-tenant` — expect 403/401.
10. Send a request without any `X-Tenant-Id` header (with `requireTenantId: true`) — expect 400/401.

### 8.3 Regression guard: customization register

Maintain a `docs/customization-register.md` that lists every overridden resource with the editor version it was validated against:

```markdown
## Customization Register — atlas-payment-gateway

| Resource                 | Type                       | Override reason               | Validated against     |
| ------------------------ | -------------------------- | ----------------------------- | --------------------- |
| `api/payments/create.js` | API handler                | Add agencyCode field          | payment-gateway:2.1.0 |
| `hooks/pre-request.js`   | Hook                       | agencyCode validation         | payment-gateway:2.1.0 |
| `page/users/profile.vue` | Page                       | Add loyaltyCardNumber input   | payment-gateway:2.1.0 |
```

Every time the editor ships a new version of the base application, run the full test suite for each row of this register before re-delivery.

---

## 9. Managing multiple environments (dev → staging → production)

### 9.1 What varies per environment

| Setting | Dev | Staging | Production |
| --- | --- | --- | --- |
| `tenantSimulator.enabled` | `true` | `false` | `false` |
| `requireTenantId` | `false` | `true` | `true` |
| Database URL | Local / Docker | Customer sandbox DB | Customer production DB |
| OIDC provider URL | Local Keycloak | Customer staging IdP | Customer production IdP |
| Webhook URLs | `localhost` mock | Staging IS endpoints | Production IS endpoints |
| Secrets | `spm`/local filesystem | `spm`/staging store or Vault | Vault |
| Encryption key | Dev key | Staging key | Production key (customer-held) |
| Log level | `DEBUG` | `INFO` | `INFO` / `WARN` |

### 9.2 Configuration overlay pattern

Never edit the editor's `bootstrap.json` for environment settings. Instead, place an environment-specific JSON file `custom.json` in the main directory of the assembled bundle. The runtime merges all `*.json` files in `configDir` at startup but overrites all duplicate keys by keys defined in `custom.json`.

Secret values referenced via `${VAR}` placeholders in configuration files are injected at container/systemd startup through environment variables — they never appear in files (see [configuration/env-var-resolution.md](../configuration/env-var-resolution.md)).

### 9.3 Database schema migration

Manage your extension tables (`atlas_user_extensions`, etc.) with **Liquibase** or **Flyway** as a separate pipeline step, not from the runtime:

```
CI pipeline step before runtime start:
  liquibase --url="${DB_URL}" \
             --username="${DB_USER}" \
             --password="${DB_PASS}" \
             --changelog-file=db/changelog.xml update
```

One changelog per tenant schema (if using `dedicated-schema`) — parameterize via Liquibase contexts or Flyway targets.

---

## 10. Generating a delivery encryption key

### 10.1 Key ownership decision

Following the same model the editor manages the Vault instance that stores your delivery encryption key, whether it is located on the customer / integrator data center or a central editor's vault :

| Model | Who manages the Vault | Use when |
| --- | --- | --- |
| **Integrator-managed Vault** | The Vault runs in Customer / Integrator site but controlled by the editor | Self-Managed deployment |
| **Shared editor Vault** | The editor manages its central Vault. Customer / Integrator decrypts using the central editor's Vault  | Centrally managed deployment |

---


## 12. Delivering the bundle to the customer

### 12.1 Delivery package contents

Prepare the following delivery package for the customer:

| Item | Description |
| --- | --- |
| `atlas-prod.zip` | The zipped new deliverable artifact developed by the integrator. |
| `editor-bundle.zip` | The zipped original encrypted editor's bundle. You could embed the atlas-prod application or a set of integrator developed applications inside the editor's bundle. |
| `payos-runtime-<version>.jar` | The runtime JAR matching the bundle. |
| Connector JARs | Any connector already embedded in the runtime JAR (database, queue, secret provider, ...) |
| `delivery-notes.md` | Human-readable delivery notes (see [§15](#15-handover-documentation)). |
| `custom.json` (template) | Config customized for client with `${ENV_VAR}` placeholders for the customer to fill in. |
| Systemd/Windows Service unit file (optional) | Service definition for the customer's ops team. |
| Optionnally, a docker image embedding every thing mentioned above | Autonomous docker image embedding all deliverables |
| The Vault dedicated instance docker image or an access to the central editor's vault | The vault that hosts the product decryption key |

**Do NOT include:**
- The raw decryption key — it never leaves Vault
- Development or staging config files.
- Any secret values in plaintext.
- The editor's source code (already encrypted inside the bundle).

### 12.2 Providing Vault access credentials to the customer

Following the key-blind model (§2.2), you **do not transmit the raw decryption key**. Instead, provision an access on the local or central Vault instance. This procedure should normally have been already done by the editor

```bash
# 1. Write the delivery key into your Vault (done once when the key is generated)
vault kv put secret/data/default encryptionKey="$(echo -n 'YOUR_KEY' | base64)"

# 2. Create a scoped policy
vault policy write atlas-unpack-policy - <<EOF
path "secret/data/default/encryptionKey" {
  capabilities = ["read"]
}
EOF
```

### 12.3 Delivery checklist

- [ ] Editor's Bundle artifact carries `P8OS` magic header.
- [ ] Integrator's customization artifacts (could be non encrypted)
- [ ] Round-trip test (unpack → start → health check) passed.
- [ ] `bootstrap-dev.json` / `bootstrap-staging.json` absent from the bundle.
- [ ] `tenantSimulator.enabled` is `false` in all included config files.
- [ ] `requireTenantId` is `true` in the production config.
- [ ] All secret values are `${ENV_VAR}` placeholders — no plaintext credentials.
- [ ] Delivery notes include the editor version, your overlay version, and the capability activation state.
- [ ] Customization register is up to date and attached (or linked) to the delivery notes.

---

## 13. Handling editor version upgrades

When the editor ships a new version of the base application:

### 13.1 Upgrade workflow

```
1. Receive the new encrypted bundle from the editor.
2. Unpack it (§2) to a new directory, e.g. received-bundle-2.2.0/
3. Diff the published application against the previous version:
      diff -rq received-bundle-2.1.0/apps/payment-gateway/ \
               received-bundle-2.2.0/apps/payment-gateway/
4. For every changed file: check the customization register.
      → If the file is listed (you have an override): review whether your override still makes sense, update it if needed, update the register.
      → If the file is not listed: no action (your overlay doesn't touch it).
5. Update the editor bundle reference in your workspace:
      cp -r received-bundle-2.2.0/. received-bundle/
6. Re-run the full test suite (§8).
7. Tag the commit against the new editor version (§5.2).
8. Re-assemble the delivery bundle (§11).
9. Update the delivery notes with the new editor version.
```

### 13.2 Breaking-change identification

Pay special attention to:
- **API endpoint handler name changes**: if the editor renames a handler, your override (which uses the same name) may now shadow a non-existent endpoint or miss the new one.
- **Configuration schema changes**: new mandatory keys added to `bootstrap.json` may need to be reflected in your `custom.json`.
- **Database schema changes**: new columns added to editor-owned tables do not affect your extension tables, but may affect your delegates (`$Api.setApp(...)`) if the response shape changes.
- **Security configuration changes**: OIDC scope or claim name changes may break your hooks or handlers that read `$Principal.get(...)`.

---

## 14. Ongoing maintenance and patches

### 14.1 Customer-reported bug triage

When the customer reports an incident:

1. Identify whether the root cause is in **your overlay** or in the **editor's application**:
   - Check `X-Correlation-Id` in the customer's logs.
   - Reproduce the issue: call the endpoint without your overlay (`/payment-gateway/...` directly). If it fails → editor bug. If it works → overlay bug.
2. **Overlay bug**: fix in your `feature/` or `hotfix/` branch → test → re-assemble → deliver a patch bundle.
3. **editor bug**: open a support ticket with the editor, including the `X-Correlation-Id` and the reproducer. 

### 14.2 Emergency patch delivery

For a critical fix that cannot wait for a full CI pipeline:

```bash
# 1. Fix the file in your overlay source
vim overlay/atlas-payment-gateway/api/payments/create.js

# 2. Re-assemble (using the assembly script, §11.1)
./build-delivery.sh

# 3. Smoke-test
java -jar payos-runtime-<version>.jar --bundle-path ./build/atlas-prod-bundle &
curl -sf http://localhost:8080/health
# ...

# 4. Deliver with an updated delivery note indicating it is a hotfix
```

### 14.3 Capability lifecycle management

When a new capability version is available or a capability must be updated:

```bash
# Uninstall the old version
cpm --uninstall --id loyalty-points --bundle-path ./build/atlas-prod-bundle

# Install the new version
cpm --install \
    --path ./capabilities/loyalty-points-1.3.0 \
    --bundle-path ./build/atlas-prod-bundle \
    --id loyalty-points@1.3.0 \
    --app atlas-payment-gateway \
    --tenant atlas
```

Since capability install/activate is a hot operation (no restart required), you can push
the new capability into a running bundle by updating the bundle directory directly — the
runtime's `ConfigWatcher` will pick up the change.

---

## 15. Handover documentation

Prepare a `delivery-notes.md` for every delivery:

```markdown
# Delivery Notes — atlas-payment-gateway

## Delivery metadata

| Field           | Value                 |
| --------------- | --------------------- |
| Delivery date   | 2026-06-15            |
| Delivered by    | [Integrator name]     |
| Overlay version | 1.2.3                 |
| editor base     | payment-gateway:2.1.0 |
| Runtime version | payos-runtime:4.5.0   |
| Kernel version  | payos-kernel:4.5.0    |

## Bundle contents

| Application           | Version | Extends         | Authorized tenants |
| --------------------- | ------- | --------------- | ------------------ |
| payment-gateway       | 2.1.0   | —               | *                  |
| atlas-payment-gateway | 1.2.3   | payment-gateway | atlas              |

## Capabilities installed

| Capability     | Version | Active for app        | Active for tenant |
| -------------- | ------- | --------------------- | ----------------- |
| loyalty-points | 1.2.0   | atlas-payment-gateway | atlas             |

## What was customized

- `api/payments/create.js` — adds mandatory `agencyCode` field validation and passes through
  to parent.
- `hooks/pre-request.js` — validates `agencyCode` presence on all POST /payments requests.
- `page/users/profile.vue` — adds `loyaltyCardNumber` input field.
- `webhooks.json` — subscribes to `api.post-request` on `/payments` to notify Atlas core
  banking at `${ATLAS_CORE_BANKING_URL}`.

## Environment variables required at startup

| Variable | Purpose |
| --- | --- |
| `ATLAS_DB_PASSWORD` | PostgreSQL password for atlas schema |
| `ATLAS_OIDC_SECRET` | OIDC client secret for atlas-payment-gateway |
| `ATLAS_CORE_BANKING_URL` | Base URL of the Atlas core banking event endpoint |
| `ATLAS_WEBHOOK_SECRET` | HMAC secret for outbound webhook signature verification |

## Vault access

The bundle decryption key is stored in the integrator-provisioned Vault and is **never
transmitted in plaintext**. 

Vault address: `https://[vault-address]:8200`  

## Known issues / limitations

- The `loyaltyCardNumber` field is validated client-side only in this version; server-side
  format validation (`ATLAS-XXXXXXXX`) is enforced by the PUT endpoint.

## Upgrade notes

When the editor delivers a new version of `payment-gateway`, the following overridden
resources must be reviewed against the editor's diff before re-delivery:
- `api/payments/create.js` (overrides editor handler)
- `page/users/profile.vue` (overrides editor component)
```
