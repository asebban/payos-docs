# PayOS - Features techniques notables

**Date :** 2026-05-13  
**Audience :** direction produit, architectes, tech leads, delivery  
**Objectif :** donner une vue synthétique des capacités techniques les plus différenciantes de PayOS, sans descendre dans le détail d'implémentation.

---

## 1. Synthèse exécutive

PayOS est un runtime de plateforme conçu pour construire, assembler et opérer des produits de paiement dans des environnements multi-clients, régulés et fortement intégrés.

Ses features techniques les plus notables sont :

| Feature | Pourquoi c'est notable |
|---|---|
| Runtime multi-transport | Une même logique métier peut être exposée via HTTP, TCP ou Queue sans dupliquer le coeur de traitement. |
| Micro-kernel applicatif | Le core reste stable pendant que les applications, capabilities et intégrations évoluent autour de lui. |
| Extensions JavaScript via GraalVM | Les comportements métier peuvent être adaptés par scripts contrôlés, sans recompiler le runtime. |
| Capabilities activables | Des modules fonctionnels peuvent être installés, activés, désactivés ou retirés sans redéploiement lourd. |
| Hot reload runtime | Les changements de configuration, d'applications et de capabilities peuvent être pris en compte sans cycle complet rebuild/redéploiement. |
| Multi-tenancy native | Le tenant est une dimension structurante du runtime : routage, sécurité, quotas, logs et réponses. |
| Surface gouvernée | APIs, événements, hooks et extension points sont exposés de façon contrôlée, versionnable et auditable. |
| Sécurité OIDC intégrée | PayOS gère login, callback, logout, session et `/me` via un service de sécurité configurable. |
| Idempotency service | Quand le service est activé, les appels API doivent fournir `X-Idempotency-Key`; PayOS bloque les clés absentes ou vides (sauf si `failOnAbsenceOfIdempotencyKey` est désactivé). |
| Audit log structuré | Les événements sécurité, session, autorisation, API et système sont journalisés en JSON avec tenant et corrélation. |
| Gestion des erreurs métier (`$Errors`) | Les scripts peuvent lever des erreurs normalisées (code, message, statut HTTP) converties automatiquement en JSON structuré. |
| Secret service (`$Secrets`) | Les secrets sont gérés via un provider pluggable (SPI) et injectés dans les scripts sous `$Secrets`. |
| Couverture PCI DSS partielle | Plusieurs mécanismes contribuent aux exigences PCI DSS, notamment audit, contrôle d'accès, traçabilité, masquage et isolation. |
| Hooks et webhooks | Chaque appel peut être enrichi, contrôlé et notifié via des hooks synchrones et des webhooks asynchrones. |
| Configuration externalisée | Le comportement du runtime est piloté par `payos.json`, le `configDir`, les variables d'environnement et les bundles. |
| Déploiement flexible | Le même modèle peut être packagé en runtime monolithique ou assemblé avec des modules/services séparés selon le contexte. |
| Connecteurs remplaçables | Transports, queue, database service, webhooks et plugins TCP peuvent évoluer sans impact sur le kernel. |
| Connector framework | Les providers Java SPI, interfaces de connecteurs et plugins runtime permettent d'ajouter des intégrations sans forker le core. |
| Outillage produit | `ppm`, `apm` et `cpm` permettent de gérer produits, applications et capabilities dans un bundle PayOS. |

---

## 2. Architecture runtime multi-transport

PayOS repose sur un modèle où les transports d'entrée sont séparés du traitement métier.

Les transports actuellement supportés sont notamment :

- HTTP / HTTPS pour les APIs, pages, composants, menus et endpoints de sécurité.
- TCP pour les intégrations protocole et les besoins de message custom.
- Queue / MoM pour les traitements asynchrones et les échanges par messages.

Tous ces transports convergent vers un modèle commun `Request` / `Response`, puis vers `Server.processRequest`. Cette approche permet de conserver une seule chaîne de traitement applicative, même lorsque les canaux d'exposition sont différents. Cela permet un ajout transparent et indépendant d'autres types de transports dans le futur.

**Intérêt technique :** le protocole devient un adaptateur. La logique métier, les règles de sécurité, le routage applicatif et la multi-tenancy restent centralisés.

**Intérêt business :** un même produit peut être intégré avec plusieurs types de clients ou partenaires sans multiplier les implémentations spécifiques.

---

## 3. Micro-kernel et applications externalisées

PayOS sépare le runtime de base des applications qu'il exécute.

Le kernel porte les mécanismes communs :

- chargement de configuration ;
- routage vers les applications ;
- modèle d'échange `Request` / `Response` ;
- sécurité ;
- multi-tenancy ;
- scripting ;
- ressources applicatives ;
- hooks, webhooks, i18n et services transverses.

