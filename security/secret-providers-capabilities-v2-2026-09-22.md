# Secret Providers — capacités exposées et guide technique d'exploitation

Created: 2026-09-15
Last updated: 2026-09-22
Version: v2

Ce document répertorie, capacité par capacité, tout ce que les deux Secret Providers livrés avec PayOS (`secret-service-filesystem` et `secret-service-vault`) savent faire, et explique techniquement comment exploiter chaque capacité — signatures exactes, exemples de code Java, structure de stockage, mapping vers les moteurs Vault, et limites de sécurité connues. Il complète [architecture/secret-provider-architecture.md](../architecture/secret-provider-architecture.md) (contrat SPI, intégration kernel, cycle de vie `$Secrets`) sans le dupliquer : ce document-ci répond à « qu'est-ce que je peux faire avec un provider, et comment » plutôt qu'à « comment le kernel le charge ». Voir aussi [§6](#6-écart-avec-la-documentation-existante) pour un écart de documentation identifié pendant la rédaction de ce document.

---

## 1. Périmètre

Les deux providers étendent `AbstractSecretProvider` (`ma.s2m.payos.secret.spi`, module `payos-foundation`) et implémentent, au-delà du contrat noyau `ISecretProvider`, les mêmes quatre interfaces de capacité optionnelles — `ITokenProvider`, `ICryptoSecretProvider`, `IAsymmetricCryptoSecretProvider`, `ICertificateSecretProvider` — plus, pour le seul provider filesystem, `IVersionedSecretProvider`. Aucun des deux n'implémente `IWatchableSecretProvider`.

`FileSystemSecretProvider.capabilities()` et `VaultSecretProvider.capabilities()` déclarent aujourd'hui exactement le même `EnumSet` :

```java
EnumSet.of(
    SecretCapability.GET, SecretCapability.SET, SecretCapability.DELETE,
    SecretCapability.LIST, SecretCapability.DESCRIBE, SecretCapability.VERSION,
    SecretCapability.TOKENIZE, SecretCapability.CRYPTO,
    SecretCapability.ASYMMETRIC_CRYPTO, SecretCapability.CERTIFICATE_AUTHORITY)
```

Ce point mérite d'être noté immédiatement : **`VaultSecretProvider` déclare `VERSION` sans implémenter `IVersionedSecretProvider`** (il n'expose que le `current_version` renvoyé par `describeSecret`, pas `getSecretVersion`/`listVersions`/`restoreVersion`/`destroyVersion`). Tester `capabilities().contains(SecretCapability.VERSION)` ne suffit donc pas à savoir si l'historique de versions est utilisable — voir [§3.2](#32-historique-de-versions-iversionedsecretprovider--filesystem-uniquement) pour le test fiable.

| Interface (capacité) | Filesystem | Vault |
|---|---|---|
| `ISecretProvider` (GET/SET/DELETE/LIST/DESCRIBE) | Oui | Oui |
| `IVersionedSecretProvider` | Oui | **Non** (malgré la capacité `VERSION` déclarée) |
| `ITokenProvider` | Oui | Oui |
| `ICryptoSecretProvider` (clé symétrique nommée) | Oui | Oui |
| `IAsymmetricCryptoSecretProvider` (paire de clés nommée) | Oui | Oui |
| `ICertificateSecretProvider` (autorité de certification) | Oui | Oui |
| `IWatchableSecretProvider` | Non | Non |

