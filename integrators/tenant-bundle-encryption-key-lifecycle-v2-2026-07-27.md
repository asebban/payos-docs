# Tenant Bundle Encryption — Key Lifecycle (Proposal)

Created: 2026-07-24
Last updated: 2026-07-24
Version: v1

## Purpose and scope

This document proposes a single, coherent, end-to-end process for the whole lifecycle of a bundle encryption key: generating an `encryptionKey`, encrypting an application bundle with it, delivering the (partially) encrypted bundle onward — possibly through an integrator — to a client, and having every downstream `payos-runtime` process decrypt it on the fly, in memory, using a key it never persists to disk. Three points are non-negotiable in the model this document proposes: **only the editor is ever allowed to encrypt**; **nobody but the editor's own internal verification is ever allowed to decrypt a bundle to disk** — every other party's `payos-runtime` decrypts on the fly, in memory, at load time, never producing a readable plaintext copy on disk; and **the raw key, once resident in a running process, is protected against casual memory inspection** by splitting it into two XOR'd shares held separately and reconstructing it only transiently, per operation (§7). This document is grounded directly in the current source (`ma.s2m.payos.security.CryptoService`, `ma.s2m.payos.config.ConfigLoader`, `payosv2-packer`'s `Main.java`/`PayOSPacker.java`) rather than in the scattered existing docs (`operations/bundle-encryption.md`, `integrators/decryption mechanism.md`, `integrators/assembling and encrypting a bundle.md`, `integrators/integration-workflow.md`, `operations/cli-tools-guide.md`), which disagree with each other and, in places, with the shipped code — this document calls out every place where it corrects or supersedes them. It is a proposal, not a description of a fully-built pipeline — the "Gaps to close" section below is exactly the list of things that need building before this is production-ready end to end.

## Actors, and who is allowed to encrypt

Three roles appear in the existing integrator documentation (`integrators/integration-workflow.md`) and this document keeps the same names for consistency — but with one correction that changes the whole flow below: **the integrator never encrypts anything.** Only the editor ever runs `edc pack`. Everything the integrator delivers onward — their own overlay files, and the editor's files as received — passes through unencrypted-by-the-integrator: the editor's files stay in whatever encrypted (or plaintext) form the editor originally shipped them in, untouched, and the integrator's own new files are never encrypted at all.

- **Editor** — the core PayOS team producing the base application/platform. **The editor is the only party that ever generates an `encryptionKey` and runs `edc pack`.** This happens once, on the editor's own proprietary code, before it ever leaves the editor's hands.
- **Integrator** — a team (in-house or partner) that customizes the editor's bundle for one specific client, using `extends`/hooks/capabilities without modifying the delivered editor code. **The integrator is never granted decrypt access to the editor's bundle on disk — there is no `edc unpack` step for the integrator, ever, ordinary or otherwise.** What the integrator *can* do is run their own local `payos-runtime` process against the still-encrypted editor bundle, exactly like a client's production instance does: the runtime decrypts on the fly, in memory, file by file, at load time (see §7) — the integrator can execute and test the combined system, but never obtains, and is never granted, a plaintext copy of the editor's files sitting on disk where they could be opened and read. The integrator develops their own overlay against the editor's documented API/data-model contracts, not against readable source. The integrator never encrypts anything either: when delivering onward, they combine the editor's files (still in their original form, byte-for-byte, whatever the editor shipped) with the integrator's own new overlay files (plaintext, never encrypted) and ship that combination as-is. No `edc pack` step happens at this hand-off.
- **Client** (a.k.a. customer) — the organization running the delivered `payos-runtime` instance in production. Needs the same kind of on-the-fly, in-memory decrypt access as the integrator's local runtime — provisioned by the **editor** (directly, or relayed through the integrator without the integrator ever holding a copy of the raw key), never by the integrator generating or granting anything of its own. Also never gets a plaintext copy on disk — same rule as the integrator, for the same reason.