Les applications sont déclarées dans la configuration et disposent de leur propre répertoire de ressources. Cela donne un modèle où le runtime agit comme un hôte stable et les applications comme des unités configurables et remplaçables.

**Intérêt technique :** le core est moins exposé aux changements fonctionnels. Les évolutions produit peuvent être portées par des applications ou des capabilities.

**Intérêt business :** PayOS peut supporter plusieurs offres, clients ou variantes sans créer une branche logicielle par client.

---

## 4. Ressources applicatives dynamiques

PayOS ne sert pas uniquement des APIs. Le runtime sait résoudre plusieurs types de ressources :

| Type de ressource | Rôle |
|---|---|
| `api` | Exécuter des endpoints JavaScript côté serveur. |
| `page` / `vue` | Assembler et servir des pages Vue. |
| `component` | Exposer des composants réutilisables. |
| `menu` / `_menu` | Composer dynamiquement les menus selon l'application, les extensions et le tenant. |
| `lib` | Charger des bibliothèques JavaScript partagées. |
| `i18n` | Fournir des traductions serveur avec héritage entre applications et capabilities. |
| `file` | Servir des ressources statiques contrôlées par le runtime. |

Cette diversité de ressources permet de construire une application complète avec des endpoints, des pages, des composants, des menus, des textes localisés et des scripts partagés.

**Point notable :** les ressources peuvent être enrichies par héritage via `extends`, ce qui permet de réutiliser ou surcharger des comportements sans recopier toute l'application.

---

## 5. Scripting contrôlé avec GraalVM JavaScript

PayOS utilise GraalVM Polyglot pour exécuter de la logique JavaScript côté serveur.

Les scripts n'accèdent pas directement au runtime interne. Ils utilisent des bindings injectés et contrôlés, par exemple :

| Binding | Usage |
|---|---|
| `$Response` | Construire la réponse applicative. |
| `$Request` | Lire les paramètres, headers et body de la requête entrante. |
| `$Api` | Appeler une autre ressource applicative. |
| `$App` | Accéder au contexte et aux paramètres de l'application. |
| `$Principal` | Consulter l'utilisateur authentifié. |
| `$Tenant` | Accéder au tenant courant. |
| `$Logger` | Journaliser proprement côté runtime. |
| `$Library` | Charger des bibliothèques JS partagées. |
| `$I18n` | Résoudre les traductions serveur. |
| `$Errors` | Lever des erreurs métier normalisées (code, message, statut HTTP). |
| `$DB` | Utiliser le service de données lorsqu'il est configuré. |
| `$Queue` | Publier ou consommer des messages via l'abstraction de queue. |
| `$Secrets` | Lire des secrets sécurisés (optionnel, si `secret-service` configuré). |
| `$WebHooks` | Émettre des événements sortants. |

**Intérêt technique :** PayOS offre un modèle d'extension flexible tout en gardant une frontière claire avec les classes internes Java.

**Intérêt business :** les adaptations spécifiques peuvent être livrées plus vite, sous forme de scripts et de packages, sans modifier le coeur.

---

## 6. Capabilities installables et activables

Une capability est un package fonctionnel versionné qui peut étendre une application PayOS.

Elle peut contenir :

- des scripts API ;
- des pages ou composants ;
- des menus ;
- de la configuration ;
- des hooks de cycle de vie ;
- des hooks synchrones ou webhooks asynchrones
- des mappings ou ressources métier ;
- des batchs (non encore implémenté)
- un `manifest.json`.

Une capability est enregistrée comme application avec `category: capability`, puis rendue disponible via le mécanisme `extends`.

Le cycle de vie est géré par `cpm` :

```text
cpm --bundle-path <bundle> --install --id <capability>
cpm --bundle-path <bundle> --activate --id <capability> [--app <appId>] [--tenant <tenantId>]
cpm --bundle-path <bundle> --deactivate --id <capability> [--app <appId>] [--tenant <tenantId>]
cpm --bundle-path <bundle> --uninstall --id <capability>
```

Les états sont conservés dans :

- `registry.json` pour l'installation ;
- `activation.json` pour les scopes d'activation ;
- `events.ndjson` pour l'audit de cycle de vie ;
- `.capabilities/<id>/` pour le contenu installé.

**Point notable :** l'activation peut être globale, limitée à une application, limitée à un tenant, ou combinée. Cela permet un rollout progressif et réversible.

---

## 7. Multi-tenancy native

La multi-tenancy n'est pas seulement une convention de configuration. Elle est portée par le runtime.

PayOS gère notamment :