Toutes les opérations, sur les deux providers, prennent `tenantId` en premier paramètre et sont validées par `AbstractSecretProvider.validateTenantId` (regex `[a-zA-Z0-9\-]+`, sinon `SecretAccessDeniedException`) avant tout accès au backend — voir la section [§11 Isolation multi-tenant](../architecture/secret-provider-architecture.md#11-isolation-multi-tenant) de l'architecture pour le détail des trois niveaux de défense.

---

## 2. Prérequis communs

### Obtenir une instance et vérifier ses capacités

```java
ISecretProvider provider = PayOSConfig.getSecretProvider();
if (provider == null) {
    throw new IllegalStateException("secret-service.configuration.enabled=false");
}
Set<SecretCapability> caps = provider.capabilities();
```

### Tester une capacité de façon fiable

Pour toute capacité au-delà du socle `ISecretProvider`, préférer `instanceof` à `capabilities()` — le seul cas où les deux divergent aujourd'hui est `VERSION`/`IVersionedSecretProvider` sur Vault (voir [§1](#1-périmètre)), mais s'appuyer systématiquement sur `instanceof` évite de reproduire ce piège si un futur provider déclare une capacité qu'il n'implémente pas complètement :

```java
if (provider instanceof IVersionedSecretProvider versioned) {
    versioned.listVersions(tenantId, name);
}
```

### `SecretValue` — durée de vie obligatoire en try-with-resources

Toute méthode qui retourne une valeur de secret (`getSecret`, `getSecretVersion`, `detokenize` via le binding `$Secrets`) retourne un `byte[]` nu (pour `ITokenProvider.detokenize`) ou un `SecretValue` `AutoCloseable` (pour `ISecretProvider`/`IVersionedSecretProvider`) qui zero ses bytes internes à `close()` :

```java
try (SecretValue value = provider.getSecret(tenantId, "stripe-api-key")) {
    String secret = new String(value.exposeBytes(), StandardCharsets.UTF_8);
} // zeroing automatique ici
```

---

## 3. Capacités et exploitation technique

### 3.1 Secrets bruts (`ISecretProvider` — socle commun)

Stocker, lire, lister, décrire et supprimer une valeur opaque (clé API, mot de passe, token OAuth, etc.) scopée à un tenant.

```java
SecretMetadata meta = new SecretMetadata(tenantId, "stripe-api-key", "api-key", null, null, 0);
provider.setSecret(tenantId, "stripe-api-key", "sk_live_xxxx".getBytes(StandardCharsets.UTF_8), meta);

try (SecretValue v = provider.getSecret(tenantId, "stripe-api-key")) { /* ... */ }

List<String> names = provider.listSecrets(tenantId);
SecretMetadata described = provider.describeSecret(tenantId, "stripe-api-key");
provider.deleteSecret(tenantId, "stripe-api-key");
```

**Filesystem** : chaque secret devient `<root>/<tenantId>/<name>.enc` (enveloppe AES/GCM/NoPadding : IV 12 octets + ciphertext + tag 128 bits) plus `<root>/<tenantId>/<name>.meta.json`. L'écriture passe par `<name>.enc.tmp` puis `Files.move(ATOMIC_MOVE, REPLACE_EXISTING)` — un lecteur concurrent voit toujours soit l'ancienne, soit la nouvelle version, jamais un fichier partiel. Le nom de secret est validé par la regex `[a-zA-Z0-9_.\-]+` (pas de `/`), et le `tenantId` par `[a-zA-Z0-9\-]+` — double protection contre le path traversal, vérifiée à la fois dans `AbstractSecretProvider` et dans `SecretPath`.

**Vault** : chaque secret devient une entrée KV v2 à `<kvMount>/data/<tenantId>/<name>` (écriture/lecture) et `<kvMount>/metadata/<tenantId>/<name>` (describe/delete/list). La valeur est encodée en Base64 sous un champ JSON nommé d'après le secret lui-même (`<name>`), avec un champ `type` associé ; à la lecture, si le contenu n'est pas du Base64 valide, un fallback UTF-8 brut est tenté. `listSecrets` s'appuie sur l'opération Vault `LIST` (premier niveau du namespace uniquement).

#### Hash déterministe (`ISecretProvider.hash`)

Depuis la v2 de ce document, `ISecretProvider` porte une méthode par défaut `hash(String value)` qui renvoie le condensé SHA-256 de `value` en hexadécimal minuscule. Contrairement à toutes les autres méthodes du contrat, elle ne prend pas de `tenantId` et n'est ni scopée à un tenant ni adossée à une clé gérée par le provider : la sortie est purement fonction de l'entrée, identique quel que soit le provider configuré, reproductible en dehors de PayOS avec n'importe quelle implémentation SHA-256 standard. Comme c'est une méthode `default` de l'interface, **filesystem et Vault l'exposent tous les deux sans code spécifique** — aucune entrée `SecretCapability` dédiée n'a été ajoutée, puisqu'il ne s'agit pas d'une capacité qui varie d'un provider à l'autre.

```java
String digest = provider.hash("stripe-api-key");
// "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824" pour value = "hello"
```

**Ce que `hash` n'est pas** : ce n'est pas un HMAC ni un hash keyé — n'importe qui connaissant la valeur d'entrée peut reproduire la sortie sans accès au provider ni à un tenant. Ne pas l'utiliser pour dériver un identifiant qui doit rester impossible à deviner à partir d'une valeur candidate (ex. jeton de session, mot de passe) : pour ça, préférer `ICryptoSecretProvider.sign` (§3.4), qui est keyé par un secret géré par le provider. `hash` convient en revanche pour des cas comme la déduplication déterministe, un identifiant de cache, ou la comparaison de deux valeurs sans les exposer dans un log — dans ce dernier cas, voir aussi `SensitiveFieldMasker` pour le masquage de champs sensibles dans l'audit, qui répond à un besoin différent (masquer, pas identifier).

Exposé côté script via le binding `$Secrets` : `$Secrets.hash(value)` — voir [developer/secrets-usage.md](../developer/secrets-usage.md#hashing).

### 3.2 Historique de versions (`IVersionedSecretProvider` — filesystem uniquement)

```java
if (provider instanceof IVersionedSecretProvider versioned) {
    List<Integer> versions = versioned.listVersions(tenantId, "stripe-api-key");
    try (SecretValue old = versioned.getSecretVersion(tenantId, "stripe-api-key", 2)) { /* ... */ }
    versioned.restoreVersion(tenantId, "stripe-api-key", 2); // crée une NOUVELLE version, n'efface rien
    versioned.destroyVersion(tenantId, "stripe-api-key", 1); // refuse si version == version courante
}
```

**Filesystem** : avant chaque `setSecret`, l'enveloppe courante est archivée dans `<root>/<tenantId>/versions/<name>/<version>.enc`. `restoreVersion` relit l'archive et la réécrit via `setSecret` (nouvelle version, historique jamais réécrit — même sémantique que le « restore » de Vault KV v2 ou AWS Secrets Manager). `destroyVersion` refuse explicitement de détruire la version courante. `deleteSecret` purge aussi tout le sous-dossier `versions/<name>/` — pas d'enveloppes orphelines.

**Vault** : non exploitable via cette interface aujourd'hui. `describeSecret` expose bien `current_version` (issu nativement de KV v2), mais `getSecretVersion`/`listVersions`/`restoreVersion`/`destroyVersion` ne sont pas implémentés — l'historique natif de Vault KV v2 n'est donc, pour l'instant, pas accessible via l'API PayOS (seulement via l'API Vault directe, hors du périmètre du provider).

### 3.3 Tokenisation (`ITokenProvider`)

Remplace une valeur sensible par un token opaque non réversible sans passer par le provider — utile pour réduire le scope PCI d'un champ (ex. stocker un token en base plutôt qu'un PAN).

