# Getting started: writing a connector from scratch

Created: 2026-09-07
Last updated: 2026-09-07
Version: v1

The other pages in this folder each document one piece of the connector lifecycle in depth, but none of them walks through the whole path in order starting from an empty directory — this page is that walkthrough. It does not replace the other pages; every step below links to the page that covers that step's full detail, and this page exists only to sequence them and to fill in the practical setup steps (in particular the `pom.xml` and local Maven repository setup) that the other pages assume you already have. Read this page first if you're starting a connector project for the first time; use the other pages as reference once you're past setup.

## 1. Prerequisites

You need JDK 21 and Maven 3.9+ — `IConnector` and `connector-sdk` are built against Java 21, and a connector jar compiled against an older release is not guaranteed to load. `JAVA_HOME` must point at a JDK 21+ installation, and `mvn --version` must resolve to 3.9 or later.

There is currently no PayOS-hosted Maven repository (no Nexus/Artifactory) that a third-party connector project can point at to resolve `ma.s2m.payos:connector-sdk` — the only supported path today is building the required modules from source and letting Maven install them into your own local `~/.m2` repository, exactly as any other PayOS developer does (see [developer/build-guide.md](../developer/build-guide.md)). Your connector project's own `pom.xml` (§2 below) then resolves `connector-sdk` from that local repository like any other dependency — no repository/credentials block is needed in your `pom.xml` or `settings.xml`.

You only need four modules installed to resolve `connector-sdk` — not the full runtime build. In this exact order, from the root that contains all the PayOS repositories as siblings:

```bash
cd payos-parent          && mvn -q -DskipTests install
cd ../payos-bom           && mvn -q -DskipTests install
cd ../payos-connector-api && mvn -q -DskipTests install
cd ../payos-connector-sdk && mvn -q -DskipTests install
```

This order matters: `payos-bom` imports `payos-parent`, `payos-connector-api` imports `payos-bom`, and `connector-sdk` (folder `payos-connector-sdk`) depends on `payos-connector-api` — installing out of order fails with an "artifact not found" error from Maven. Verify the last step worked by confirming `~/.m2/repository/ma/s2m/payos/connector-sdk/1.2.0-RELEASE/connector-sdk-1.2.0-RELEASE.jar` exists; if it does, your machine can now build any connector project against `connector-sdk` without needing these four repositories again, until a newer SDK version is released.

## 2. Create your connector project