- `X-Tenant-Id` comme identifiant de tenant ;
- `X-Correlation-Id` pour la traçabilité de bout en bout ;
- validation du tenant à l'entrée ;
- propagation du tenant dans le contexte de requête ;
- ouverture d'un `TenantScope` ;
- enrichissement des logs via MDC ;
- quotas par tenant ;
- codes d'erreur normalisés (`400`, `403`, `429`) ;
- filtrage des capabilities et menus selon le tenant.

**Intérêt technique :** l'isolation tenant est appliquée de manière cohérente entre transports et ressources.

**Intérêt business :** PayOS peut servir plusieurs clients ou entités avec des niveaux d'activation et de visibilité différents, sans multiplier les déploiements.

---

## 8. Surface gouvernée pour partenaires et clients

PayOS matérialise une surface d'écosystème gouvernée : APIs, événements, SDKs futurs et extension points.

Les propriétés attendues sont :

| Propriété | Matérialisation dans PayOS |
|---|---|
| Authentification | OIDC, sessions, endpoints de sécurité. |
| Autorisation | Principal, tenant, application scope, politiques runtime. |
| Versioning | APIs, packages, capabilities, événements et manifests. |
| Auditabilité | Correlation ID, tenant ID, logs, événements de cycle de vie. |
| Isolation | Aucun accès direct obligatoire au runtime interne ou à la base. |

**Point notable :** la plateforme expose des points d'entrée contrôlés, pas son implémentation interne. C'est essentiel pour une plateforme destinée aux banques, fintechs, PSPs, payment facilitators et intégrateurs.

---

## 9. Sécurité OIDC et gestion de session

PayOS intègre la sécurité HTTP autour d'OIDC.

Les éléments notables sont :

- provider de sécurité configurable ;
- implémentation Nimbus recommandée ;
- fallback legacy pac4j ;
- endpoints `/me`, `/callback`, `/logout` ;
- gestion de session ;
- stratégie différente navigateur / XHR pour les redirections de login ;
- propagation du principal dans les scripts via `$Principal`.

**Intérêt technique :** la sécurité est centralisée et intégrée au runtime HTTP, au lieu d'être réimplémentée dans chaque application.

**Intérêt business :** PayOS s'aligne mieux avec les contraintes d'environnements financiers, où l'identité, la session et l'audit doivent être maîtrisés.

---

## 10. Hooks synchrones et webhooks asynchrones

PayOS propose deux niveaux d'événements autour des APIs et pages.

### Hooks synchrones

Les hooks s'exécutent dans la chaîne de traitement :

- `pre-request.js` avant un script API ;
- `post-request.js` après un script API ;
- `on-error.js` en cas d'erreur ;
- `page-pre-serve.js` avant l'assemblage d'une page ;
- `page-post-serve.js` après l'assemblage ;
- `page-on-error.js` en cas d'erreur page.

Ils permettent d'ajouter du contrôle, de l'enrichissement, de la validation ou de la délégation.

### Webhooks asynchrones

Les webhooks sortants permettent de notifier des systèmes externes. Ils supportent :

- déclaration dans `webhooks.json` ;
- événements natifs ou custom ;
- signature HMAC-SHA256 ;
- exécution asynchrone ;
- retry ;
- dead-letter ;
- déduplication entre événements natifs et événements émis par script.

**Point notable :** PayOS combine extension in-process et intégration événementielle sortante dans un modèle cohérent.

---

## 11. Configuration externalisée

PayOS est piloté par la configuration plutôt que par du code figé.

Les principaux éléments sont :

- `payos.json` comme point d'entrée de bundle ;
- `configDir` pour les fichiers externes ;
- résolution des variables d'environnement ;
- configuration des serveurs, applications, sécurité, data sources (`bootstrap.json`) et catalogues ;
- support des chemins relatifs au bundle via `--bundle-path`.

L'option `--bundle-path` définit le répertoire logique du runtime. Les chemins relatifs de configuration, catalogues, packages ou applications sont résolus par rapport à ce bundle (répertoire courant par défaut).

**Intérêt technique :** un même binaire peut être utilisé dans plusieurs environnements, avec une configuration adaptée.

**Intérêt delivery :** les bundles clients deviennent des unités d'installation cohérentes et transportables.

---

## 12. Hot reload runtime

Le hot reload est un atout technique important de PayOS. Le runtime surveille la configuration et peut recharger des changements sans imposer un cycle complet rebuild / redéploiement / redémarrage fonctionnel.

Les changements concernés incluent notamment :

- configuration globale ;
- fichiers du `configDir` ;
- descriptors applicatifs ;
- répertoire `.capabilities/` ;
- activation ou désactivation de capabilities ;
- évolutions de ressources applicatives selon les règles de chargement.