```java
if (provider instanceof ITokenProvider tokens) {
    String token = tokens.tokenize(tenantId, sensitiveBytes);   // UUID v4
    byte[] restored = tokens.detokenize(tenantId, token);        // lève TokenNotFoundException si absent/révoqué
    boolean exists = tokens.tokenExists(tenantId, token);
    tokens.revokeToken(tenantId, token);
    List<String> allTokens = tokens.listTokens(tenantId);
}
```

**Filesystem** : chaque token devient `<root>/<tenantId>/tokens/<uuid>.enc`, chiffré AES-GCM sous la même clé maîtresse que les secrets.

**Vault** : chaque token est une entrée KV v2 ordinaire sous `<kvMount>/data/<tenantId>/tokens/<uuid>`. Comme `listSecrets` ne liste que le premier niveau du namespace du tenant, `tokens/` n'apparaît jamais dans `listSecrets()` — `listTokens` interroge spécifiquement ce sous-préfixe. **Limite connue côté Vault** : `revokeCertificate` (§3.6) et `revokeToken` sont deux mécanismes indépendants ; rien dans le module ne purge automatiquement un token expiré, la révocation est une suppression explicite de l'entrée KV.

### 3.4 Chiffrement et signature par clé symétrique nommée (`ICryptoSecretProvider`)

Une clé AES-256 nommée, générée et conservée côté provider — l'appelant ne voit jamais la clé, seulement les opérations qu'elle permet.

