# Guide développeur — Framework de connecteurs (`$Connector`)

Ce guide couvre l'utilisation du binding `$Connector` **depuis un script applicatif** — c'est-à-dire le point de vue du développeur qui appelle un connecteur métier/paiement déjà déployé (`CardNetwork`/`visa`, `Switch`/`cmi`, etc.), pas celui de la personne qui écrit le connecteur lui-même. Pour écrire un connecteur (implémenter `IConnector`, empaqueter le JAR, le descripteur SPI), voir [connector-developer/README.md](../connector-developer/README.md) — ce guide, déjà complet et vérifié contre le code, ne sera pas dupliqué ici. Pour la configuration opérateur complète (`connectors.json`, versions d'API, politiques de retry/DLQ), voir [configuration/connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md).

> **Câblé dans `BootServer` depuis le 2026-07-27.** `ConnectorFrameworkInitializer.initialize(...)` scanne `<runtimeBaseDir>/connectors/`, valide les JARs contre `connectors.json`, initialise les connecteurs, puis appelle `PayOSConfig.setConnectorRegistry(...)` — au démarrage et à chaque hot-reload. Chaque tenant déclaré dans `multitenancy.tenants` voit la **même** liste de connecteurs partagée (repli sur un tenant unique `"default"` si la multi-tenancy n'est pas configurée — voir §6 pour le détail exact du modèle de tenant-scoping). Un `connectors.json` absent ou vide reste une configuration normale et pleinement supportée : le framework reste simplement inactif.

## 1. Résolution — comment `$Connector(...)` trouve un connecteur

`$Connector` est directement **appelable comme une fonction** (pas un objet avec une méthode `.get(...)`) et accepte 0, 1 ou 2 arguments :

```javascript
var handle1 = $Connector('CardNetwork', 'visa');  // 2 arguments : type + nom exact
var handle2 = $Connector('CardNetwork');           // 1 argument : première instance de ce type
var handle3 = $Connector('visa');                  // 1 argument : recherche directe par nom
```

| Appel | Résolution |
| --- | --- |
| `$Connector(type, name)` | Correspondance exacte type + nom. |
| `$Connector(typeOrName)` | Essaie d'abord `typeOrName` comme **type** (retourne la première instance enregistrée de ce type) ; si aucune correspondance, retente comme **nom** direct. Si plusieurs connecteurs de types différents partagent ce nom, la résolution est ambiguë (voir §4). |
| `$Connector()` | Toujours un échec de résolution (résout la chaîne littérale `"unavailable"`, qui n'est jamais enregistrée). |

Chaque appel retourne un **nouvel objet handle** — rien n'est mis en cache côté script entre deux appels, même pour le même type/nom.

## 2. Le handle retourné — `execute(...)` et les accesseurs

Le handle expose exactement ceci aux scripts (jamais l'implémentation Java du connecteur elle-même) :

```javascript
var response = handle.execute({ amount: 100, currency: "MAD" });

response.status();        // "SUCCESS" | "ERROR"
response.data();          // objet de résultat (uniquement en cas de succès)
response.errorCategory(); // "NONE" | "RETRYABLE_ERROR" | "PERMANENT_ERROR" | "TIMEOUT" | "UNKNOWN_ERROR"
response.errorCode();     // code d'erreur stable, ou null
response.errorMessage();  // message d'erreur, ou null
response.correlationId(); // corrélation de la requête courante
response.tenantId();      // tenant courant
```

`.execute(payload)` est la **seule** méthode d'interaction — il n'y a pas d'étape de connexion/lookup séparée. Le `payload` est un objet JS simple, converti côté plateforme en `Map<String, Object>` ; il n'y a pas de type d'enveloppe imposé.

Le handle expose aussi, avant tout appel à `execute(...)`, `status()`/`errorCode()`/`message()` reflétant l'échec de **résolution** lui-même (connecteur introuvable, ambigu) — voir §4.

### 2.1 Exemple complet dans un script

```javascript
function execute(request, controlData) {
    var handle = $Connector('CardNetwork', 'visa');
    var response = handle.execute({ amount: controlData.amount, currency: "MAD" });

    if (response.status() == "SUCCESS") {
        return response.data();
    }

    switch (response.errorCategory()) {
        case "RETRYABLE_ERROR":
        case "TIMEOUT":
            $Errors.serviceUnavailable("Payment network temporarily unavailable, retry later");
            break;
        default:
            $Errors.badRequest(response.errorMessage());
    }
}
```

## 3. Idempotence — automatique, pas un paramètre de `execute(...)`

La clé d'idempotence **n'est jamais passée à `execute(...)`** — elle est liée automatiquement depuis l'en-tête `X-Idempotency-Key` de la requête HTTP courante (ou le nom d'en-tête configuré pour le service d'idempotence général, si différent). Le script n'a rien à faire de spécial pour en bénéficier.

Points importants à connaître avant d'utiliser un connecteur en production :

- Si le descripteur du connecteur déclare `connector.requires.idempotency=true` (typiquement les connecteurs de paiement non répétables), un appel `execute(...)` **sans** clé d'idempotence dans la requête est rejeté immédiatement avec `errorCode() == "CONNECTOR_IDEMPOTENCY_KEY_REQUIRED"`, sans jamais atteindre le connecteur.
- Un rejeu avec la **même** clé retourne la réponse mise en cache de la tentative précédente (`REPLAY`) plutôt que de ré-exécuter le connecteur ; un rejeu concurrent pendant qu'une exécution est encore en cours est suspendu (`SUPPRESS`, retourne une `RETRYABLE_ERROR`).
- **⚠️ La clé d'idempotence n'est pas qualifiée par type/nom de connecteur** dans le magasin de déduplication par défaut — si un même script appelle deux connecteurs différents avec la même clé d'idempotence de requête, leurs états de déduplication peuvent entrer en collision. Ne réutilisez jamais la même clé pour représenter deux opérations métier distinctes dans le même script.
- Ce mécanisme est **complètement séparé** du service d'idempotence HTTP général (`idempotency.enabled`/`ttlSeconds`/`headerName`, voir [configuration/idempotency.md](../configuration/idempotency.md)) qui met en cache la réponse HTTP entière indépendamment de tout appel `$Connector`. Les deux lisent la même valeur d'en-tête mais fonctionnent comme deux magasins indépendants.

## 4. Résolution en échec — pas d'exception, un handle en erreur

Si la résolution échoue (connecteur non trouvé, ambigu, ou registre absent), `handle.execute(...)` retourne une `ConnectorResponse` en erreur — **il n'y a jamais d'exception JavaScript levée pour un connecteur non résolu**, toujours vérifier `status()` :

```javascript
var handle = $Connector('CardNetwork', 'unknown-provider');
var response = handle.execute({ amount: 100 });
// response.status() == "ERROR", response.errorCategory() == "PERMANENT_ERROR"
```

## 5. Erreurs d'exécution — jamais l'exception Java brute

Si l'implémentation du connecteur lève une exception pendant `execute(...)`, le script ne voit **jamais** l'exception Java d'origine — elle est journalisée côté serveur puis traduite en `ConnectorResponse` d'erreur sanitisée :

| Cause côté connecteur | `errorCategory()` côté script | `errorCode()` |
| --- | --- | --- |
| `ConnectorExecutionException` avec une catégorie explicite | La catégorie déclarée par le connecteur (`RETRYABLE_ERROR`/`PERMANENT_ERROR`/`TIMEOUT`/`UNKNOWN_ERROR`) | Le code fourni par le connecteur |
| N'importe quelle autre `RuntimeException` non gérée | Toujours `UNKNOWN_ERROR` | `CONNECTOR_UNCAUGHT_EXCEPTION` (message générique, jamais le détail réel) |

Ne construisez jamais de logique métier qui dépend du texte exact d'un message d'erreur venant d'un connecteur tiers — seule la paire `errorCategory()`/`errorCode()` est un contrat stable.

## 6. Comment `$Connector` devient disponible, et son modèle de tenant-scoping

`ApiResourceHandler` (et `HookEngine`) injectent `$Connector` uniquement si `PayOSConfig.getConnectorRegistry()` est non-null. Ce registre est peuplé par `ConnectorFrameworkInitializer.initialize(...)` (`ma.s2m.payos.config.connector`), appelé depuis `BootServer` au démarrage et à chaque hot-reload : chargement de `connectors.json`, résolution des références `${ENV_VAR}` dans les paramètres, scan de `<runtimeBaseDir>/connectors/`, validation des JARs, instanciation isolée (un `URLClassLoader` par connecteur), puis construction du `TenantConnectorRegistry`.

**Modèle de tenant-scoping** (décision documentée en [configuration/connector-framework-parameters-v2-2026-07-12.md §5](../configuration/connector-framework-parameters-v2-2026-07-12.md#5-tenant-scoping)) : `connectors.json` n'a aucun champ tenant — chaque connecteur `READY` configuré globalement est visible par **tous** les tenants déclarés dans `multitenancy.tenants` (même liste d'entrées partagée). Si `multitenancy.tenants` est absent ou vide, un unique tenant `"default"` est enregistré. Il n'existe aujourd'hui aucun mécanisme pour restreindre un connecteur à un sous-ensemble de tenants.

Si `connectors.json` est absent, vide, ou si aucun JAR valide n'est trouvé dans `connectors/`, le framework reste inactif sans faire échouer le démarrage — `$Connector` est alors simplement absent des scripts, exactement comme n'importe quel autre binding optionnel (`$Cache`, `$SlidingWindow`, ...). Vérifiez toujours sa présence si votre code doit rester compatible avec un déploiement où aucun connecteur n'est configuré :

```javascript
if (typeof $Connector !== "undefined") {
    var response = $Connector('CardNetwork', 'visa').execute(payload);
    // ...
}
```

## 7. Tester un connecteur sans passer par un script

Si vous développez ou intégrez un connecteur et voulez vérifier son comportement sans dépendre du câblage `BootServer` (actuellement absent, §6), `ConnectorTestHarness` (module `payos-connector-sdk`, package `test`) permet d'appeler `init`/`execute`/`close` directement en Java, sans runtime PayOS complet — voir [`payos-connector-sdk/README.md` §7](../../payos-connector-sdk/README.md#7-exemple-minimal-complet) pour un exemple.

## 8. Références

- [connector-developer/README.md](../connector-developer/README.md) *(anglais)* / [`payos-connector-sdk/README.md`](../../payos-connector-sdk/README.md) *(français)* — guide complet pour **écrire** un connecteur (contrat `IConnector`, descripteur, SPI, empaquetage, versionnement) ; les deux documents couvrent exactement le même contenu vérifié contre le code, dans deux langues.
- [configuration/connector-framework-parameters-v2-2026-07-12.md](../configuration/connector-framework-parameters-v2-2026-07-12.md) — référence de configuration complète (`connectors.json`, compatibilité de version d'API, politiques de retry/DLQ, diagnostics).
- [scripting-bindings.md](scripting-bindings.md) — index de tous les bindings `$`, y compris `$Connector` et `$Errors`.
- [configuration/idempotency.md](../configuration/idempotency.md) — le service d'idempotence HTTP général, distinct du mécanisme décrit en §3.