**Intérêt technique :** PayOS réduit la friction entre configuration, packaging et exécution. Les équipes peuvent ajuster un bundle ou activer une capability de manière beaucoup plus rapide qu'avec un packaging applicatif traditionnel.

**Intérêt business :** le hot reload soutient les déploiements progressifs, les pilotes client, les ajustements de configuration et les opérations de delivery sans immobiliser toute la plateforme.

**Limite importante :** le hot reload ne remplace pas une stratégie de release. Les changements structurels, les nouvelles dépendances Java, les migrations complexes ou les évolutions de contrats doivent rester maîtrisés par un processus de versioning, test et rollback.

---

## 13. Déploiement flexible : monolithe, modules et services

PayOS peut être assemblé selon plusieurs formes de déploiement.

### Runtime packagé monolithique

Le projet `payos-runtime` assemble le kernel, les serveurs HTTP/TCP/Queue, le database service, le connecteur NATS et le service webhook dans un JAR exécutable. Ce mode est adapté aux contextes où l'on veut une unité de déploiement simple, notamment :

- environnements on-premise ;
- démonstrations ;
- installations client contrôlées ;
- bundles autonomes ;
- plateformes où l'exploitation préfère un artefact unique.

### Modules et services séparés

Les mêmes capacités sont aussi structurées en modules séparés :

- `payos-kernel` pour le coeur runtime ;
- `payos-server-http` pour HTTP ;
- `payos-server-tcp` pour TCP ;
- `payos-server-queue` pour Queue/MoM ;
- `database-service` pour l'accès aux données ;
- `queue-service-nats` pour NATS ;
- `webhook-service-http` pour les webhooks.

Cette structure permet de faire évoluer, remplacer ou packager certains blocs indépendamment selon les besoins.

**Point notable :** PayOS n'impose pas un seul modèle opérationnel. Il peut être livré comme runtime intégré ou comme assemblage de modules, ce qui le rend compatible avec on-premise, cloud public, PaaS S2M ou self-managed PaaS.

---

## 14. Modularité par SPI, plugins et connecteurs remplaçables

PayOS s'appuie sur des points d'extension techniques pour éviter de figer les choix d'infrastructure dans le kernel.

Exemples notables :

- les serveurs concrets sont fournis par des modules dédiés ;
- les providers de serveur sont découverts via SPI Java ;
- les handlers TCP peuvent être chargés depuis des JARs externes ;
- la queue passe par l'abstraction `IQueueClient` ;
- l'implémentation NATS est un connecteur séparé ;
- le service de webhooks est externalisé ;
- le database service est un service spécialisé plutôt qu'une logique enfouie dans le kernel.

**Intérêt technique :** PayOS peut changer de transport, de connecteur ou de service périphérique sans remettre en cause le coeur applicatif.

**Intérêt architecture :** cette modularité soutient le principe “Rigid Core + Flexible Guest Layer” : le core reste stable, les intégrations et extensions évoluent autour.

---

## 15. Un codebase, plusieurs profils de déploiement

PayOS suit une logique : un coeur technique commun, plusieurs profils d'utilisation.

Les profils visés incluent :

| Profil | Ce que PayOS permet |
|---|---|
| On-premise | Runtime packagé, configuration explicite, connecteurs locaux, exploitation client. |
| Cloud public | Même runtime, mais avec bindings vers services managés lorsque le contexte l'autorise. |
| S2M PaaS | Déploiement standardisé et opéré par S2M. |
| Self-managed PaaS | Modèle plateforme industrialisé, mais opéré dans l'environnement client. |

**Point notable :** la variation ne doit pas créer des forks de plateforme. Elle est portée par la configuration, le packaging, les connecteurs et les profils de déploiement.

---

## 16. Chemins hot, warm et cold

PayOS peut être lu selon trois temporalités de traitement :

| Chemin | Temporalité | Exemples PayOS |
|---|---|---|
| Hot path | Sous la seconde, requête/réponse directe | APIs HTTP, échanges TCP, sécurité, routage, scripts courts. |
| Warm path | Secondes, traitement asynchrone | Queue, webhooks, callbacks, traitements différés. |
| Cold path | Minutes à heures | Batch, reporting, réconciliation, traitements d'audit ou de masse. |

Cette lecture est importante dans les paiements : tout ne doit pas être traité dans le chemin critique. PayOS permet de séparer les intégrations synchrones, les traitements asynchrones et les traitements de fond.

**Intérêt technique :** la plateforme peut choisir le bon canal selon la criticité et la latence attendue.

**Intérêt business :** cela aide à concilier performance transactionnelle, richesse fonctionnelle et coûts d'exploitation.

---

## 17. Package managers produit, application et capability

PayOS dispose d'un outillage dédié pour gérer le contenu d'un bundle :