Two delivery shapes both satisfy this model:
- **Editor → Client, direct** (no integrator involved): the editor encrypts, ships the encrypted bundle straight to the client, and provisions the client's on-the-fly decrypt access directly.
- **Editor → Integrator → Client**: the editor encrypts and ships the bundle to the integrator, provisioning the integrator's local runtime with its own scoped on-the-fly decrypt access (never a disk-unpack capability); the integrator adds their plaintext overlay and ships the combination onward; the client's own, separately-scoped on-the-fly decrypt access is provisioned by the editor (the integrator only ever facilitates/relays that access request — it does not mint or hold a client-facing credential of its own, and never gains disk-level plaintext access itself in the process).

## Grounding: what the mechanism actually does today

- **Detection is a 4-byte magic header, checked per file.** `CryptoService.decryptIfEncrypted(byte[] data)` (`payos/src/main/java/ma/s2m/payos/security/CryptoService.java:60-76`) looks at the first 4 bytes of whatever byte array it's handed. `P8G2` → AES/GCM/NoPadding, 12-byte IV, 128-bit tag (authenticated). `P8OS` → AES/ECB/PKCS5Padding (legacy, unauthenticated, decrypt-only). Anything else passes through unchanged — there is no "this must be encrypted or the load fails" mode. This per-file, tolerant-of-plaintext design is exactly what makes the mixed editor-encrypted + integrator-plaintext bundle above work at all: `payos-runtime` doesn't care that some files in the tree carry a magic header and others don't, it just decrypts the ones that do.
- **`ConfigLoader` calls this per JSON file, not once per bundle.** Every time `payos.json` or any file under an application's `configDir` is read, its raw bytes go through `decryptIfEncrypted` before Jackson parses them (`ConfigLoader.java:90,99,109,493,510`). There is no bundle-level manifest or single "unpack everything, then load" step.
- **`edc` (the `payosv2-packer` module) only ever produces `P8OS`.** `PayOSPacker.pack()` walks every regular file under the input directory recursively (`Files.walk(...).filter(isRegularFile)`) with zero exclusion list — `.js`, `.vue`, `.json`, anything — and encrypts each one **in place**, overwriting the original, with `Cipher.getInstance("AES")` (JCE default = AES/ECB/PKCS5Padding). There is no code path anywhere in `payosv2-packer` that writes `P8G2`. `CryptoService` can *decode* `P8G2` (forward-compatible), but nothing currently *produces* it. Several existing docs (`integrators/decryption mechanism.md:22-23`, `operations/cli-tools-guide.md`) state that `edc` produces `P8G2` today — that is incorrect; this document corrects it.
- **The key always comes from the platform's own secret provider**, via `secretProvider.getSecret(tenantId, "encryptionKey")` (`CryptoService.java:139`) — the same `filesystem`/`vault` connector used for every other secret, not a separate mechanism.
- **Tenant scoping for the key is hardcoded to `"default"` today — correctly.** `resolveSecretTenantId()` (`CryptoService.java:156-168`) has a real per-request resolution implementation (MDC tenant, `X-Tenant-Id` header, bootstrap tenant-simulator fallback) sitting **commented out**, and unconditionally returns `DEFAULT_TENANT` (`"default"`). See "Key model" below for why this is the correct behavior for this specific key, not dead code to reactivate.
- **No single-archive artifact exists.** `edc pack` does not produce one `.enc` file the way `integrators/assembling and encrypting a bundle.md:113,131` describes (that doc also references an `--outputdir` unpack flag that does not exist in `Main.java`) — it mutates every file in the directory tree in place. This document corrects that too.

## Key model: `encryptionKey` is bundle-scoped, not tenant-scoped — by design

This is worth stating precisely, because it's the reason `resolveSecretTenantId()` is hardcoded to `"default"`, and that hardcoding is **correct, not a limitation or dead code to reconsider.**

`encryptionKey` is a special case among the platform's secrets. Every *other* secret — a tenant's payment-gateway credentials, a tenant's database password, anything an application reads via `$Secrets`/`ISecretProvider` — is correctly resolved against the **real** business tenant handling the current request, exactly as `multitenancy.tenants` intends, and that resolution already works today (`SecretsBinding` captures the request's actual resolved `tenantId` and passes it straight through to `secretProvider.getSecret(tenantId, name)`). Generic per-tenant secret management is the right model there, and nothing about it needs to change.

