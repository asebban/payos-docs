# PayOS — Inventaire des aspects de sécurité implémentés

## 1. Authentification

| Mécanisme | Classe | Détail |
|---|---|---|
| **OIDC – Nimbus** (principal) | `NimbusSecurityService` | Code flow avec state + nonce (anti-CSRF/anti-replay), JWK fetching, refresh silencieux, validation JWT |
| **OIDC – Pac4j** (legacy) | `SecurityService` | Fallback pac4j pour compatibilité |

Ces mécanismes permettent gérer un IAM compatible OIDC (Open ID Connect) comme keycloak par exemple. Ils permettent d'implémenter les mécanismes d'authentification et d'autorisation pour les accès à PayOS.

---

## 2. Autorisation — RBAC

- Vérification de `requiredRoles` sur chaque ressource protégée — retourne **403 Forbidden** si rôle insuffisant
- Logique : au moins un rôle de la liste suffit (permissive matching)
- Extraction des rôles depuis les claims JWT du principal OIDC
- `$Principal` injecté dans les scripts avec `id`, `roles`, et l'ensemble des claims
- Toutes les autorisations refusées sont tracées dans l'audit

## 3. Cryptographie (PCI-DSS Req 3.5)

Un service de cryptographie permet de crypter / décrypter les applications PayOS livrées au client afin de protéger la propriété intellectuelle de S2M.

### Caractéristiques

- Tailles de clé supportées : 128, 192, 256-bit (256 recommandé)
- Clés chargées depuis le secret provider (secret nommé `encryptionKey`) — **jamais embarquées dans le code source**
- Déchiffrement transparent de tous les fichiers chiffrés (si la licence est valide)
- Tout échec de déchiffrement est tracé dans l'audit (détection de falsification)

---

## 4. Multi-tenancy et isolation tenant

### Validation et propagation

- Validation du header `X-Tenant-Id` contre la liste des tenants configurés
- Auto-génération de `X-Correlation-Id` (UUID v4) si absent à l'ingress
- Les deux headers sont propagés inchangés et renvoyés dans la réponse

### Rate limiting par tenant

- Quota de requêtes par minute (RPM) configurable par tenant
- Suivi par fenêtre glissante
- Retourne **429 Too Many Requests** en cas de dépassement (jusqu'à décision de ce qu'il faudra faire en cas de dépassement du quota)

### Tenant Simulator

- Mode dev/test privilégié permettant d'usurper un tenant
- Chaque activation est tracée dans l'audit (PCI-DSS 10.2.1.5)

### Classes clés

```
ma.s2m.payos.multitenancy  → TenantPolicyService, TenantScope
```

---

## 5. Audit (PCI-DSS Req 10)

### Architecture

IAuditLogger  (interface) : Slf4jAuditLogger  (implémentation par défaut → JSON sur SLF4J) [Swappable via AuditLogger.setInstance() → NATS, BDD, SIEM...]

### Événements tracés

| Catégorie | Événements |
|---|---|
| Authentification | `logAuthSuccess()`, `logAuthFailure()` |
| Autorisation | `logAuthorizationGranted()`, `logAuthorizationDenied()` |
| Sessions | `logSessionCreated()`, `logSessionDestroyed()` |
| Opérations privilégiées | `logTenantSimulatorActivated()` |
| Cryptographie | `logDecryptionFailure()` |
| Système | `logStartup()`, `logShutdown()`, `logApiExecution()` |

### Format des événements

JSON mono-ligne (ndjson) avec les champs standardisés :
`timestamp` · `event` · `userId` · `tenantId` · `appId` · `correlationId` · `path` · `result`

### Recommandations de rétention (PCI-DSS 10.5 / 10.7)

- Appender dédié en mode append-only protégé

### Classes clés

```
ma.s2m.payos.security  → IAuditLogger, AuditLogger, Slf4jAuditLogger, AuditEvent
```

---

## 6. Traçabilité des requêtes (Correlation & Tracing)

| Header | Source | Comportement |
|---|---|---|
| `X-Correlation-Id` | Requête entrante ou UUID généré | Propagé inchangé · MDC `correlationId` · renvoyé en réponse |
| `X-Tenant-Id` | Requête ou simulateur | Validé · MDC `tenantId` · renvoyé en réponse |

`TenantScope` est un `AutoCloseable` qui lie les identifiants au MDC SLF4J pour la durée de la requête, puis restaure l'état précédent à la fermeture — garantissant l'isolation entre requêtes concurrentes.

---

## 7. Sandbox GraalVM — Sécurité des scripts (PCI-DSS Req 6/7)

Les scripts JS déposés par les clients s'exécutent dans un contexte GraalVM Polyglot durci :

```java
Context.newBuilder("js")
    .allowAllAccess(false)                        // deny-all par défaut
    .allowHostAccess(HostAccess.ALL)              // proxies Java explicites uniquement
    .allowHostClassLookup(className -> false)     // bloque java.lang.System et toutes classes JVM
    .allowIO(IOAccess.NONE)                       // aucun accès filesystem
    .allowCreateThread(false)                     // aucun threading
    .allowNativeAccess(false)                     // aucun code natif
    .allowCreateProcess(false)                    // aucune création de processus
    .build();
```

### Bindings injectés (seule surface accessible aux scripts)

| Binding | Rôle |
|---|---|
| `$Request` | Requête entrante |
| `$Response` | Construction de la réponse |
| `$Api` | Appels API interne |
| `$App` | Paramètres de l'application |
| `$Principal` | Principal authentifié |
| `$Tenant` | Identifiant tenant |
| `$DB` | Service base de données (optionnel) |
| `$Queue` | Client MoM (optionnel) |
| `$WebHooks` | Dispatcher webhooks |
| `$Logger` | Logger SLF4J |
| `$Library` | Utilitaires |
| `$I18n` | Internationalisation |

Les scripts n'ont **aucun** accès aux classes internes, au filesystem, au réseau direct, ni à l'API JVM.

### Classes clés

```
ma.s2m.payos.scripting.graalvm  → PolyglotScriptingEngine
ma.s2m.payos.resources.api      → ApiResourceHandler (injection des bindings)
```

---

## 8. Idempotency

### Idempotency (`IdempotencyService`)

- Clé d'idempotency extraite du header de requête (nom configurable) et obligatoire quand le service est activé
- Requête bloquée en `400 Bad Request` si la clé est absente ou vide
- Cache de réponses avec TTL configurable (en secondes)
- Sur replay : retourne la réponse en cache avec `X-Idempotency-Replayed: true`
- Activable / désactivable par configuration

---

## Synthèse — Couverture PCI-DSS

| Exigence PCI-DSS | Couverture PayOS |
|---|---|
| **Req 3.5** — Protection des données stockées | `CryptoService` AES/GCM |
| **Req 6** — Sécurité des applications | Sandbox GraalVM, validation inputs |
| **Req 7** — Restriction d'accès | RBAC, `requiredRoles`, tenant isolation |
| **Req 8** — Identification et authentification | OIDC Nimbus, sessions, cookies sécurisés |
| **Req 10** — Journalisation et surveillance | `AuditLogger`, tous événements critiques tracés |
| **Req 10.2.1.5** — Actions administrateurs | `logTenantSimulatorActivated()` |
| **Req 10.5 / 10.7** — Rétention des logs | Recommandations documentées (12 mois) |