| Outil | Rôle |
|---|---|
| `ppm` | Product Package Manager : gestion de packages produit. |
| `apm` | Application Package Manager : gestion d'applications. |
| `cpm` | Capability Package Manager : gestion de capabilities. |

Les trois commandes partagent le même principe : elles ciblent un bundle via `--bundle-path`.

**Point notable :** l'administration du contenu PayOS devient scriptable et industrialisable. Cela ouvre la voie à des catalogues, des pipelines de livraison et des bundles spécifiques par client ou environnement.

---

## 18. Data access et services optionnels

PayOS peut injecter des services optionnels dans les scripts, notamment `$DB`, `$Queue`, `$Secrets` et `$WebHooks`.

Cette approche permet de garder le kernel minimal tout en connectant des services spécialisés :

- accès aux bases via `database-service` ;
- queue / MoM via abstractions et implémentations externes ;
- secrets via `secret-service` et le provider SPI ;
- webhooks via `webhook-service-http` ;
- transports concrets dans des modules dédiés.

**Intérêt technique :** le runtime n'est pas monolithique dans ses intégrations. Les services peuvent évoluer séparément et être assemblés selon le déploiement.

---

## 19. Server-side i18n avec héritage

PayOS supporte la localisation côté serveur via `$I18n`.

Les traductions sont résolues à partir :

- d'overrides explicites ;
- du principal ;
- de `Accept-Language` ;
- d'une configuration de fallback ;
- des bundles `i18n/{locale}/*.json`.

Le runtime fusionne les traductions héritées depuis les applications ou capabilities parentes, puis applique les traductions locales en override.

**Point notable :** l'i18n suit le même modèle d'héritage que les ressources applicatives. Une capability peut donc enrichir l'expérience utilisateur sans casser les textes existants.

---

## 20. Observabilité orientée tenant et corrélation

PayOS intègre la traçabilité comme une dimension transverse du runtime.

Les éléments notables sont :

- `X-Correlation-Id` propagé entre transports, contexte, réponses et logs ;
- `X-Tenant-Id` conservé avec le contexte d'exécution ;
- enrichissement MDC avec `tenantId`, `correlationId` et `appId` ;
- normalisation des erreurs multi-tenant ;
- événements de cycle de vie des capabilities dans `events.ndjson` ;
- logs SLF4J/Logback plutôt que sorties console dispersées.

**Intérêt technique :** le diagnostic et l'investigation sont facilités dans un environnement multi-client.

**Intérêt compliance :** corrélation, tenant et auditability sont essentiels dans des contextes financiers régulés.

---

## 21. Idempotency service

PayOS intègre un service d'idempotence pour sécuriser les appels API sensibles contre les doubles soumissions, retries réseau ou répétitions accidentelles.

Le fonctionnement est le suivant :

- le runtime lit une clé d'idempotence dans le header configuré, par défaut `X-Idempotency-Key` ;
- si la clé est absente ou vide, la requête est bloquée avant l'exécution du script avec une erreur `400 Bad Request` — sauf si `failOnAbsenceOfIdempotencyKey` est mis à `false`, auquel cas la requête est simplement exécutée sans vérification d'idempotence ;
- si la clé existe et qu'une réponse non expirée est déjà stockée, PayOS retourne directement la réponse en cache ;
- la réponse rejouée reçoit le header `X-Idempotency-Replayed: true` ;
- si aucune réponse n'est stockée pour cette clé, la requête est exécutée et la réponse est stockée avec un TTL après exécution réussie ;
- le TTL par défaut est de 24 heures (`86400` secondes).

Tous ces paramètres (`enabled`, `ttlSeconds`, `headerName`, `failOnAbsenceOfIdempotencyKey`) sont configurables via le bloc `idempotency` de `bootstrap.json`, avec repli sur propriété système ou variable d'environnement — voir [configuration/idempotency.md](configuration/idempotency.md).

Le service est conçu autour d'une abstraction de store :

| Élément | Rôle |
|---|---|
| `IdempotencyService` | Orchestre la détection, le replay et le stockage des réponses. |
| `IdempotencyConfig` | Configure activation, TTL et nom du header. |
| `IIdempotencyStore` | Contrat de stockage. |
| `InMemoryIdempotencyStore` | Store mémoire, utile pour exécution simple ou tests. |
| `DatabaseIdempotencyStore` | Store persistant pour scénarios où l'idempotence doit survivre au process. |

**Intérêt technique :** l'idempotence est traitée au niveau runtime, avant et après l'exécution du script, ce qui évite de réimplémenter cette logique dans chaque endpoint critique.

**Intérêt business :** dans les paiements, cela réduit les risques de double opération lors de retries client, timeouts, refreshs ou erreurs réseau.