`encryptionKey` doesn't fit that model, because of what it protects: **one bundle's files, encrypted exactly once, on disk, shared by every tenant that bundle serves.** A single `payos-runtime` deployment can — and normally does — run multiple business tenants at once (`multitenancy.tenants` listing several entries against one running instance). All of those tenants load the *same* physical encrypted files; there is exactly one ciphertext per file, so there can only be exactly one key that decrypts it. "Tenant A's `encryptionKey`" and "tenant B's `encryptionKey`" isn't a finer-grained security boundary in this case, it's a contradiction — the file was only ever encrypted once, so whichever tenant's request happens to trigger loading it must be able to decrypt it with the same key as every other tenant. That's exactly why `CryptoService.resolveSecretTenantId()` always returns `"default"` regardless of which tenant is making the request: `"default"` here isn't a business tenant identifier at all, it's a fixed, constant slot in the secret store reserved for the one bundle-wide key — unrelated to whatever real tenant IDs exist in `multitenancy.tenants`.

**Do not re-enable the commented-out per-request tenant resolution in `CryptoService`** (MDC tenant / `X-Tenant-Id` / bootstrap tenant-simulator) for this key — doing so would break any bundle serving more than one tenant, since it would look for a per-tenant `encryptionKey` that was never created (and structurally shouldn't be, per the above) instead of the one shared key every tenant actually needs. That dead code was clearly written with the *generic* secret model in mind and just happens to live inside `CryptoService`; it's the right pattern for ordinary tenant secrets and the wrong one for this specific key.

What *is* still a legitimate per-deployment decision, separate from tenant-scoping: the editor may choose to mint a **different `encryptionKey` per delivered bundle/installation** (e.g. one key for client1's installation, a different key for client2's installation, each potentially itself serving several business tenants) so that a key leaked from one installation can't decrypt another installation's copy of the software. That's a decision about how many distinct *bundles* the editor produces, not about tenant-scoping the key within a single bundle — within any one bundle, however many tenants it serves, there is exactly one `encryptionKey`, looked up under the fixed `"default"` slot, for all of them.

## End-to-end flow

### 1. Editor generates the client's encryption key

```bash
spm keygen --out /opt/payos/editor-keys/client1/master.key
```

This is the **master key** (32 random bytes, AES-256) that will wrap every secret in this client's secret store — including the bundle's own `encryptionKey`, generated next. This step, and every step through §4, happens entirely on the **editor's** side — nothing here runs on integrator or client infrastructure.

```bash
spm key-create \
    --root /opt/payos/editor-keys/client1 \
    --keyfile /opt/payos/editor-keys/client1/master.key \
    --tenant default \
    --key-name encryptionKey
```

(`spm key-create` generates a named symmetric key via `ICryptoSecretProvider.generateKey` — see `secret-service-filesystem/README.md`'s CLI section for the full command reference. If Vault is the secret store for this client instead, create the equivalent Transit/KV entry there — see `secret-service-vault/README.md`.)

Generate one such key **per client**, never reuse one client's `encryptionKey` for another's bundle.

### 2. Decide key custody

Two supported models, both already used elsewhere in the platform for the equivalent master-key problem (`secret-service-filesystem/README.md`'s Security Model section) — the **editor** decides this, per client, since the editor is the only party that ever holds the raw key:

| Model | How it works | Use when |
| --- | --- | --- |
| **Filesystem + keyfile, out-of-band delivery** | `master.key` lives on the client's disk, outside the bundle directory, delivered once through a separate secure channel (not bundled with the artifact) — e.g. a secrets manager, a sealed envelope process, a Kubernetes Secret mounted as a file. | Simple, single-instance, on-premise deployments that don't want to run Vault. |
| **Key-blind Vault delivery** | Neither the integrator nor the client ever receives the raw key at all — only a Vault access token scoped by policy to read exactly `secret/data/default/encryptionKey` and nothing else. `payos-runtime` fetches the key from Vault at decrypt time via the `vault` secret provider, on the fly, and holds it only as described in §7. | Anything beyond the simplest deployment — this is the **recommended default**, and the only model where the invariant "the integrator never holds a raw key" is enforced by the infrastructure itself rather than by process discipline. Grant the integrator's local runtime and the client's production runtime **separate** scoped policies — never the same credential — so revoking one never affects the other. Already partially specified in `integrators/integration-workflow.md` §2.2/§10/§12 (Vault policy example, key-ownership models) — that section's framing of the integrator's access as "for key-blind *decryption*" needs re-reading in light of §7 below: it grants the *running process* on-the-fly decrypt capability, not a disk-unpack capability. |

Either way: never place the `encryptionKey`'s custody material inside the bundle directory that gets encrypted (`secret-service-filesystem/README.md` already states this requirement for the master key; it applies identically here).

### 3. Editor scopes what actually gets encrypted

`edc pack` today encrypts every regular file it finds with no exclusions. Before running it, the editor explicitly decides what must stay plaintext because it needs to be readable **before** the secret provider (and therefore the decrypt key) is even available — this is the chicken-and-egg case `ConfigLoader.java:136-137`'s own comment flags: bootstrap settings must be available early enough to initialize the secret provider before any encrypted app config can be decrypted.

Recommended exclusion list for the packing step:
- The `secret-service` configuration block itself (wherever it lives — typically inside `bootstrap.json`) — if `bootstrap.json` as a whole must be encrypted, keep the secret-service credentials resolvable via environment variables (`${ENV_VAR}` placeholders, see `configuration/env-var-resolution.md`) rather than needing decryption first.
- `.capabilities/registry.json` / `activation.json` / `events.ndjson` — read by `ConnectorLoader`/`ConfigWatcher` during bootstrap.
- Connector JARs under `connectors/` — these are loaded by the classloader, not parsed as PayOS config, and encrypting them provides no real protection while adding a decrypt-before-classload step that doesn't exist today.

Everything else that is the editor's own proprietary code (`api/*.js`, `page/*.vue`, `hook/*.js`, `lib/*.js`, the editor application's own `config/*.json`, `i18n/*.json`) is the actual IP being protected and should be encrypted.

### 4. Editor encrypts its own bundle

**Only the editor ever runs this step.**

```bash
edc --encryption pack \
    --inputdir ./build/client1-editor-bundle \
    --secret-provider filesystem \
    --root /opt/payos/editor-keys/client1 \
    --keyfile /opt/payos/editor-keys/client1/master.key \
    --secret-tenant default \
    --secret-name encryptionKey
```

(Substitute the Vault flags from `operations/bundle-encryption.md`/`operations/cli-tools-guide.md` if using key-blind delivery.) Verify the result the same way `operations/bundle-encryption.md` already documents:

```bash
head -c 4 ./build/client1-editor-bundle/config/bootstrap.json | xxd
# Expected: 50 38 4f 53   (P 8 O S — today's actual output; see "Gaps to close" re: P8G2)
```

Because `edc` mutates files **in place** with no backup, always run it against a disposable copy of the assembled editor bundle, never the working tree.

**Nothing downstream of this step ever runs `edc pack` again for this bundle.** If an integrator is in the loop, they receive the output of this step as-is and never re-encrypt it — see §5. If the editor later needs to change something in their own code, they go back to their plaintext source, fix it, and re-run this exact step to produce a new encrypted artifact; there is no "partially re-encrypt" operation.

### 5. Integrator (if any) adds a plaintext overlay, without touching the encrypted files

This is the step that most needed correcting from earlier drafts of this document (and from `integrators/integration-workflow.md`'s own assembly script, §11 — see "Reconciliation" below, which shows the *integrator* running `edc pack`, which is exactly the model this document rejects).

The integrator:
1. Receives the editor's already-encrypted bundle from §4, and (via the key custody model from §2, scoped to the integrator's own local dev/test environment) on-the-fly decrypt access so their **local `payos-runtime` process** can boot and be exercised — never a disk-unpack, never a plaintext copy of the editor's files the integrator could open in an editor/IDE. The integrator runs the system to verify their overlay integrates correctly; they don't — and can't — read the editor's decrypted source.
2. Writes their own new files — an overlay application, hooks, capabilities — that `extends` the editor's application without ever modifying (or needing to read) the editor's delivered files.
3. Assembles a combined bundle: the editor's files exactly as received in §4 (still carrying whatever `P8OS`/`P8G2` header the editor gave them, untouched, byte-for-byte, never decrypted-to-disk at any point) plus the integrator's own new files (plaintext, never encrypted, since the integrator has no encryption capability of its own in this model).
4. **Does not run `edc pack` on the combined result, and does not run `edc unpack` at any point either.** The delivered artifact is a mix — some files encrypted (the editor's, unchanged since §4), some files plaintext (the integrator's own) — and that mix is delivered to the client exactly as assembled. The integrator's own verification that the combined bundle works (e.g. before shipping) is done the same on-the-fly way: run `payos-runtime` against the mixed bundle directly and smoke-test it, not by unpacking it to plaintext first (see the correction to `integrators/assembling and encrypting a bundle.md` §11.4 in "Reconciliation").

If there is no integrator in the loop, this step is skipped entirely and the editor's own output from §4 goes straight to §6.

### 6. Deliver to the client

Whoever is doing the delivering (editor directly, or integrator on the editor's behalf) hands the client the combined artifact from §5 (or the plain output of §4 if there's no integrator), plus decrypt access for the editor-encrypted portion — provisioned by the **editor**:

- **Key-blind (Vault) model:** the editor provisions a scoped read-only policy against exactly the `encryptionKey` path for this client (`integrators/integration-workflow.md` §12.2 has a working example) and grants the client's runtime (or the client's own Vault AppRole) that scoped access directly — never the raw key value, and never routed through a credential the integrator itself holds long-term.
- **Filesystem model:** the editor delivers `master.key` through a channel entirely separate from the bundle artifact, directly to the client — e.g. a secrets-manager-mediated transfer, a separate encrypted email/portal exchange, or a Kubernetes Secret provisioned by the client's own ops team — and confirms its integrity (checksum) out of band from the bundle delivery itself.

In both models, the integrator's role in this step is at most logistics (forwarding the editor's delivery instructions/credentials to the client) — it never mints, holds long-term, or re-derives a key of its own.