```java
if (provider instanceof ICryptoSecretProvider crypto) {
    crypto.generateKey(tenantId, "webhook-hmac-key"); // échoue si la clé existe déjà — pas d'écrasement silencieux
    boolean exists = crypto.keyExists(tenantId, "webhook-hmac-key");
    byte[] ciphertext = crypto.encrypt(tenantId, "webhook-hmac-key", plaintext);
    byte[] plaintext2 = crypto.decrypt(tenantId, "webhook-hmac-key", ciphertext);
    byte[] mac = crypto.sign(tenantId, "webhook-hmac-key", data);
    boolean valid = crypto.verify(tenantId, "webhook-hmac-key", data, mac);
}
```

**Filesystem** (`NamedKeyStore`) : clé AES-256 générée par `SecureRandom`, stockée chiffrée (« wrapped ») sous la clé maîtresse à `<root>/<tenantId>/keys/<keyName>.key.enc`. `sign`/`verify` implémentent du HMAC-SHA256 (`Mac.getInstance("HmacSHA256")`) — **pas** une signature asymétrique ; `verify` compare en temps constant via `MessageDigest.isEqual`.

**Vault** (moteur **Transit**) : le nom de clé PayOS est mappé sur `<tenantId>_<keyName>` (namespacing par préfixe, pas par chemin dédié), type de clé `aes256-gcm96`. `sign`/`verify` sont routés vers les endpoints Transit **`hmac`/`verify`** (et non `transit/sign`), cohérent avec le fait que la clé est symétrique. Attention au partage de l'espace de noms Transit : `ICryptoSecretProvider` et `IAsymmetricCryptoSecretProvider` utilisent le **même** namespace `<tenantId>_<keyName>` côté Vault — un nom de clé ne peut pas être réutilisé entre une clé symétrique (§3.4) et une paire de clés (§3.5) pour le même tenant.

### 3.5 Paires de clés asymétriques nommées (`IAsymmetricCryptoSecretProvider`)

Génère et conserve une paire de clés (RSA ou EC) côté provider ; seule la clé publique est exposée à l'appelant.

```java
if (provider instanceof IAsymmetricCryptoSecretProvider asym) {
    byte[] publicKeyPem = asym.generateKeyPair(tenantId, "device-signing-key", KeyAlgorithm.RSA_2048);
    byte[] pub = asym.getPublicKey(tenantId, "device-signing-key");        // X.509 SubjectPublicKeyInfo, PEM
    boolean exists = asym.keyPairExists(tenantId, "device-signing-key");
    byte[] sig = asym.signWithKeyPair(tenantId, "device-signing-key", data);
    boolean ok = asym.verifyWithKeyPair(tenantId, "device-signing-key", data, sig);
    boolean okExternal = asym.verifyWithPublicKey(externalPublicKeyPem, data, sig); // clé publique fournie par l'appelant
}
```

`KeyAlgorithm` : `RSA_2048`, `RSA_3072`, `RSA_4096`, `EC_P256`. Les méthodes sont nommées `signWithKeyPair`/`verifyWithKeyPair` (et non `sign`/`verify`) précisément pour éviter une collision de signature Java si un provider implémentait un jour les deux interfaces avec des méthodes de même nom mais de sémantique différente.

**Filesystem** (`AsymmetricKeyStore`, Bouncy Castle) : clé privée encodée PKCS#8, chiffrée sous la clé maîtresse à `<root>/<tenantId>/keypairs/<keyName>.priv.enc` ; clé publique en PEM clair (non secrète) à `<root>/<tenantId>/keypairs/<keyName>.pub.pem`. Signature via `java.security.Signature` (`SHA256withRSA` ou `SHA256withECDSA`).

**Vault** (moteur **Transit**) : mêmes namespace `<tenantId>_<keyName>` et endpoints `transit/sign`/`transit/verify` que pour les clés symétriques (types `rsa-2048`/`rsa-3072`/`rsa-4096`/`ecdsa-p256`). Vault Transit retourne/attend des signatures au format auto-descriptif `vault:v1:<base64>` — `VaultCryptoCodec` déshabille/rhabille ce format à la frontière du provider pour que `signWithKeyPair`/`verifyWithKeyPair` manipulent des bytes PKCS#1 (RSA) ou DER `(r,s)` (EC) **portables**, identiques au format produit par l'implémentation filesystem. **Limite connue** : le préfixe est toujours reconstruit en `v1` côté Vault — pas de support de rotation de clé (une clé Transit alternée en `v2` chez Vault ne serait pas gérée correctement par ce codec aujourd'hui).