**Point d'attention :** la qualité de garantie dépend du store utilisé. Un store mémoire est simple mais local au process ; un store persistant est préférable pour des déploiements multi-instance ou critiques.

---

## 22. Audit log structuré

PayOS dispose d'un mécanisme d'audit log dédié, distinct du logging applicatif classique.

Le point d'entrée est `AuditLogger`, une façade statique qui délègue à une implémentation `IAuditLogger`. L'implémentation par défaut écrit des événements JSON vers le logger SLF4J `AUDIT`.

Événements couverts :

- authentification réussie ou échouée ;
- logout ;
- création et destruction de session ;
- autorisation accordée ou refusée ;
- activation du tenant simulator ;
- startup et shutdown système ;
- exécution d'API ;
- échec de déchiffrement ;
- événements métier custom via `AuditEvent`.

Chaque événement standardise les champs clés :

| Champ | Usage |
|---|---|
| `timestamp` | Horodatage de l'événement. |
| `event` | Type d'événement. |
| `userId` | Utilisateur concerné lorsque connu. |
| `tenantId` | Tenant concerné. |
| `appId` | Application concernée. |
| `correlationId` | Corrélation de bout en bout. |
| `path` | Ressource ou chemin concerné. |
| `result` | Résultat (`SUCCESS`, `FAILURE`, `DENIED`, etc.). |

**Intérêt technique :** l'audit log est extensible : l'implémentation peut être remplacée au démarrage pour router les événements vers une queue, un SIEM, une base ou un stockage append-only.

**Intérêt compliance :** les événements sont structurés, corrélables et séparables des logs techniques. Cela facilite investigation, supervision sécurité et conservation réglementaire.

---

## 23. Couverture de certaines exigences PCI DSS

PayOS ne remplace pas une certification PCI DSS complète, qui dépend aussi de l'infrastructure, des procédures, de l'exploitation et du périmètre audité. En revanche, plusieurs mécanismes techniques du runtime contribuent déjà à couvrir certaines exigences.

| Zone PCI DSS | Contribution PayOS |
|---|---|
| Req 10 - audit et traçabilité | `AuditLogger`, événements JSON, API execution audit, corrélation, tenant, session et sécurité. |
| Req 7 / 8 - contrôle d'accès | OIDC, principal, rôles, endpoints sécurité, checks d'autorisation centralisés. |
| Req 6 - développement sécurisé | sandbox GraalVM avec accès contrôlé aux bindings, pas d'accès direct libre au runtime interne. |
| Req 10.5 / 10.7 - protection et rétention des logs | support d'un logger `AUDIT` séparé ; l'append-only et la rétention sont à appliquer via la configuration logback et l'infrastructure. |
| Traçabilité multi-tenant | propagation de `X-Tenant-Id` et `X-Correlation-Id`, MDC, réponses et logs enrichis. |
| Protection contre log injection | nettoyage des champs sensibles ou suspects dans certains événements d'audit. |
| Cryptographic monitoring | audit des échecs de déchiffrement comme signal de tampering potentiel. |

**Point notable :** PayOS intègre des briques favorables à PCI DSS dans le runtime plutôt que de les laisser à chaque application.

**Point d'attention :** le document doit rester précis : PayOS fournit une base technique alignée avec certaines exigences PCI DSS, mais la conformité finale dépend du déploiement, des runbooks, du durcissement, de la rétention des logs, de la gestion des secrets, des contrôles opérationnels et des audits.

---

## 24. Connector framework (SPI backends) et connector framework métier/paiement

PayOS est conçu pour brancher des technologies d'intégration sans modifier le kernel.

> **Deux mécanismes distincts partagent le mot « connecteur »** : les connecteurs SPI
> ci-dessous (backends database/queue/secret/webhook, câblés dans `BootServer`) et un
> **connector framework métier/paiement** plus récent (`IConnector`, `$Connector(...)`,
> `connectors.json`, module `connector-sdk`) — voir la sous-section dédiée après le tableau.

Le connector framework (SPI) se matérialise par plusieurs niveaux :

| Niveau | Mécanisme | Exemple |
|---|---|---|
| Transport server providers | Java SPI via `ServerProvider` et `ServiceLoader` | HTTP, TCP, Queue. |
| Queue connector | Interface `IQueueClient` et factories de connecteurs | `queue-service-nats`. |
| TCP plugins | Découverte de JARs externes | `TcpMessageDecoder`, `TcpMessageEncoder`, `TcpMessageHandler`. |
| Webhook dispatcher | Implémentation injectable via runtime config | `webhook-service-http`. |
| Audit sink | Interface `IAuditLogger` remplaçable | SLF4J par défaut, queue/SIEM possible. |
| Database service | Service optionnel injecté dans les scripts via `$DB` | `database-service`. |
| Secret provider | Interface `ISecretProviderFactory` via SPI | `secret-service-filesystem`, `secret-service-vault` ; providers cloud/HSM additionnels possibles. |