### 7. Runtime startup and decrypt-at-load, with in-memory key protection

This section applies identically wherever a `payos-runtime` process runs a bundle containing editor-encrypted files — the integrator's local dev/test box (§5) and the client's production instance (§6) both follow it the same way. Neither party ever runs `edc unpack`; there is no step, for anyone but the editor's own internal verification, that produces a plaintext copy of the editor's files on disk.

**The existing, unchanged part:** `payos-runtime` starts, `BootServer`/`ConfigLoader` initializes the configured secret provider from (plaintext, per §3) bootstrap settings, and from that point every subsequent config file read runs through `CryptoService.decryptIfEncrypted` transparently — files carrying `P8OS`/`P8G2` decrypt in memory, plaintext files (including everything the integrator added in §5) pass through unchanged. This part is already "on the fly" today: `ConfigLoader` never writes a decrypted file back to disk, it decrypts bytes in memory and hands them straight to Jackson/the scripting engine.

**The new part this document proposes — protecting the key while it's resident in memory.** Today, `CryptoService.loadKey()` fetches the raw `encryptionKey` bytes from the secret provider fresh on every decrypt call, uses them, and relies on `SecretValue.close()` zeroing them afterward (see `integrators/decryption mechanism.md`) — the key exists as one contiguous, readable byte array for the (short) duration of each call. To raise the bar against a memory-dump/debugger attack against a running `payos-runtime` process (a real concern precisely because the whole point of on-the-fly decryption is that the process has to be able to reconstruct the key while it runs), split the key instead of holding it whole:

