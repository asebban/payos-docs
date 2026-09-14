# Identification du code concerné par PCI-DSS

Created: 2026-09-12
Last updated: 2026-09-12
Version: v1

Ce document répond à deux questions : quelles parties du code PayOS sont aujourd'hui concernées par une exigence PCI-DSS, et comment identifier ce code de façon fiable et pérenne plutôt qu'au travers d'un audit manuel ponctuel comme celui-ci. Il complète [architecture/security/security-inventory.md](../architecture/security/security-inventory.md), qui documente les mécanismes de sécurité existants côté architecture ; ce document-ci est orienté traçabilité — retrouver le code depuis une exigence PCI-DSS, et inversement.

---

## 1. Inventaire actuel par domaine PCI-DSS

### 1.1 Données porteur (PAN, track1/2, CVV, date d'expiration, nom du porteur)

Aucune classe métier typée (`PAN`, `cardNumber`, `track1`, `cvv`, etc.) n'existe dans le cœur PayOS (`payos`, `payos-foundation`, `payos-connector-api`, `payos-connector-sdk`). Les données porteur ne transitent que sous forme de `Map<String,Object>` opaques (`ConnectorConfig.parameters`, `ConnectorExecutionContext`) à l'intérieur du pipeline connecteur générique. La saisie, le parsing et le stockage effectifs des données porteur sont délégués aux connecteurs métier (`IConnector`), qui sont des plugins externes non présents dans ce workspace. Conséquence : c'est une bonne réduction de scope PCI pour le noyau, mais PayOS core n'offre aucune garantie de type empêchant un connecteur mal écrit de faire fuiter du PAN brut sous un nom de champ non couvert par la dénylist (section 1.3).

### 1.2 Chiffrement / cryptographie (Req 3.5, 3.6, 3.6.1)

| Composant | Classe | Détail |
|---|---|---|
| Chiffrement des bundles applicatifs | `ma.s2m.payos.security.CryptoService` (`payos`) | AES/GCM/NoPadding (format `P8G2`), clés 128/192/256 bits, clé jamais en dur dans le code (chargée via secret provider) |
| Format legacy | `CryptoService` | AES/ECB/PKCS5Padding (format `P8OS`) toujours supporté en lecture seule pour compatibilité descendante — faiblesse cryptographique connue, `WARN` loggé à l'usage |
| Chiffrement au repos (secrets) | `EncryptedFileStore`, `MasterKeyLoader` (`secret-service-filesystem`) | AES/GCM, IV aléatoire 12 octets, clé maître 256 bits obligatoire |
| Chiffrement centralisé (Vault) | `VaultTransitClient`, `VaultCryptoCodec` (`secret-service-vault`) | Transit engine HashiCorp Vault, AES256-GCM96, signatures RSA/EC |
| PKI / certificats | `LocalCertificateAuthority`, `PemCodec` (`secret-service-filesystem`), `VaultPkiClient`, `VaultCsrBuilder` (`secret-service-vault`) | Émission de certificats, CSR |