**Intérêt technique :** les intégrations sont des adaptateurs autour du kernel. PayOS peut accueillir un nouveau transport, un nouveau broker, un nouveau sink d'audit ou un nouveau dispatcher sans casser le modèle applicatif.

**Intérêt delivery :** cela facilite l'adaptation aux contraintes clients : on-premise, réseau fermé, broker imposé, SIEM existant, base de données ou connecteur spécifique.

**Intérêt architecture :** ce modèle protège le core et évite les forks. La variabilité est portée par les connecteurs, les providers et la configuration.

### Connector framework métier/paiement (`$Connector`) — mécanisme distinct

Un second mécanisme, plus récent et sans rapport avec les connecteurs SPI ci-dessus, permet
d'intégrer des systèmes de paiement (réseaux carte, switch national, PSP) directement depuis
les scripts, via `$Connector(type[, name]).execute(payload)`. Construit sur 5 épics :

| Capacité | Mécanisme |
|---|---|
| Contrat SDK indépendant | Module Maven `connector-sdk` (`IConnector`, `AbstractConnector`) — aucune dépendance sur `payos-kernel` |
| Descripteur auto-déclaratif | `META-INF/connector.properties` (`connector.type`, `connector.name`, `connector.api.version`, ...) |
| Configuration opérateur | `connectors.json` — type/nom/JAR/paramètres, tokens `${...}` pour les secrets |
| Isolation et rechargement à chaud | Classloading isolé par connecteur, hot-reload piloté par `config-hot-reload-enabled` |
| Idempotence et déduplication plateforme | Le connecteur ne décide jamais de suppression/replay — la plateforme s'en charge |
| Politique de retry déterministe | Budget de tentatives et catégories retryables configurables (code, pas encore exposé en JSON) |
| État d'exécution persistant | Traçabilité RUNNING/SUCCEEDED/RETRYING/FAILED par tentative |
| Routage terminal DLQ / Connector State | Décision auditable après épuisement des retries ou erreur permanente |
| Diagnostics dédiés | Catégorie d'évènement `Diagnostics` (`nature: "connector"`), catégorie SLF4J `DIAGNOSTICS`, indépendante du journal d'audit PCI-DSS |

**Statut :** entièrement implémenté et testé (`payos`, `payos-connector-sdk`), mais **pas encore
câblé dans `BootServer`** — `$Connector` n'est donc pas disponible dans un déploiement en
production aujourd'hui. Détails complets :
[connector-framework-parameters-v2-2026-07-12.md](configuration/connector-framework-parameters-v2-2026-07-12.md).

---

## 25. Secret service et binding `$Secrets`

PayOS intègre un service de gestion des secrets accessible aux scripts via le binding `$Secrets`.

Le service est désactivé par défaut et s'active via la clé `secret-service.configuration.enabled: true` dans `bootstrap.json`. L'implémentation concrète est un connecteur chargé depuis `connectors-dir` via SPI Java (`ISecretProviderFactory`).

Deux connecteurs sont livrés avec PayOS : `secret-service-filesystem` (référence, stockage chiffré AES-256-GCM sur disque local) et `secret-service-vault` (HashiCorp Vault KV v2, pour la production).

#### CLI d'administration `spm`

Le module `secret-service-filesystem` embarque un outil CLI autonome (`spm.jar`) pour provisionner et gérer les secrets du provider filesystem sans passer par une API applicative.

```bash
# Générer la clé maîtresse AES-256
spm keygen --out /opt/payos/secrets/.keyfile

# Stocker un secret
spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key --value sk_live_xxx --type api-key

# Rotation d'un secret (version auto-incrémentée)
spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key --value sk_live_yyy

# Lister, décrire, supprimer
spm list     --root ... --keyfile ... --tenant acme
spm describe --root ... --keyfile ... --tenant acme --name stripe-api-key
spm delete   --root ... --keyfile ... --tenant acme --name stripe-api-key
```

Des scripts d'installation (`install.sh` / `install.ps1`) sont fournis dans le module pour rendre la commande `spm` disponible globalement dans le PATH. Voir le [Guide des outils CLI](operations/cli-tools-guide.md#8-spm--secret-package-manager).

#### API `$Secrets` dans les scripts

| Méthode | Description |
|---------|-------------|
| `$Secrets.get(name)` | Retourne la valeur du secret `name` pour le tenant courant. |
| `$Secrets.list()` | Retourne la liste des noms de secrets disponibles pour le tenant courant. |
| `$Secrets.tokenize(value)` | Remplace une valeur sensible par un token opaque (UUID v4), non réversible sans le provider. |
| `$Secrets.detokenize(token)` | Retrouve la valeur sensible associée à un token. |

