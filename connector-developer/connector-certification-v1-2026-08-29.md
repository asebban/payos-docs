# Connector certification

Created: 2026-08-29
Last updated: 2026-08-29
Version: v1

`ConnectorCertificationGate` (module `payos-connector-sdk`, package `ma.s2m.payos.connector.certification`) is a static, offline check that a connector respects the SDK boundary and packaging rules described in [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) and [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md). It doesn't execute the connector and doesn't need a live PayOS runtime — given a built jar and its `pom.xml`, anyone can run it. This page documents what it checks and how to actually run it; running it automates most of the manual checklist in [testing-and-delivery-checklist-v3-2026-08-29.md §2](testing-and-delivery-checklist-v3-2026-08-29.md#2-checklist-before-delivery).

## 1. What it checks

`ConnectorCertificationGate.certify(ConnectorCertificationInput)` returns a `ConnectorCertificationReport` — `report.passed()` is `true` only when `report.findings()` is empty. Each finding carries one of these categories:

| Category | Fails when |
| --- | --- |
| `INVALID_DESCRIPTOR` | No connector descriptor evidence was supplied (see §1 of [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md#1-the-descriptor-meta-infconnectorproperties)). |
| `MISSING_SDK_DEPENDENCY` | No `ma.s2m.payos:connector-sdk` dependency is declared. |
| `INVALID_SDK_SCOPE` | `connector-sdk` is declared but not in `provided` scope. |
| `FORBIDDEN_DEPENDENCY` | Any other `ma.s2m.payos:*` dependency is declared — a connector may depend on `connector-sdk` only. |
| `UNAPPROVED_DEPENDENCY` | A third-party dependency (non-`ma.s2m.payos`, non-`test` scope) isn't in the approved-coordinates set — see [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md) for what "approved" means and where that set comes from. |
| `FORBIDDEN_IMPORT` | Code imports a `ma.s2m.payos.*` type outside `ma.s2m.payos.connector.*`. |
| `INVALID_PACKAGING` | `connector-sdk` itself was bundled/shaded into the jar instead of left `provided`. |
| `MISSING_ISOLATION_EVIDENCE` | The runtime classloader-isolation assumption hasn't been acknowledged. |

## 2. Running it: `ConnectorCertificationCli`

Module `payos-connector-sdk`, package `ma.s2m.payos.connector.certification.approval`. This is the actual entry point — nobody has to hand-build a `ConnectorCertificationInput`:

```
java -cp connector-sdk-<version>.jar ma.s2m.payos.connector.certification.approval.ConnectorCertificationCli \
    --jar   /path/to/the-connector.jar \
    --pom   /path/to/the-connector/pom.xml \
    --registry payos-docs/connector-developer/connector-approved-deps-registry.json \
    --isolation-documented \
    [--import <fully.qualified.Type>]...
```

| Flag | Required | Meaning |
| --- | --- | --- |
| `--jar` | Yes | The connector's built jar — read for `META-INF/connector.properties` (descriptor) and scanned for bundled `connector-sdk` classes. |
| `--pom` | Yes | The connector's `pom.xml` — scanned for its directly declared dependencies. |
| `--registry` | Yes | Path to `connector-approved-deps-registry.json` (§4 of [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md#4-the-approved-dependencies-registry)). |
| `--isolation-documented` | No (default off) | Acknowledges the runtime-isolation assumption — a human sign-off, not something derivable from the artifacts. |
| `--import <type>` | No, repeatable | A fully-qualified type your code imports, to be checked for `FORBIDDEN_IMPORT`. Not auto-derived — would need bytecode scanning, which the CLI doesn't do. |

From `--jar`, `--pom`, and `--registry` alone, the CLI derives:

- The **descriptor** straight from `META-INF/connector.properties` inside the jar (`ConnectorDescriptorParser` — the same parser the runtime itself uses).
- The **dependency list** by scanning the pom's top-level `<dependencies>` block (`ConnectorDependencyScanner` — `<dependencyManagement>`/`<profiles>` entries are ignored, since they aren't what ships in the jar).
- The **approved-coordinate set** by reading the registry JSON and filtering to entries scoped `all connectors` or to this connector's type/name, excluding any entry with a non-null `revoked` (`ApprovedDependencyRegistryReader` / `ApprovedDependencyRegistry.approvedCoordinatesFor(...)`).
- Whether **connector-sdk got bundled**, by checking the jar for any `ma/s2m/payos/connector/**/*.class` entry (those classes belong to the SDK, so their presence means it was shaded in).

Exit codes: `0` (certification passed), `1` (certification failed — findings printed to stdout), `2` (usage error — missing/malformed flags, printed to stderr). The `0`/`1` split makes it usable directly as a CI gate. Treat any `UNAPPROVED_DEPENDENCY` finding as a failure to investigate — a mismatch between what's actually bundled and what's approved for that connector — not something to route around by widening the registry.

## 3. Calling it programmatically

To embed this in your own tooling instead of shelling out to the CLI, use the same pieces directly:

```java
ConnectorDescriptor descriptor = ConnectorDescriptorParser.parse(descriptorInputStream);
List<DependencyCoordinate> dependencies = ConnectorDependencyScanner.scan(pomPath);
ApprovedDependencyRegistry registry = ApprovedDependencyRegistryReader.read(registryJsonPath);
Set<String> approved = registry.approvedCoordinatesFor(descriptor);

ConnectorCertificationInput input = new ConnectorCertificationInput(
        descriptor, dependencies, imports, approved, connectorSdkBundled, isolationDocumented);
ConnectorCertificationReport report = ConnectorCertificationGate.certify(input);
```

See `ConnectorCertificationCliTest` and `ApprovedDependencyCertificationWorkflowTest` (package `ma.s2m.payos.connector.certification.approval`, module `payos-connector-sdk`) for full working examples, including the failure cases (bundled SDK, wrong-scope approval, revoked approval).

## Next

- [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md) — the approved-dependencies registry this reads for `UNAPPROVED_DEPENDENCY`, and how to get a coordinate added.
- [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) — the descriptor and packaging rules this gate enforces.
- [testing-and-delivery-checklist-v3-2026-08-29.md](testing-and-delivery-checklist-v3-2026-08-29.md) — where certification fits in the pre-delivery process.