### 3.6 Autorité de certification par tenant (`ICertificateSecretProvider`)

Émission de certificats X.509 depuis une CSR externe, ou pour une paire de clés déjà détenue par le provider (§3.5) — dans les deux cas, la clé privée du sujet ne transite jamais par le provider :

```java
if (provider instanceof ICertificateSecretProvider ca) {
    X509Certificate caCert = ca.getCACertificate(tenantId);
    X509Certificate cert1 = ca.issueCertificateFromCsr(tenantId, "CN=device-42", csrPemBytes);
    X509Certificate cert2 = ca.issueCertificateForKey(tenantId, "CN=svc-internal", "device-signing-key");
    X509Certificate fetched = ca.getCertificate(tenantId, "device-42");
    List<String> names = ca.listCertificates(tenantId);
    ca.revokeCertificate(tenantId, cert1.getSerialNumber().toString(16));
}
```

**Filesystem** (`LocalCertificateAuthority`, Bouncy Castle) : **une CA auto-signée par tenant**, provisionnée paresseusement (au premier appel, `synchronized`) — RSA-4096, validité 10 ans, `BasicConstraints(true)`, usages `keyCertSign|cRLSign`, stockée à `<root>/<tenantId>/ca/ca.key.enc` (chiffrée) et `<root>/<tenantId>/ca/ca.cert.pem`. Les certificats émis ont une validité de 1 an, `BasicConstraints(false)`, usages `digitalSignature|keyEncipherment|keyAgreement`, identifiants de clé sujet/autorité renseignés, stockés à `<root>/<tenantId>/certs/<name>.cert.pem`. `issueCertificateFromCsr` vérifie d'abord la signature de la CSR elle-même (preuve de possession de la clé privée) avant d'émettre. `revokeCertificate` ajoute le numéro de série à un fichier `<root>/<tenantId>/ca/revoked.txt` (liste dédupliquée, append-only).

**Vault** (moteur **PKI**) : **une seule CA, partagée par tous les mounts/tenants** — Vault PKI n'a pas de notion de CA par tenant. L'isolation se fait au niveau d'un **rôle** Vault PKI nommé `<tenantId>_<pki-role>`, provisionné côté opérateur (pas par le provider lui-même) — c'est l'opérateur qui définit, par rôle, les contraintes (domaines autorisés, TTL max, usages). Appels : `GET pki/ca/pem`, `POST pki/sign/<role>`, `GET pki/cert/<serial>`, `POST pki/revoke`. Comme Vault PKI n'indexe que par numéro de série (pas par nom), un index nom→série est maintenu séparément comme entrées KV v2 ordinaires à `<kvMount>/data/<tenantId>/certs/<name>`. Pour `issueCertificateForKey`, un `ContentSigner` Bouncy Castle personnalisé (`VaultTransitContentSigner`) construit la CSR en appelant Transit `/sign/<name>` à la volée — **la clé privée ne quitte jamais Vault, y compris pendant la construction de la CSR**.

**Limites de sécurité importantes, sur les deux providers** :
- La révocation est un registre administratif append-only (`revoked.txt` côté filesystem, appel `pki/revoke` côté Vault) — **rien dans `ICertificateSecretProvider` ne consulte cette liste** au moment de `getCertificate`/vérification de CSR. Une intégration CRL/OCSP applicative reste à construire par l'appelant si ce contrôle est requis.
- Côté Vault spécifiquement, `revokeCertificate` ne nettoie pas l'index nom→série en KV v2 : un certificat révoqué peut continuer à apparaître dans `listCertificates`.
- Côté Vault, `getCACertificate` renvoie la **même** CA pour tous les tenants — un opérateur qui veut une isolation cryptographique de la racine de confiance entre tenants doit déployer des mounts PKI Vault distincts (hors périmètre du provider actuel), pas seulement des rôles distincts.

### 3.7 Notifications de changement (`IWatchableSecretProvider`) — non implémenté

```java
public interface IWatchableSecretProvider {
    void watch(String tenantId, String name, SecretChangeListener listener);
    void unwatch(String tenantId, String name);
}
```