```javascript
const token = $Secrets.get("external-api-token");
$Response.setBody({ ok: true });
```

Les secrets sont scopés au tenant courant : l'accès est isolé par tenant sans configuration supplémentaire. `$Secrets` n'expose pas d'écriture (`set`/`delete`/`describe`) — ces opérations passent par `spm`, l'API Vault, ou l'API Java directe.

**Intérêt sécurité :** les credentials (clés API, mots de passe, certificats) ne transitent pas dans les fichiers de configuration. Le provider est remplaçable sans modifier les scripts (Vault est déjà livré ; KMS cloud, HSM, etc. peuvent être ajoutés via un connecteur custom).

**Intérêt PCI DSS :** le contrôle d'accès aux secrets via un provider dédié contribue à Req 3 (protection des données sensibles) et Req 7 (contrôle d'accès aux données).

---

## 26. Gestion des erreurs métier avec `$Errors`

PayOS expose un binding `$Errors` dans tous les scripts API et hooks pour lever des erreurs métier normalisées.

Quand un script appelle `$Errors`, le runtime capture l'exception et retourne automatiquement une réponse JSON structurée avec le bon statut HTTP, sans que le script ait besoin de construire la réponse manuellement.

#### API `$Errors`

| Méthode | Statut | Description |
|---------|--------|-------------|
| `$Errors.business(code, message)` | `400` | Erreur métier générique. |
| `$Errors.business(code, message, statusCode)` | `statusCode` | Erreur métier avec statut HTTP personnalisé. |
| `$Errors.business(code, message, statusCode, details)` | `statusCode` | Avec objet `details` optionnel dans la réponse. |
| `$Errors.badRequest(code, message)` | `400` | Requête invalide. |
| `$Errors.conflict(code, message)` | `409` | Conflit de ressource. |
| `$Errors.notFound(code, message)` | `404` | Ressource introuvable. |

#### Format de la réponse JSON

```json
{
  "error": "Business error",
  "code": "DUPLICATE_PAYMENT",
  "message": "Un paiement avec cette référence existe déjà.",
  "details": { "existingId": "PAY-001" }
}
```

Le champ `details` n'est inclus que s'il est fourni. Le `Content-Type` est toujours `application/json; charset=UTF-8`.

```javascript
// Dans un script API
const existing = $DB.findByRef(ref);
if (existing) {
    $Errors.conflict("DUPLICATE_PAYMENT", "Un paiement avec cette référence existe déjà.");
}
```

**Intérêt technique :** le modèle d'erreur est uniforme dans toute la plateforme. Le script reste lisible (flux positif uniquement) et l'erreur est capturée proprement dans le pipeline, incluant le hook `on-error` et le dispatch webhook natif `api.on-error`.

---

## 27. Points de différenciation technique

Les features ci-dessus forment un ensemble cohérent. Le caractère notable de PayOS ne vient pas d'une seule technologie, mais de leur combinaison :

1. Un runtime stable et minimal.
2. Des applications externalisées.
3. Des capabilities activables par scope.
4. Un hot reload utile pour les ajustements runtime et le delivery progressif.
5. Une surface gouvernée pour les intégrations.
6. Une multi-tenancy appliquée au coeur du traitement.
7. Des scripts contrôlés pour accélérer l'adaptation métier.
8. Des hooks et webhooks pour intégrer les processus externes.
9. Des package managers pour industrialiser l'installation.
10. Une configuration externalisée pour supporter on-premise, PaaS et cloud.
11. Une architecture modulaire capable d'être assemblée en runtime monolithique ou en modules/services spécialisés.
12. Une séparation des chemins hot, warm et cold pour mieux aligner technique, performance et coûts.
13. Un service d'idempotence utile pour les APIs de paiement et les opérations sensibles aux retries.
14. Un audit log structuré et remplaçable, aligné avec des besoins de compliance.
15. Des briques runtime contribuant à certaines exigences PCI DSS.
16. Un connector framework pour brancher transports, queues, audit sinks, webhooks et plugins sans forker la plateforme.
17. Un service de secrets pluggable (`$Secrets`) isolant les credentials des scripts et de la configuration.
18. Un modèle d'erreurs métier uniforme (`$Errors`) évitant la construction manuelle de réponses d'erreur dans les scripts.

En résumé, PayOS est notable techniquement parce qu'il cherche à concilier deux objectifs souvent opposés :

- **stabilité du core** pour les environnements financiers régulés ;
- **flexibilité contrôlée** pour livrer rapidement des offres, variantes et intégrations.

---
