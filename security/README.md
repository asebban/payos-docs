# Security

Cross-cutting security compliance documentation — how PCI-DSS (and future compliance frameworks) requirements map onto PayOS code, as distinct from [architecture/security-architecture.md](../architecture/security-architecture.md) and [architecture/security/security-inventory.md](../architecture/security/security-inventory.md), which document the security mechanisms themselves.

| Document | Purpose |
| --- | --- |
| [PCI-DSS identification](pci-dss-identification-v1-2026-09-12.md) | Inventory of the code concerned by each PCI-DSS requirement, known coverage gaps, and the proposed method (annotation + cross-repo inventory script) to keep that mapping accurate over time. |
| [Secret Providers — capabilities & technical exploitation guide](secret-providers-capabilities-v1-2026-09-15.md) | Full capability matrix and technical usage reference for the `filesystem` and `vault` secret providers — secrets CRUD, versioning, tokenization, symmetric/asymmetric crypto, per-tenant PKI/certificate issuance — with exact API signatures, storage layout, Vault engine mapping, and known security limitations. |