HSM (Req 3.6 — opérations PIN) est documenté comme type de connecteur prévu (`hsm`, PKCS#11, Thales payShield, Utimaco/Atalla) mais **aucune implémentation n'existe** dans le workspace (pas de `secret-service-hsm` ni de connecteur HSM).

### 1.3 Masquage / rédaction des données sensibles dans les logs

`ma.s2m.payos.connector.util.SensitiveFieldMasker` (`payos-connector-api`) est l'utilitaire canonique de masquage par nom de champ (dénylist normalisée, insensible à la casse et aux séparateurs, couvrant notamment `pan`, `cardnumber`, `cvv`, `cvv2`, `pin`, `pinblock`, `ssn`, `iban`, `token`, `secret`, `password`). Il est appelé par :

- `ma.s2m.payos.events.audit.AuditEvent` (`payos-foundation`) — chaque champ ajouté via `field(key, value)` est vérifié ; une valeur brute sous un nom sensible est remplacée et un `WARN` est loggé
- `ma.s2m.payos.scripting.ConnectorScriptHandle` (`payos`) — masque le payload connecteur avant de l'écrire dans l'audit trail

Limite connue : le masquage se fait par nom de clé, pas par pattern de valeur — un champ sensible portant un nom non prévu par la dénylist (ex. `secretCode`, `encryptedTrack`) n'est pas masqué. Aucun filet de sécurité n'existe au niveau du framework de logging (pas de `logback.xml` / `log4j2.xml` avec scrubbing) : toute la protection repose sur le fait que le code appelant passe bien par `AuditEvent` ou `SensitiveFieldMasker` plutôt que par un `logger.info(...)` brut.

### 1.4 Authentification et contrôle d'accès (Req 7, 8)

| Mécanisme | Classe |
|---|---|
| OIDC/JWT (principal) | `ma.s2m.payos.security.oidc.nimbus.NimbusSecurityService` (`payos`) |
| OIDC pac4j (legacy) | `ma.s2m.payos.security.oidc.pac4j.SecurityService` (`payos`) |
| RBAC par ressource | `ma.s2m.payos.resources.api.ApiResourceHandler` (`payos`) — vérifie `requiredRoles`, 403 si insuffisant |
| Isolation / politique tenant | `ma.s2m.payos.multitenancy.TenantPolicyService` (`payos`) |
| Sessions distribuées | `session-service-redis` (`RedisSessionStore`) |

### 1.5 Audit trail (Req 10)

| Composant | Détail |
|---|---|
| `ma.s2m.payos.security.AuditLogger` (`payos`) | Façade statique, documentée dans le code comme « PCI-DSS Req 10 » |
| `ma.s2m.payos.events.audit.AuditEvent` / `IAuditLogger` (`payos-foundation`) | Modèle d'événement, masquage intégré (section 1.3), politique d'allowlist `BusinessKeysPolicy` (deny-by-default sur les clés métier non approuvées) |
| `payos-buffered-audit-trail` | File bornée en mémoire, écriture asynchrone vers le store durable |
| `payos-audit-trail-store-filesystem` | Stockage append-only partitionné par tenant/jour, **chaînage SHA-256** pour la détection d'altération (Req 10.5.5), déduplication par ID d'événement |

Rétention (Req 10.5 / 10.7, 12 mois recommandés) : documentée dans `security-inventory.md` mais **non appliquée dans le code** — aucune purge ou archivage automatique dans `payos-audit-trail-store-filesystem`.

### 1.6 Réseau / transport (Req 4.1)

| Composant | Détail |
|---|---|
| `ma.s2m.payos.servers.impl.HttpServer` (`payos-server-http`) | `buildSslContext()` construit un `SSLContext` à partir d'un keystore configuré |
| `payos-server-tcp` | **Aucun code TLS** — écart à vérifier, la spec du framework connecteur exige `ssl=true` par défaut pour les connecteurs `iso8583`/`iso20022`/`hsm` |
| `payos-server-queue`, `queue-service-nats` | Pas de code TLS applicatif — dépend de la configuration du client NATS sous-jacent |
| `webhook-service-http` | Pas de TLS applicatif dédié — repose sur le `HttpClient` JDK standard pour les URLs `https://` |

### 1.7 Secrets et gestion de clés (Req 3.5, 3.6, 7)

- `secret-service-filesystem` : stockage chiffré par tenant (`<root>/<tenant>/<name>.enc`), clé maître via fichier ou variable d'environnement `PAYOS_SECRET_MASTER_KEY`, tokenisation (`FileSystemTokenProvider`)
- `secret-service-vault` : HashiCorp Vault (KV v2, Transit, PKI), authentification AppRole, namespacing des clés par tenant (`<tenantId>_<keyName>`)
- Les deux fournisseurs refusent explicitement la création implicite de clé (pas d'auto-création silencieuse)

### 1.8 Idempotency / cache (Req 3.4)

- `RedisIdempotencyStore` (`idempotency-service-redis`) met en cache le corps de réponse HTTP **complet**, non chiffré, sans étape de masquage, sous la clé d'idempotency, avec TTL Redis. Si une réponse de paiement contient un jour du PAN ou une donnée adjacente, elle serait mise en cache telle quelle — point à vérifier en pratique.
- `ICacheStore` / binding de script `$Cache` (`cache-service-redis`, `cache-service-memory`) : cache générique sans connaissance des clés sensibles, contrairement à `AuditEvent`.

### 1.9 Frontend (`vue-app`)

Aucune saisie, affichage ou transmission de données porteur trouvée dans le code du dépôt. Le frontend est une coquille générique qui charge dynamiquement les pages/composants applicatifs depuis le backend au runtime — si une UI de saisie carte existe en production, elle vit dans un bundle applicatif tenant (configuration, pas du code source) hors du périmètre de cette recherche.

---

## 2. Lacunes identifiées

1. Aucune donnée porteur typée dans le noyau — bon pour la réduction de scope, mais aucune garantie de type côté connecteur externe.
2. Format de chiffrement legacy AES/ECB toujours actif dans `CryptoService`.
3. HSM documenté comme requis PCI (opérations PIN) mais non implémenté.
4. Absence de TLS dans `payos-server-tcp` malgré une exigence documentée dans la spec connecteur.
5. `RedisIdempotencyStore` met en cache des réponses complètes non chiffrées sans masquage.
6. Les couches cache génériques (`ICacheStore`, `$Cache`) n'ont aucune connaissance des clés sensibles.
7. Aucun filet de sécurité au niveau du framework de logging (pas de scrubbing centralisé).
8. Rétention des logs d'audit (Req 10.5/10.7) documentée mais non appliquée en code.

---

## 3. Proposition — identifier ce code de façon continue

L'inventaire ci-dessus a été produit manuellement par recherche dans 25+ dépôts ; il sera périmé dès le prochain changement significatif. Trois mécanismes complémentaires, à combiner :

### 3.1 Annotation `@PciDssScope`

Une annotation Java dans `payos-foundation` (dépendance quasi universelle du graphe PayOS), posée sur les classes concernées :

```java
@PciDssScope(requirement = PciDssRequirement.REQ_10_LOG_MONITORING,
             note = "Audit trail tamper-evidence via chaînage SHA-256")
public class FilesystemAuditTrailStore { ... }
```

`PciDssRequirement` doit être une énumération fermée (pas une chaîne libre) pour éviter la dérive de nommage. Rétention `RUNTIME` et `@Documented` pour apparaître dans le Javadoc généré et être détectable par réflexion ou par un outil d'inventaire. À poser en priorité sur les classes listées en section 1 : `CryptoService`, `SensitiveFieldMasker`, `AuditEvent`, `AuditLogger`, `VaultTransitClient`, `EncryptedFileStore`, `RedisIdempotencyStore`.

### 3.2 Script d'inventaire cross-repo

Un script (dans `payos-pm` ou `payosv2-packer`, qui orchestrent déjà l'ensemble des dépôts) qui scanne tous les repos pour :

- les usages de `@PciDssScope`
- les usages de `SensitiveFieldMasker`, `Cipher`, `KeyStore`, `Vault*`
- les motifs `pan|cvv|track[12]|cardNumber` dans le code source
- les champs nommés `secret*`/`key*` non couverts par la dénylist de `SensitiveFieldMasker`

Il génère un CSV versionné (`pci-dss-inventory-vN-YYYY-MM-DD.csv`), exploitable dès maintenant sans attendre que tout le code existant soit annoté, et rejouable en CI pour détecter une régression de couverture.

### 3.3 Document de synthèse régénéré

Fusionner la table « Couverture PCI-DSS » de `security-inventory.md` avec le CSV du point 3.2, pour que la correspondance Req → mécanisme → fichier reste vérifiable plutôt que maintenue à la main.

---

## 4. Prochaines étapes

- [ ] Créer l'annotation `@PciDssScope` et l'énumération `PciDssRequirement` dans `payos-foundation`
- [ ] Annoter les classes de la section 1
- [ ] Écrire le script d'inventaire cross-repo
- [ ] Traiter les lacunes de la section 2, en commençant par la plus critique pour le contexte de production (TLS sur `payos-server-tcp`, ou masquage du cache d'idempotency)