Ni `FileSystemSecretProvider` ni `VaultSecretProvider` n'implémentent cette interface aujourd'hui. Un appelant qui a besoin de réagir à une rotation de secret (ex. recharger une clé API après rotation) doit aujourd'hui interroger `describeSecret` (champ `version`/`modified`) en polling, ou s'appuyer sur le hot reload de configuration du kernel (qui recrée le provider, pas un secret individuel).

---

## 4. Sélectionner et configurer un provider

Sélection via le bloc `secret-service.configuration` (`bootstrap.json`), résolu par `SecretProviders` via `ServiceLoader<ISecretProviderFactory>` — voir [architecture/secret-provider-architecture.md §8](../architecture/secret-provider-architecture.md#8-intégration-kernel) pour le détail du chargement SPI. Référence exhaustive des clés : [configuration/secret-service.md](../configuration/secret-service.md).

```json
"secret-service": {
  "configuration": {
    "enabled": true,
    "type": "filesystem",
    "root": "/opt/payos/secrets",
    "keyfile": "/opt/payos/secrets/.keyfile"
  }
}
```

| Clé (`type: "filesystem"`) | Obligatoire | Défaut | Rôle |
|---|---|---|---|
| `root` | Non | `secrets` | Racine du stockage chiffré |
| `keyfile` | Non* | — | Chemin vers la clé maîtresse AES-256 (32 octets bruts). *Sans `keyfile` lisible, retombe sur la variable d'environnement `PAYOS_SECRET_MASTER_KEY` (Base64, 32 octets décodés) ; si aucune des deux n'est exploitable, le provider refuse de démarrer (`SecretProviderException`) |

```json
"secret-service": {
  "configuration": {
    "enabled": true,
    "type": "vault",
    "address": "https://vault.internal:8200",
    "role-id": "...", "secret-id": "...",
    "kv-mount": "secret", "transit-mount": "transit", "pki-mount": "pki", "pki-role": "payos-tenant"
  }
}
```

| Clé (`type: "vault"`) | Obligatoire | Défaut | Rôle |
|---|---|---|---|
| `address` | Oui | — | URL Vault |
| `token` | Conditionnel | — | Auth par token statique |
| `role-id` / `secret-id` | Conditionnel | — | Auth AppRole — **prioritaire** sur `token` si les deux sont présents |
| `approle-mount` | Non | `approle` | Mount de login AppRole |
| `kv-mount` | Non | `secret` | Mount KV v2 (secrets, tokens) |
| `transit-mount` | Non | `transit` | Mount Transit (crypto symétrique/asymétrique) |
| `pki-mount` | Non | `pki` | Mount PKI (certificats) |
| `pki-role` | Conditionnel | — | Rôle PKI, requis uniquement pour l'émission de certificats |
| `namespace` | Non | — | Namespace Vault Enterprise (distinct du `tenantId` PayOS) |
| `tls-skip-verify` | Non | `false` | **Dev uniquement** — désactive la validation TLS, `WARN` loggé si `true` |
| `timeout` | Non | `10` | Timeout HTTP (secondes) |

`VaultConfig.from` lève `IllegalArgumentException` si `address` est vide, ou si ni `token` ni la paire `role-id`/`secret-id` n'est fournie.

L'échec d'initialisation d'un provider configuré avec `enabled=true` est **fatal** au démarrage du runtime (`SecretServiceInitializer`) — pas de démarrage silencieux avec `$Secrets` indisponible.

---

## 5. Considérations de sécurité transverses

| Sujet | Filesystem | Vault |
|---|---|---|
| Authentification/transport | Système de fichiers local — sécurité déléguée aux permissions OS | AppRole (recommandé) ou token statique ; ré-authentification automatique sur `403` (`VaultAuth.onTokenRejected()`, un seul retry) |
| TLS | N/A | Trust store JVM par défaut (importer la CA Vault si autosignée) ; `tls-skip-verify=true` installe un `X509TrustManager` permissif — jamais en production |
| Permissions fichiers | **Non appliqué par le code** — aucun `chmod`/`PosixFilePermissions` dans le module ; `chmod 700` sur `root` est une recommandation opérationnelle, pas une garantie du provider | N/A |
| Clé maîtresse / clé racine | Chargée une fois au démarrage, copie défensive en mémoire JVM, zeroed par `close()` — pas de protection contre un heap dump au crash | Jamais détenue par PayOS — reste dans Vault |
| Tampering | AES-GCM authentifié — toute modification d'un `.enc` est détectée au déchiffrement (`"Decryption failed (tampered data or wrong key)"`) | Intégrité déléguée à Vault |
| Cross-tenant | Triple défense : `tenantId` immuable capturé à l'injection du binding, validé par regex dans `AbstractSecretProvider`, isolation physique par répertoire dans `SecretPath` | Même validation regex ; isolation par préfixe de chemin KV/Transit et par rôle PKI — pas d'isolation physique de la CA (§3.6) |
| Audit | Chaque appel journalisé sur le logger SLF4J `payos.secret.audit` (succès/refus/absence/écriture/suppression/liste) — **la valeur du secret n'est jamais incluse**, seuls `tenantId`, `secretName`, `callerId`, `correlationId`, `result` le sont | Identique — le mécanisme d'audit est dans `AbstractSecretProvider`, commun aux deux providers |

Le masquage par nom de champ dans les logs applicatifs (`ma.s2m.payos.connector.util.SensitiveFieldMasker`) est un mécanisme **distinct** de l'audit des secret providers : il protège les champs métier nommés `pan`, `cvv`, `token`, `password`, etc. dans `AuditEvent`/`ConnectorScriptHandle`, pas les accès aux secret providers eux-mêmes (qui ne loguent déjà aucune valeur). Ne pas réimplémenter un masquage ad hoc pour les secrets — il n'est pas nécessaire ici, l'audit `payos.secret.audit` ne transporte que des métadonnées.

---

## 6. Écart avec la documentation existante

`architecture/secret-provider-architecture.md` (dernière mise à jour indiquée : 2026-07-03) contient, à la date de rédaction de ce document, des affirmations obsolètes vérifiées par lecture directe du code source :

- Il indique que `IAsymmetricCryptoSecretProvider` et `ICertificateSecretProvider` ne sont implémentées par aucun provider filesystem livré (« prévu en phase 3 ») — **faux aujourd'hui** : `FileSystemSecretProvider` implémente les deux (`AsymmetricKeyStore`, `LocalCertificateAuthority`), tout comme `VaultSecretProvider`.
- Sa section 15 (structure des modules) liste `ICryptoSecretProvider`, `IWatchableSecretProvider` et `ICertificateSecretProvider` comme « aucun provider livré » — obsolète pour les deux premières.
- Son tableau `SecretCapability` (§4) omet `CRYPTO`, `ASYMMETRIC_CRYPTO`, `CERTIFICATE_AUTHORITY` — l'énumération réelle les inclut, comme confirmé en [§1](#1-périmètre) de ce document.
- Le module `payos-secret-api` qu'elle référence comme dépendance Maven autonome a été consolidé dans `payos-foundation` (le package `ma.s2m.payos.secret.api` n'a pas changé, mais l'ancien module `payos-secret-api` ne contient plus de code source).

Ce document-ci reflète l'état du code au 2026-09-15 ; en cas de divergence future entre les deux, se fier au code source (`FileSystemSecretProvider.capabilities()`, `VaultSecretProvider.capabilities()`) plutôt qu'à l'un ou l'autre document.

---

## 7. Documents liés

- [architecture/secret-provider-architecture.md](../architecture/secret-provider-architecture.md) — contrat SPI, intégration kernel, cycle de vie `$Secrets`, isolation multi-tenant (à jour sur ces aspects, obsolète sur les capacités — voir [§6](#6-écart-avec-la-documentation-existante))
- [configuration/secret-service.md](../configuration/secret-service.md) — référence exhaustive des clés de configuration
- [developer/secrets-usage.md](../developer/secrets-usage.md) — usage du binding `$Secrets` dans les scripts
- [operations/secrets-management.md](../operations/secrets-management.md) — provisionnement et rotation en production
- [cli-tools/spm.md](../cli-tools/spm.md) — CLI `spm` (module filesystem)
- [developer/vault-secret-id-secure-injection.md](../developer/vault-secret-id-secure-injection.md) — injection sécurisée de `secret-id` via Docker secrets
- [security/pci-dss-identification-v1-2026-09-12.md](pci-dss-identification-v1-2026-09-12.md) — mapping PCI-DSS, section 1.7 « Secrets et gestion de clés »
