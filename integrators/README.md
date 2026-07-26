# Integrator guide

Practical guides for partner teams customizing an editor-delivered PayOS application for a specific client, and for delivering the result securely.

| Document | Covers |
| --- | --- |
| [integration-workflow.md](integration-workflow.md) | *(French)* The full integrator workflow — receiving an editor bundle, customizing via `extends`/hooks/capabilities without modifying delivered code, multi-tenancy, secrets, i18n, testing, environment management, and delivering to the client. |
| [tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md](tenant-bundle-encryption-key-lifecycle-v1-2026-07-24.md) | End-to-end lifecycle of a bundle's `encryptionKey`: generating it, deciding custody (filesystem keyfile vs. key-blind Vault delivery), what to encrypt vs. exclude, encrypting with `edc`, delivering to the client, rotation, and the gaps to close before this is fully production-ready. |
| [assembling and encrypting a bundle.md](assembling%20and%20encrypting%20a%20bundle.md) | The wider bundle-assembly process (assembly script, pre-encryption smoke test) that the encryption step above slots into. |
| [decryption mechanism.md](decryption%20mechanism.md) | How the client-side runtime actually decrypts a packed bundle at load time (`CryptoService`) — the read side of the process documented in this folder. |
| [extensibility-mechanisms.md](extensibility-mechanisms.md) | Extension points available to integrators (connectors, capabilities, hooks) without modifying the editor's delivered code. |

## Next

- [Developer guide](../developer/README.md) — for building the application itself, as opposed to customizing/delivering someone else's.
- [Operations: bundle encryption](../operations/bundle-encryption.md) — the `edc` CLI reference.
- [Configuration: secret service](../configuration/secret-service.md) — the `filesystem`/`vault` providers backing `encryptionKey` storage.
