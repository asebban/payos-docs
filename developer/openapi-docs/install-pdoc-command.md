# pdoc

`pdoc` is the standalone PayOS OpenAPI documentation CLI. It generates OpenAPI artifacts from PayOS application, capability, and product bundles without starting PayOS Runtime or executing JavaScript.

This repository contains:

- Java 21 Maven project
- package root `ma.s2m.payos.pdoc`
- runnable Java artifact entrypoint
- static target resolution for app, capability, and product targets
- OpenAPI annotation extraction, validation, assembly, enrichment, and deterministic output writing
- JUnit 5 and AssertJ test setup
- no dependency on PayOS Runtime startup classes

## Build

```bash
mvn test
mvn package
java -jar target/pdoc-1.0.0-SNAPSHOT.jar --app merchant-portal --bundle-path ./bundle
java -jar target/pdoc-1.0.0-SNAPSHOT.jar --capability payment-links --bundle-path ./bundle --output ./target/openapi
java -jar target/pdoc-1.0.0-SNAPSHOT.jar --product acquiring
java -jar target/pdoc-1.0.0-SNAPSHOT.jar --help
java -jar target/pdoc-1.0.0-SNAPSHOT.jar --version
```

The command parses target, bundle, and output destination options. Application targets are resolved from the selected bundle's `payos.json`, and application API JavaScript resources are discovered from the resolved app's `api/**/*.js` files in deterministic order. Capability targets are resolved from `category: capability` bundle metadata or installed `.capabilities/<capability-id>` directories, and capability API JavaScript resources are discovered statically from the resolved capability's `api/**/*.js` files while lifecycle hooks are ignored. Product targets are resolved from installed bundle metadata by selecting application and capability entries whose `_contributors` include the requested product id; `extends` declarations are not traversed for product target resolution. The resolved resources are scanned for `@payos.openapi` blocks, parsed into OpenAPI fragments, validated, assembled into an OpenAPI 3.1 document, enriched with PayOS conventions, compatibility-checked, and written as YAML by default or JSON when the output filename ends in `.json`. When `--output` is omitted, `pdoc` computes `target/openapi/applications/<app-id>/openapi.yaml`, `target/openapi/capabilities/<capability-id>/openapi.yaml`, or `target/openapi/products/<product-id>/openapi.yaml`.

## Documentation

- [PayOS OpenAPI annotation format](docs/annotation-format.md)
- [Runtime safety and separation rules](docs/runtime-safety.md)
- [Generation ownership](docs/generation-ownership.md)
- [Package manager integration contract](docs/package-manager-integration.md)
- [Future runtime docs serving and IDE assistance](docs/future-runtime-and-ide.md)
- [Regulated delivery and auditability](docs/regulated-delivery-auditability.md)
- [Troubleshooting and validation examples](docs/troubleshooting.md)
- [pdoc sequence diagrams](docs/sequence-diagrams.md)
- [Target resolution class diagram](docs/target-resolution-class-diagram.md)

During `mvn package`, Maven writes the project version into `pdoc-version.properties` inside the runnable jar. Installed launchers use that jar, so `pdoc --version` works outside the source checkout.

## Launchers

After `mvn package`, the scripts in `scripts/` can invoke the packaged CLI with forwarded arguments:

```bash
./scripts/pdoc --help
```

```powershell
.\scripts\pdoc.ps1 --help
.\scripts\pdoc.cmd --help
```

Installers copy the built jar and launchers into a `bin`/`lib` layout. The default install root is `$HOME/.payos/tools/pdoc`, or pass a custom install root as the first argument. The Bash installer adds the `bin` directory to `$HOME/.profile`; override the profile with `PDOC_PROFILE_PATH` or set `PDOC_SKIP_PATH_UPDATE=true` to skip PATH updates. Source the profile or open a new shell before running `pdoc` by command name. The PowerShell installer also adds the `bin` directory to the current session PATH and to the Windows user PATH.

```bash
./scripts/install-pdoc-tools.sh ~/.payos/tools/pdoc
```

```powershell
.\scripts\install-pdoc-tools.ps1 -InstallRoot "$HOME\.payos\tools\pdoc"
```

After Bash installation, run `source ~/.profile` or open a new shell to pick up the persisted PATH. After PowerShell installation, `pdoc` is available in the current session.
