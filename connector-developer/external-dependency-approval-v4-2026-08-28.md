# External dependency approval for connectors

Created: 2026-08-28
Last updated: 2026-08-28
Version: v4

This page defines how a connector author gets a third-party (non-PayOS) library approved for use in a business/payment connector, and where the list of already-approved libraries lives. It complements [packaging-and-deployment-v1-2026-07-27.md §3](packaging-and-deployment-v1-2026-07-27.md#3-deployment-connectorsjson), which covers how to package a connector jar, and [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md), which covers the `IConnector` contract itself.

## 1. Why this exists

A connector is allowed to bundle third-party libraries directly into its own jar — each connector loads in its own isolated classloader (see [packaging-and-deployment-v1-2026-07-27.md §3](packaging-and-deployment-v1-2026-07-27.md#3-deployment-connectorsjson)), so two connectors can safely bundle different, even conflicting, versions of the same library. That isolation removes the *technical* conflict risk, but it doesn't remove the need for platform-level oversight: PayOS is a financial/PCI DSS-scoped platform, and every third-party library that ends up executing inside a connector jar is code the platform is ultimately trusting with payment data, even if it never imports a PayOS package directly.

`ConnectorCertificationGate` (module `payos-connector-sdk`, package `ma.s2m.payos.connector.certification`) enforces this at certification time: any dependency that isn't `ma.s2m.payos:connector-sdk`, isn't another forbidden `ma.s2m.payos:*` artifact, and isn't scope `test`, must appear in `ConnectorCertificationInput.approvedExternalDependencies` or certification fails with `UNAPPROVED_DEPENDENCY`. This page defines what "approved" means and how a library gets there — the certification gate itself only checks the list; it has no opinion on how the list is populated.

## 2. Scope

**Needs approval:** any third-party (non-`ma.s2m.payos`) library your connector bundles into its delivered jar and that ships in a scope other than `test` — HTTP clients, JSON/XML libraries, crypto/signing libraries, SDKs provided by the external payment network or bank you're integrating with, etc.

**Does not need approval:** `ma.s2m.payos:connector-sdk` itself (it's the one PayOS artifact connectors are required to depend on, in `provided` scope); libraries scoped `test` in your `pom.xml` (the gate already exempts them, since they never ship in the delivered jar); the JDK's own standard library.

## 3. Approval criteria

A library is approved only if all of the following hold:

- **License compatibility.** Permissive licenses (Apache-2.0, MIT, BSD-2/3-Clause) are approved by default. Strong copyleft licenses (GPL, AGPL, LGPL where static linking applies) require an explicit legal/compliance sign-off before approval, since PayOS ships as commercial banking middleware — do not assume a copyleft license is fine without that sign-off.
- **Pinned exact version.** The approval covers one specific version (`groupId:artifactId:version`), not a version range. A version bump is a new approval request (see §5), even for a patch release, so the registry always reflects exactly what was reviewed.
- **No known critical or high-severity vulnerability** against the requested version at review time, per whatever vulnerability database/scanner the reviewer checks (e.g. the OSS index / NVD for the exact version). A library with an open critical/high CVE and no fixed release is rejected until a patched version is requested instead.
- **No native/JNI component.** A library that loads native code (`.so`/`.dll` via JNI/JNA) runs outside the JVM's classloader boundary and can undermine the per-connector isolation guarantee described in §1 — these require an explicit architecture review beyond this checklist, not a routine approval.
- **Actively maintained**, or if unmaintained, small and simple enough that the reviewer is comfortable the connector team can fork/patch it if a vulnerability surfaces later.
- **Not a duplicate** of functionality already available through `connector-sdk` or the JDK — if the same job can be done without adding a dependency, prefer that.

## 4. The approved dependencies registry

The source of truth for what's currently approved is [connector-approved-deps-registry.json](connector-approved-deps-registry.json), a structured JSON file that sits alongside this document. A coordinate not listed there is not approved, regardless of how reasonable it seems. It's JSON rather than a table in this page so that tooling (§6) can consume it directly instead of scraping prose, and so a malformed entry fails a schema check in CI instead of silently vanishing.

Each element of its top-level `entries` array has this shape:

```json
{
  "coordinate": "com.visa:client",
  "version": "3.1.0",
  "requestedFor": "CardNetwork / visa",
  "approvedBy": "a.reviewer",
  "approvedDate": "2026-08-20",
  "scope": "visa",
  "notes": "VisaNet SDK",
  "revoked": null
}
```

| Field | Meaning |
| --- | --- |
| `coordinate` | `groupId:artifactId`, no version — the version is tracked separately below. |
| `version` | The exact reviewed version (§3: "pinned exact version", not a range). |
| `requestedFor` | Free-text record of who asked and for what, for traceability — not read by tooling. |
| `approvedBy` / `approvedDate` | Filled in by the reviewer on approval; `null` while a request is still pending review. |
| `scope` | Either `"all connectors"` (cleared for any connector) or the specific connector `type` or `name` it was reviewed for (see [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md) for that distinction) — use a specific scope by default, broaden to `"all connectors"` only on a second request that explicitly asks for it. |
| `notes` | Free text, e.g. what the library is for. |
| `revoked` | `null` while the approval stands; otherwise `{"date": "...", "reason": "..."}` — see §7. |

An empty registry is `{ "entries": [] }`.

## 5. Request process

1. **Check the registry (§4) first.** If the exact coordinate and version you need is already listed with a scope that covers your connector, you're done — no new request needed.
2. **Open a request** as a pull request against `connector-approved-deps-registry.json`, appending an entry with `coordinate`, `version`, `requestedFor`, and `notes` filled in, and `approvedBy`/`approvedDate`/`revoked` left `null` — the reviewer fills those in.
3. **Reviewer checks the request against §3** (license, pinned version, vulnerability status, no native component, maintenance status, not a duplicate). Today this reviewer is the PayOS Platform team (connector-sdk / connector framework maintainers) — confirm this against your actual team structure and update this line if ownership differs.
4. **On approval**, the reviewer fills in `approvedBy` and `approvedDate` and merges the PR. On rejection, the reviewer replies on the PR with the specific criterion that failed; the requester can resubmit against a different version (e.g. a patched release) or a different library.
5. **Use the approved coordinate** in your `pom.xml` at normal (non-`provided`) scope, and bundle it into your connector jar per [packaging-and-deployment-v1-2026-07-27.md §3](packaging-and-deployment-v1-2026-07-27.md#3-deployment-connectorsjson).

## 6. How this feeds certification today

`ConnectorCertificationGate.certify(...)` reads `ConnectorCertificationInput.approvedExternalDependencies` as a plain `Set<String>` of `groupId:artifactId` coordinates — it does not read this document directly. Module `payos-connector-sdk`, package `ma.s2m.payos.connector.certification.approval`, provides the actual certification entry point, `ConnectorCertificationCli`, so nobody has to hand-copy coordinates into a `ConnectorCertificationInput`:

```
java -cp connector-sdk-<version>.jar ma.s2m.payos.connector.certification.approval.ConnectorCertificationCli \
    --jar   /path/to/the-connector.jar \
    --pom   /path/to/the-connector/pom.xml \
    --registry payos-docs/connector-developer/connector-approved-deps-registry.json \
    --isolation-documented
```

Given those three paths, the CLI derives everything it can and runs `ConnectorCertificationGate.certify(...)` itself:

- The connector **descriptor** (type, name, API version, required params) comes straight from `META-INF/connector.properties` inside the jar (§1 of [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md#1-the-descriptor-meta-infconnectorproperties)) — the same file the runtime itself reads, so there's no separate place to retype it.
- The **dependency list** comes from scanning the connector's `pom.xml` (`ConnectorDependencyScanner`) — only the top-level `<project><dependencies>` block, since `<dependencyManagement>`/`<profiles>` entries aren't what actually ships in the jar.
- The **approved-coordinate set** comes from parsing `connector-approved-deps-registry.json` (`ApprovedDependencyRegistryReader` / `ApprovedDependencyRegistry.approvedCoordinatesFor(...)`), filtered to entries scoped `all connectors` or to this connector's type/name, with any entry carrying a non-null `revoked` (§7) excluded.
- Whether **connector-sdk got bundled** is detected by checking the jar for any `ma/s2m/payos/connector/**/*.class` entry — those classes belong to the SDK itself, so their presence means it was shaded in rather than left `provided`.

Two things are intentionally *not* derived automatically, and stay explicit CLI inputs: the imported-types list (`--import <fully.qualified.Type>`, repeatable — would need bytecode scanning, which this doesn't do) and the runtime-isolation acknowledgement (`--isolation-documented` — a human sign-off, not a fact discoverable from the artifacts). The CLI exits `0` when certification passes and `1` when it doesn't, so it's usable as a CI gate. See `ConnectorCertificationCliTest` in that package for the full wiring, including how a bundled-SDK jar and a wrong-scope approval each get caught.

Treat any `UNAPPROVED_DEPENDENCY` finding that comes out of this — a mismatch between what's actually bundled in the jar and what's listed as approved for that connector — as a certification failure to investigate, not something to route around by widening the registry's scope.

`ApprovedDependencyRegistryReader` deserializes `connector-approved-deps-registry.json` strictly: an entry missing a required field, or the file containing an unrecognized field, fails to load rather than being silently skipped — a malformed edit to the registry surfaces as a read error, not a quietly-wrong approved set.

## 7. Revocation and re-review

If a previously approved coordinate is later found to have a critical/high vulnerability, a license problem, or is otherwise withdrawn, the reviewer sets that entry's `revoked` field to `{"date": "<date>", "reason": "<reason>"}` rather than deleting the entry, so the registry keeps an audit trail of what was approved and when (relevant for PCI DSS change-management evidence). A revoked coordinate must not be used in new certifications — `ApprovedDependencyRegistry.approvedCoordinatesFor(...)` excludes any entry with a non-null `revoked`, so `ConnectorCertificationCli` rejects it automatically; connectors already shipping with it should be flagged to their owning team for a version bump or replacement.

## Next

- [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) — descriptor, SPI registration, classloader isolation, `connectors.json`.
- [testing-and-delivery-checklist-v2-2026-08-28.md](testing-and-delivery-checklist-v2-2026-08-28.md) — pre-delivery checklist, which should be read alongside this page before shipping a connector that bundles third-party libraries.