Your connector is its own standalone Maven project — it does **not** inherit from `payos-parent` (that parent POM carries PayOS's own internal build conventions and isn't meant to be a third party's parent) and it is not built from inside any PayOS repository. Pick your own `groupId`/`artifactId`; only the Maven *coordinate* of the dependency on `connector-sdk` is fixed, not your own project's coordinates.

Directory layout:

```
my-connector/
├── pom.xml
└── src
    ├── main
    │   ├── java/com/example/connectors/cmi/CmiPaymentConnector.java
    │   └── resources
    │       └── META-INF
    │           ├── connector.properties
    │           └── services/ma.s2m.payos.connector.api.IConnector
    └── test
        └── java/com/example/connectors/cmi/CmiPaymentConnectorTest.java
```

A minimal, complete `pom.xml` that resolves against the four modules installed in §1:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.connectors</groupId>
    <artifactId>cmi-connector</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- The only ma.s2m.payos:* dependency a connector is allowed to declare — see
             packaging-and-deployment-v1-2026-07-27.md §3. Must stay "provided": the runtime
             supplies this jar itself at execution time, your connector must never bundle it. -->
        <dependency>
            <groupId>ma.s2m.payos</groupId>
            <artifactId>connector-sdk</artifactId>
            <version>1.2.0-RELEASE</version>
            <scope>provided</scope>
        </dependency>

        <!-- Test-only — exempt from the approved-dependency registry (external-dependency-approval,
             §2) because scope=test never ships inside the delivered jar. -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.3</version>
            <scope>test</scope>
        </dependency>

        <!-- Add any third-party library your connector needs at normal (non-provided,
             non-test) scope here — for example a bank/network SDK, an HTTP client, etc.
             It must be on the approved-dependencies registry before you can certify or
             deliver the connector — see §7 below and external-dependency-approval. -->
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

Notes on this `pom.xml`: `maven.compiler.release` fixes the Java 21 requirement from §1 without needing a parent POM to inherit it from. `connector-sdk`'s `provided` scope is not a style choice — [connector-certification-v1-2026-08-29.md](connector-certification-v1-2026-08-29.md) fails a jar with `INVALID_SDK_SCOPE` if this is anything else, and `connector-sdk`'s own transitive dependency `ma.s2m.payos:payos-connector-api` (the module that actually declares `IConnector`) rides along automatically through `provided` scope — you never declare it yourself. Junit/AssertJ versions above match what `connector-sdk` itself is tested against internally, so your test code and the SDK's own compiled classes were validated against the same library versions; pin any versions you prefer instead, since these aren't fixed contractually the way the SDK coordinate is.

## 3. Implement `IConnector`

This is the actual business logic, and it's covered in full in [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md): the five-method contract, the model types (`ConnectorConfig`, `ConnectorExecutionContext`, `ConnectorResponse`, the error-category enum), the idempotency ownership split, and a complete working example both as plain `IConnector` and via the `AbstractConnector` base class. Write your class under `src/main/java` at whatever package you chose in §2 (e.g. `com.example.connectors.cmi.CmiPaymentConnector`) — nothing about its package name is constrained, only `getType()`/`getName()`'s *return values* need to match the descriptor you write next.

## 4. Add the descriptor and SPI registration file

Two plain-text files under `src/main/resources/META-INF/` (Maven copies everything under `src/main/resources` onto the classpath root, so this is all that's needed for both to end up at the right path inside your jar — no extra plugin configuration).

`META-INF/connector.properties` — the descriptor, full field reference in [packaging-and-deployment-v1-2026-07-27.md §1](packaging-and-deployment-v1-2026-07-27.md#1-the-descriptor-meta-infconnectorproperties):

```properties
connector.type=PaymentGateway
connector.name=cmi
connector.api.version=1.0
connector.required.params=merchantId,apiKey,endpointUrl
connector.requires.idempotency=true
```

`META-INF/services/ma.s2m.payos.connector.api.IConnector` — the SPI registration file the JDK's `ServiceLoader` reads, full detail in [packaging-and-deployment-v1-2026-07-27.md §2](packaging-and-deployment-v1-2026-07-27.md#2-spi-registration-meta-infservices). Its content is a single line, the fully-qualified name of your implementation class:

```
com.example.connectors.cmi.CmiPaymentConnector
```

## 5. Build the jar

Plain `mvn clean package` (`maven-jar-plugin`, the Maven default — no extra configuration needed) is enough **as long as your connector bundles nothing beyond `connector-sdk` itself** (which stays out of the jar anyway, since it's `provided`) — the default jar plugin packages your compiled classes and everything under `src/main/resources`, which is exactly the descriptor and SPI file from §4, and nothing else.

If you added a third-party library at normal scope in §2 (an HTTP client, a bank SDK, …), `mvn package` alone will **not** put that library's classes into your jar — the default jar plugin never bundles dependencies. You need a shading/assembly plugin for that; `maven-shade-plugin` is the one used across the rest of this codebase (see [build-and-release/build-conventions.md](../build-and-release/build-conventions.md)), and its default behavior of excluding `provided`-scope dependencies from the shaded jar is exactly what you want here — it bundles your normal-scope third-party libraries while leaving `connector-sdk` out automatically, with no extra `<excludes>` configuration required:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
        </execution>
    </executions>
</plugin>
```

Either way, the resulting jar must contain your compiled classes, `META-INF/connector.properties`, `META-INF/services/ma.s2m.payos.connector.api.IConnector`, and (if applicable) your bundled third-party libraries' classes — and must **not** contain any `ma/s2m/payos/connector/**/*.class` entry, since that would mean `connector-sdk` got bundled in by mistake (§2's `provided` scope should already prevent this; [connector-certification-v1-2026-08-29.md §1](connector-certification-v1-2026-08-29.md#1-what-it-checks) is what actually verifies it, as `INVALID_PACKAGING`).

## 6. Test it without a running PayOS runtime

`ConnectorTestHarness` calls `init`/`execute`/`close` on your implementation directly from a plain JUnit test, no PayOS runtime needed — full detail and a working example in [testing-and-delivery-checklist-v3-2026-08-29.md §1](testing-and-delivery-checklist-v3-2026-08-29.md#1-testing-without-a-full-payos-runtime). Write this under `src/test/java` at the same package as your connector class. At minimum, cover a successful execution, every error category you can realistically produce, and (if you declared `connector.required.params`) the required-params validation path.

## 7. Get any bundled third-party library approved

Skip this step if your connector bundles nothing beyond `connector-sdk` (§5's plain-jar case). Otherwise, every third-party coordinate you added at normal scope in §2 needs to already be on the approved-dependencies registry before you can certify or deliver the connector — check [external-dependency-approval-v5-2026-08-29.md §4](external-dependency-approval-v5-2026-08-29.md#4-the-approved-dependencies-registry) first, and if your exact `groupId:artifactId:version` isn't listed with a scope that covers your connector, open a request per [external-dependency-approval-v5-2026-08-29.md §5](external-dependency-approval-v5-2026-08-29.md#5-request-process) — this can take some back-and-forth with a reviewer, so start it as early as you know which library you need, not right before delivery.

## 8. Certify before delivery

Run `ConnectorCertificationCli` against your built jar and `pom.xml` — it automates most of the manual checklist in step 9, full detail in [connector-certification-v1-2026-08-29.md](connector-certification-v1-2026-08-29.md):

```bash
java -cp connector-sdk-1.2.0-RELEASE.jar ma.s2m.payos.connector.certification.approval.ConnectorCertificationCli \
    --jar   target/cmi-connector-1.0.0.jar \
    --pom   pom.xml \
    --registry path/to/payos-docs/connector-developer/connector-approved-deps-registry.json \
    --isolation-documented \
    [--import <fully.qualified.Type>]...
```

Exit code `0` means it passed; `1` prints the specific findings to fix (unapproved dependency, wrong SDK scope, forbidden import, …) to stdout.

## 9. Work through the pre-delivery checklist and deliver

[testing-and-delivery-checklist-v3-2026-08-29.md §2](testing-and-delivery-checklist-v3-2026-08-29.md#2-checklist-before-delivery) has the full list, most of which step 8 already checked automatically for you; the remaining items (`close()` actually releasing every resource, `connector.requires.idempotency` set correctly, every foreseeable error mapped to an explicit category) are things only you can verify by reading your own code. Once everything on that list is done, hand the built jar and its `pom.xml` to whoever operates the target PayOS deployment — deploying it is their side of the contract (placing the jar and adding an entry to `connectors.json`), documented in [packaging-and-deployment-v1-2026-07-27.md §3](packaging-and-deployment-v1-2026-07-27.md#3-deployment-connectorsjson); you're not expected to have runtime access yourself unless you're also the operator.

## Next

- [README.md](README.md) — the full document index for this folder, and the "not to be confused with SPI connectors" distinction.
- [writing-a-connector-v1-2026-07-27.md](writing-a-connector-v1-2026-07-27.md) — the `IConnector` contract in full (step 3 above).
- [packaging-and-deployment-v1-2026-07-27.md](packaging-and-deployment-v1-2026-07-27.md) — descriptor, SPI registration, `connectors.json`, versioning (steps 4 and 9 above).
- [testing-and-delivery-checklist-v3-2026-08-29.md](testing-and-delivery-checklist-v3-2026-08-29.md) — `ConnectorTestHarness` and the full pre-delivery checklist (steps 6 and 9 above).
- [external-dependency-approval-v5-2026-08-29.md](external-dependency-approval-v5-2026-08-29.md) — getting a bundled library approved (step 7 above).
- [connector-certification-v1-2026-08-29.md](connector-certification-v1-2026-08-29.md) — running `ConnectorCertificationCli` (step 8 above).
- [developer/build-guide.md](../developer/build-guide.md) — the full PayOS module build order, if you ever need more than the four modules in §1 (e.g. to run a real runtime locally for end-to-end testing).