1. **At startup, once:** fetch the raw key from wherever it's custodied (§2) — exactly as today. Generate a same-length random mask via `SecureRandom`. Compute `share = key XOR mask`. Immediately zero the raw key bytes. Hold `mask` and `share` for the process's lifetime, in two separate objects (not two fields of the same object, so they're not trivially found together by a single reference walk).
2. **Per decrypt-on-the-fly call** (i.e. inside what `decryptGcm`/`decryptLegacy` do today): reconstruct `key = mask XOR share` into a fresh, short-lived buffer, use it to initialize the `Cipher`, then zero that reconstructed buffer immediately in a `finally` block — the same "hold it only as long as the operation needs it, then zero it" discipline the codebase already applies to `SecretValue`, just applied to a value that's reconstructed on demand instead of being read fresh from the secret provider every time.
3. The raw, whole key is therefore never resident in memory as one contiguous block for longer than a single `Cipher` operation, at any point after startup — including during the (likely very frequent) steady state of a running instance decrypting many files/requests over its lifetime.

**Be honest about what this does and doesn't buy you**, the same way `integrators/decryption mechanism.md` is honest about the limits of the current simple zero-after-use scheme: splitting the key defeats a *naive* memory scan (e.g. `strings` on a core dump, or a heap walk looking for one 32-byte high-entropy blob) — each half independently is indistinguishable from random noise. It does **not** defend against an attacker who can actively instrument the running JVM (a debugger, a native memory-read primitive, a scripting-sandbox escape) and catch the reconstructed key during the brief window it exists for each `Cipher` call, nor does the JVM give you a hard guarantee that `mask` and `share` are physically non-adjacent in the heap — "two separate objects" is a logical, not a physical, separation guarantee in a managed runtime. Treat this as a real, worthwhile hardening layer that raises the cost of an attack, not as an absolute guarantee — state that plainly to anyone relying on it, exactly as the codebase's existing security docs already do for the current mechanism.

### 8. Rotation and revocation

Only the **editor** can rotate a client's key, since only the editor ever holds encrypt capability for that client's bundle:
1. Generate a new `encryptionKey` for the client (§1) — do not overwrite the old one until the rotation is complete.
2. Re-run §4 against the editor's current plaintext source for that client, producing a new encrypted artifact.
3. If an integrator is in the loop, they redo §5 (drop in the new encrypted editor files, keep their own already-plaintext overlay unchanged) and re-deliver.
4. Deliver the new bundle and grant access to the new key (§6); keep the old key's Vault policy (or keyfile) valid for a defined overlap window in case rollback is needed.
5. Record the key version alongside the delivered bundle version in delivery notes — `operations/bundle-encryption.md` already recommends this; make it a hard checklist item, not just "operational guidance."
6. Revoke access to the old key only once the client confirms the new bundle is running successfully.

### 9. Audit trail

`CryptoService` already logs decryption **failures** via `AuditLogger.logDecryptionFailure(...)` (`CryptoService.java:98,121`). It does not currently log successful decrypt-at-load events. For a full audit trail of "which key decrypted which bundle, when" (useful for the key-version-tracking recommendation above), successful decryption should be logged too — see "Gaps to close."

## Gaps to close before this is production-ready

In priority order:

1. **The split-key/XOR in-memory protection from §7 doesn't exist yet** — `CryptoService.loadKey()` fetches and uses the raw, whole key today; it needs to be reworked to split-once-at-startup/reconstruct-transiently-per-call as described in §7. This is the mechanism this document's newest revision is built around, and it's entirely unbuilt today.
2. **`edc` doesn't produce `P8G2` (authenticated AES/GCM) at all** — everything shipped today is AES/ECB (`P8OS`), which has no integrity/tamper protection and the well-known pattern-leakage weakness of ECB mode. `CryptoService` can already decode `P8G2`; `payosv2-packer` needs a `pack` path that writes it. This is the single highest-value fix for the "securely, from the beginning" goal in the original ask.
3. **`edc unpack` (disk-level decryption) has no access control distinguishing "editor doing internal verification" from "anyone else"** — today it's just a CLI command anyone with the key can run. If the process rule "nobody but the editor ever unpacks to disk" matters as much as this document assumes, `edc unpack` itself is worth restricting (e.g. a separate credential/role from the on-the-fly runtime decrypt path) or removing from any integrator/client-facing distribution of the tool entirely, keeping it as an editor-internal-only build tool.
4. **`edc pack`/`unpack` mutate files in place with no backup** — a failed or interrupted run leaves the editor's bundle in an inconsistent, partially-encrypted state with no way to recover the originals from the artifact itself. Minimum fix: write to a separate output directory (this also happens to be what `integrators/assembling and encrypting a bundle.md` already assumed, incorrectly, already exists as `--outputdir`).
5. **No exclusion list in `edc`** — it would happily encrypt connector JARs and `.capabilities/` state files if pointed at a directory containing them, both of which need to be readable before any decrypt key is available. Needs either a `--exclude` mechanism or a convention `edc` itself understands (e.g. skip `connectors/`, skip `.capabilities/`).
6. **No successful-decryption audit logging** — only failures are logged today; add a success-path audit event to `CryptoService.decryptGcm`/`decryptLegacy` for a complete trail.
7. **Key generation produces a raw 16-character string, not a proper key** (`payosv2-packer/README.md`'s own admission: "Pas de gestion de clés sécurisée (clé passée en clair via CLI)") — `--generatekey` should at minimum support the same 24/32-byte sizes `CryptoService` already accepts, and stop requiring the key to be typed/passed as a plain CLI argument when sourcing it from a secret provider is available.
8. **Nothing in `edc`/`spm` enforces "only the editor may run `pack`"** — this is a process rule in this document, not something the tooling checks. Worth considering whether `edc pack` should require a credential/role distinct from whatever the integrator uses for local decrypt-only access, so the process rule has some technical backing rather than relying entirely on discipline.

None of these block writing this document or following the process above with today's `P8OS`/AES-ECB mechanism — they're what separates "functional" from "PCI-DSS-grade," which the code's own comments (`CryptoService.java:70-71`) already say is the target.

## Reconciliation with existing documentation

| Existing doc | What it gets wrong today | Status after this document |
| --- | --- | --- |
| `integrators/integration-workflow.md` §11 (`build-delivery.sh`) and `integrators/assembling and encrypting a bundle.md` §11.3 | Both show the **integrator** running `edc --encryption pack` on the fully-assembled bundle (editor files + overlay) before delivering to the client — this document's central correction is that the integrator never encrypts anything; only the editor does, once, before the integrator ever receives the bundle. | These two files need the same correction applied here (move the `pack` step to the editor's side, before hand-off, and change the integrator's delivery step to "combine, don't encrypt") — flagged here, not rewritten in full in this pass. |
| `integrators/decryption mechanism.md` | States `P8G2` "is produced by `edc`/`payosv2-packer`" (it is not — decode-only, forward-compatible) | Needs a correction to that one line; everything else in that file is accurate and this document doesn't repeat it. |
| `integrators/assembling and encrypting a bundle.md` §11.3-11.4 | Shows a single `.enc` archive artifact and an `--outputdir` unpack flag; neither exists in `Main.java`/`PayOSPacker.java` (which mutates files in place, `--inputdir` only) | Treat §11.3-11.4's exact commands as aspirational, not current behavior — this document's §4 is the accurate version. |
| `integrators/assembling and encrypting a bundle.md` §11.4 specifically | The "encryption round-trip test" runs `edc unpack` to a plaintext directory (`./build/atlas-verify/`) and boots the runtime against that — this document's §7 says nobody but the editor's own internal verification should ever produce a plaintext copy on disk. | Verification should run `payos-runtime` directly against the still-encrypted artifact (with scoped on-the-fly decrypt access), not against an unpacked copy — needs correcting alongside the actor-model fix above. |
| `operations/bundle-encryption.md` | Accurate — matches the code — but silent on *who* is expected to run `edc`. | No factual change needed; this document adds the actor model (editor-only) and the full lifecycle around it. |
| `operations/cli-tools-guide.md` | Describes a "P8OS header followed by GCM data" hybrid that matches neither format | Needs a correction; out of scope for this document to rewrite in full. |

## References

- `payos/src/main/java/ma/s2m/payos/security/CryptoService.java` — `decryptIfEncrypted`, `loadKey`, `resolveSecretTenantId`.
- `payos/src/main/java/ma/s2m/payos/config/ConfigLoader.java` — where decrypt is actually invoked, per file.
- `payosv2-packer/src/main/java/ma/s2m/PayOSPacker.java`, `Main.java` — what `edc` actually does.
- [operations/bundle-encryption.md](../operations/bundle-encryption.md) — accurate `edc` CLI reference.
- [integrators/decryption mechanism.md](decryption%20mechanism.md) — accurate description of the decrypt side (bar the one `P8G2`-production claim noted above).
- [integrators/assembling and encrypting a bundle.md](assembling%20and%20encrypting%20a%20bundle.md) — the wider bundle-assembly process this document's §4-5 slot into (with the actor correction noted in "Reconciliation").
- [integrators/integration-workflow.md](integration-workflow.md) §2.2, §10, §12 — the key-blind Vault delivery model and delivery checklist this document builds on rather than repeats.
- `secret-service-filesystem/README.md` — master-key security model, `spm` CLI reference.
- `secret-service-vault/README.md` — Vault Transit-backed secret provider.
