## 11. Assembling and encrypting the final customer bundle

### 11.1 Assembly script

Create a repeatable assembly script that builds the delivery bundle from its components.
This script is committed to source control (without secrets):

```bash
#!/usr/bin/env bash
# build-delivery.sh — assembles the production bundle for the atlas customer
set -euo pipefail

editor_BUNDLE="./received-bundle"
OVERLAY_SRC="./overlay/atlas-payment-gateway"
CAPABILITIES_SRC="./capabilities"
OUTPUT_DIR="./build/atlas-prod-bundle"
CONNECTORS_SRC="./connectors/prod"

echo "=== Cleaning output directory ==="
rm -rf "$OUTPUT_DIR"
mkdir -p "$OUTPUT_DIR"

echo "=== Copying editor bundle (read-only base) ==="
cp -r "$editor_BUNDLE/." "$OUTPUT_DIR/"

echo "=== Injecting overlay application ==="
mkdir -p "$OUTPUT_DIR/apps"
cp -r "$OVERLAY_SRC" "$OUTPUT_DIR/apps/atlas-payment-gateway"

echo "=== Injecting custom capabilities (if any) ==="
if [ -d "$CAPABILITIES_SRC" ]; then
  for cap in "$CAPABILITIES_SRC"/*/; do
    cp -r "$cap" "$OUTPUT_DIR/apps/$(basename "$cap")"
  done
fi

echo "=== Injecting production connectors ==="
cp "$CONNECTORS_SRC"/*.jar "$OUTPUT_DIR/connectors/"

echo "=== Copying production config overlay ==="
cp environments/prod/bootstrap-atlas.json   "$OUTPUT_DIR/config/"
cp environments/prod/bootstrap-prod.json    "$OUTPUT_DIR/config/"
# bootstrap-dev.json and bootstrap-staging.json are intentionally NOT copied

echo "=== Running apm to register the overlay application ==="
apm --install \
    --path "$OUTPUT_DIR/apps/atlas-payment-gateway" \
    --bundle-path "$OUTPUT_DIR" \
    --app atlas-payment-gateway

echo "=== Installing and activating capabilities ==="
cpm --install \
    --path "$OUTPUT_DIR/apps/loyalty-points" \
    --bundle-path "$OUTPUT_DIR" \
    --id loyalty-points \
    --app atlas-payment-gateway \
    --tenant atlas

echo "=== Bundle assembly complete: $OUTPUT_DIR ==="
```

### 11.2 Pre-encryption validation

Before encrypting, perform a smoke-test of the assembled bundle:

```bash
# Start the runtime against the assembled (unencrypted) bundle
java -jar payos-runtime-<version>.jar --bundle-path ./build/atlas-prod-bundle &
RUNTIME_PID=$!
sleep 5

# Smoke-test: health check
curl -sf http://localhost:8080/health || { echo "Health check FAILED"; kill $RUNTIME_PID; exit 1; }

# Smoke-test: overlay app reachable
curl -sf -o /dev/null \
     -H "X-Tenant-Id: atlas" \
     http://localhost:8080/atlas-payment-gateway/api/payments || \
     echo "Warning: payment endpoint returned non-2xx (may be auth-protected)"

# Smoke-test: check no bootstrap-dev.json snuck in
ls ./build/atlas-prod-bundle/config/bootstrap-dev.json 2>/dev/null && \
    { echo "ERROR: dev config found in prod bundle"; kill $RUNTIME_PID; exit 1; }

kill $RUNTIME_PID
echo "=== Pre-encryption validation passed ==="
```

### 11.3 Encrypting the delivery bundle

> **Correction (2026-07-24):** two separate issues with §11.3-11.4 below.
> 1. **Actor:** these commands show the *integrator* running `edc pack` on the fully-assembled
>    bundle (editor files + your overlay). Only the **editor** ever encrypts — once, on their own
>    code, before it's ever handed to an integrator. An integrator combines the editor's
>    already-encrypted files (untouched) with their own plaintext overlay and never re-packs.
> 2. **Command shape:** the commands also show a single `atlas-prod-bundle.enc` artifact and an
>    `--outputdir` unpack flag. Neither exists in the actual `edc`/`payosv2-packer` code today —
>    `edc pack`/`unpack` mutate every regular file **in place**, recursively, inside `--inputdir`;
>    there is no single combined archive output and no `--outputdir` flag.
>
> See [tenant-bundle-encryption-key-lifecycle-v2-2026-07-27.md §4-5](tenant-bundle-encryption-key-lifecycle-v2-2026-07-27.md#4-editor-encrypts-its-own-bundle)
> for the corrected actor model and command shape.

```bash
# Using a key stored in the filesystem secret provider
edc --encryption pack \
    --inputdir ./build/atlas-prod-bundle \
    --secret-provider filesystem \
    --root ~/.payos/delivery-keys \
    --secret-tenant delivery \
    --secret-name atlas-delivery-key

# Or using Vault (recommended for teams)
edc --encryption pack \
    --inputdir ./build/atlas-prod-bundle \
    --secret-provider vault \
    --vault-address https://vault.internal:8200 \
    --vault-auth-method approle \
    --vault-role-id "${VAULT_ROLE_ID}" \
    --vault-secret-id "${VAULT_SECRET_ID}" \
    --vault-kv-mount secret \
    --secret-tenant delivery \
    --secret-name atlas-delivery-key
```

The command produces a packed artifact (with `P8OS` header) in the output. Verify the header:

```bash
head -c 4 atlas-prod-bundle.enc | xxd
# Expected: 50 38 4f 53
```

### 11.4 Encryption round-trip test

Always verify that the packed artifact can be successfully unpacked and started:

```bash
edc --encryption unpack \
    --inputdir ./atlas-prod-bundle.enc \
    --secret-provider filesystem \
    --root ~/.payos/delivery-keys \
    --secret-tenant delivery \
    --secret-name atlas-delivery-key \
    --outputdir ./build/atlas-verify/

java -jar payos-runtime-<version>.jar \
     --bundle-path ./build/atlas-verify &
VERIFY_PID=$!
sleep 5
curl -sf http://localhost:8080/health && echo "Round-trip OK" || echo "Round-trip FAILED"
kill $VERIFY_PID
```

---
