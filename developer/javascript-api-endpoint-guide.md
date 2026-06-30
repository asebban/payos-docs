# Guide JavaScript des Endpoints API dans PayOS

Dernier alignement: 2026-04-26

Ce document explique, de façon pratique et détaillée, comment écrire des scripts JavaScript pour les endpoints API dynamiques dans PayOS.

## 1. Vue d'ensemble

Un endpoint API dynamique est défini par une ressource JavaScript qui expose trois fonctions :

- `loadControlData(request)`
- `execute(request, controlData)`
- `emitInsight(request, response, payload)`

Le moteur PayOS :

1. Résout la ressource via la route API ;
2. Crée un contexte d'exécution JavaScript ;
3. Injecte des objets globaux (`$Response`, `$Api`, `$App`, `$Principal`, `$Tenant`, `$Logger`, `$Library`, `$I18n`, `$Errors`, `$DB` si configuré, `$Queue` si configuré) ;
4. Appelle les fonctions du cycle de vie dans l'ordre ci-dessus.

## 2. Objets Request et Response

Ce sont respectivement les objets passés aux fonctions JS et l'objet de retour de la fonction `execute()`

### 2.1. Request

Cette section décrit les fonctions réellement exposées par les objets Java `Request` et `Response` dans le runtime. `Request` représente la requête unifiée et normalisée pour les scripts JS.

Fonctions principales :

- `getMethod() : String`
- `getType() : String`
- `setType(type: String) : void`
- `isApiRequest() : boolean`
- `isPageRequest() : boolean`
- `getBody() : String`
- `getJsonBody() : Map<String, String>` (Map<String, String> est interprété comme objet Json côté JS)
- `getParameters() : Map<String, String>`
- `setParameters(parameters: Map<String, String>) : void`
- `getPathVariables() : Map<String, String>`
- `setPathVariables(pathVariables: Map<String, String>) : void`
- `getContextData() : Map<String, Object>`
- `setContextData(contextData: Map<String, Object>) : void`
- `addToContextData(key: String, value: Object) : void`
- `getPath() : String`
- `setPath(path: String) : void`
- `getHeaders() : Map<String, String>`
- `setHeaders(headers: Map<String, String>) : void`
- `addHeader(key: String, value: String) : void`
- `removeHeader(key: String) : void`
- `getHeader(key: String) : String`

Points d'attention sur `Request.getJsonBody()` :

- valide réellement le JSON ;
- accepte objet JSON et tableau JSON ;
- lève une erreur si le body n'est ni un objet ni un tableau JSON.

Exemple :

```javascript
var method = request.getMethod();
var isApi = request.isApiRequest();
var tenant = request.getHeader("X-Tenant-Id");
var body = request.getJsonBody();
var userId = request.getPathVariables()?["userId"];
```

### 2.2. Response

`Response` représente la réponse renvoyée au client et permet de construire le body, le status et les en-têtes.

Fonctions principales :

- `getBody() : byte[]`
- `getJsonBody() : Json`
- `setBody(body: byte[]) : void`
- `setBody(body: String) : void`
- `setBody(body: Map<String, Object>) : void`
- `setBody(value: Value) : void` (objet GraalVM)
- **`setJsonBody(body: Map<String, Object>) : void`** ← à préférer en JS pour les réponses JSON
- **`setJsonBody(body: Object) : void`** ← à préférer en JS pour les réponses JSON
- `write(body: byte[]) : Response`
- `write(body: String) : Response`
- `write(body: Map<String, Object>) : Response`
- `write(value: Value) : Response`
- `getStatusCode() : int`
- `setStatusCode(statusCode: int) : void`
- `getMessage() : String`
- `setMessage(message: String) : void`
- `getCookies() : Map<String, String>`
- `setCookies(cookies: Map<String, String>) : void`
- `addCookie(name: String, value: String) : void`
- `getCookie(name: String) : String`
- `getPath() : String`
- `setPath(path: String) : void`
- `getHeaders() : Map<String, String>`
- `setHeaders(headers: Map<String, String>) : void`
- `addHeader(key: String, value: String) : void`
- `removeHeader(key: String) : void`
- `getHeader(key: String) : String`
- `getContextData() : Map<String, Object>`
- `setContextData(contextData: Map<String, Object>) : void`
- `addContextData(key: String, value: Object) : void`
- `getContextData(key: String) : Object`
- `removeContextData(key: String) : void`

> **`contextData` n'est pas renvoyé au client**
>
> `contextData` sert à transporter des données internes au backend (consommées côté Java, ex. après l'exécution du script JS) — contrairement à `body`, `statusCode`, `message` et `cookies`, il n'est jamais sérialisé dans la réponse HTTP.

Points d'attention sur `Response.getJsonBody()` :

- valide réellement le JSON ;
- accepte objet JSON et tableau JSON ;
- lève une erreur si le body n'est ni un objet ni un tableau JSON.

> **`setJsonBody` vs `setBody` pour les réponses JSON**
>
> En contexte JavaScript (GraalVM), `setBody` est surchargé plusieurs fois en Java. Le moteur JS peut ne pas résoudre la bonne surcharge lorsqu'on lui passe un objet JS natif.
> `setJsonBody` est l'entrée dédiée, non surchargée, prévue pour les scripts : elle sérialise l'objet passé en JSON, encode en UTF-8 et positionne automatiquement l'en-tête `Content-Type: application/json`.
>
> **Règle** : utiliser `setJsonBody` dès que la réponse est un objet ou un tableau JSON.

Exemple :

```javascript
$Response.setJsonBody({ status: "OK", data: { id: "123" } });
$Response.setStatusCode(200);
$Response.addCookie("SESSION", "abc123");
return $Response;
```

## 3. Objets globaux disponibles dans le script JS (injectés par le kernel)

### 3.1. `$DB`

Service d'accès base de données (`IDatabaseService`) exposé aux scripts JavaScript. Permet les opérations de lecture/écriture génériques et la gestion transactionnelle.

#### Accès base de données avec `$DB`

#### Lecture

```javascript
var params = $DB.newParams();
params.put("username", "johndoe");
const rows = $DB.list("select p from Payment p where p.username = :username", params, 0, 100);
```

Important : le tenant courant est injecté automatiquement.

- En lecture, un filtre `tenantId` est ajouté automatiquement si la requête ne le contient pas déjà.
- Le paramètre `tenantId` est alimenté automatiquement côté service.

#### Écriture en auto-commit (par défaut)

```javascript
const created = $DB.save("User", {
  username: "johndoe",
  firstName: "John",
  lastName: "Doe"
});
```

Important : en écriture, la colonne `tenantId` est injectée automatiquement dans l'entité (`save`, `update`, `delete`).

#### Transaction manuelle

```javascript
$DB.setAutoCommit(false);
$DB.beginTransaction();

try {
  var body = request.getJsonBody();
  $DB.update("Account", { id: body?.from, balance: body?.fromBalance });
  $DB.update("Account", { id: body?.to, balance: body?.toBalance });

  $DB.commitTransaction();
} catch (e) {
  $DB.rollbackTransaction();
  throw e;
} finally {
  $DB.setAutoCommit(true);
}
```

#### Référence complète des fonctions `$DB`

Les fonctions ci-dessous correspondent à l'interface `IDatabaseService` exposée au runtime.

##### Gestion de scope / tenant / session

- `beginRequestScope()`
- `endRequestScope()`
- `setCurrentTenant(tenantId)`
- `clearCurrentTenant()`
- `setRetiredSessionFactoryCloseDelaySeconds(seconds)`

##### Gestion transactionnelle

- `setAutoCommit(autoCommit)`
- `isAutoCommit()`
- `beginTransaction()`
- `commitTransaction()`
- `rollbackTransaction()`

##### Paramètres de requêtes

- `newParams()`

##### Requêtes de lecture

- `list(query, params, firstResult, maxResults)`
- `list(query, params, maxResults)`
- `find(query, params)`
- `first(query, params)`
- `unique(query, params)`
- `findAll(entityName)`
- `get(entityName, id)`

##### Requêtes d'écriture

- `executeUpdate(query, params)`
- `save(entityName, entity)`
- `update(entityName, entity)`
- `delete(entityName, entity)`
- `deleteById(entityName, id)`

##### Exemple rapide couvrant plusieurs fonctions

```javascript
var params = $DB.newParams();
params.put("username", "johndoe");

var payments = $DB.list("select p from Payment p where p.username = username", params, 0, 50);

var firstPayment = $DB.first("select p from Payment p where p.username = :username order by p.createdAt desc", params);

var oneUser = $DB.unique("select u from User u where u.username = :username", params);
```

## 3.2. `$App` : contexte applicatif

`$App` permet au script de lire ou modifier l'application active.

**Exemple** :

```javascript
const app = $App.getApplication(); // get default application (app that owns the endpoint)

if (!app || app !== "payos") {
  var app = $App.get("payos"); // get "payos" application
  $App.setApplication(app); // set the app as the new App
}
```

expose les fonctions suivantes:

- `getApplication()`-> retourne l'application actuellement active
- `setApplication()`-> Modifie l'application active
- `get(appId: String): Application` -> retourne l'application correspondante à l'appId fourni

**Cas d'usage** :

- scripts multi-applications ;
- routage logique métier selon l'application ;
- comportement dépendant du contexte applicatif.

## 3.3. `$Api` : façade endpoint

`$Api` regroupe les fonctions utilitaires pour appeler un autre endpoint, local ou distant.

Méthodes disponibles :

- `get(apiPath: String) : Response`
- `post(apiPath: String, body: String) : Response`
- `put(apiPath: String, body: String) : Response`
- `delete(apiPath: String) : Response`
- `setApp(app: Application) : void` — change l'application cible pour les appels locaux
- `getApp() : Application`
- `setHeaders(headers: Map<String, String>) : void` — en-têtes ajoutés à chaque appel (local et distant)
- `getHeaders() : Map<String, String>`
- `buildParameters(path: String) : Map<String, String>` — parse les paramètres de query string d'une URL

**Comportement de résolution local / distant**

Pour chaque appel, `$Api` tente d'abord de résoudre l'endpoint **localement** (dans l'application active).
Si l'endpoint local est introuvable _et_ que `apiPath` est une URL HTTP/HTTPS complète valide (host + port + path), l'appel est automatiquement redirigé vers le service distant.

**Appel local**

```javascript
var userId = request.getPathVariables()["userId"];
$Api.setApp($App.get("app1"));
var r = $Api.get(`/orders/${userId}`);
```

**Appel distant**

```javascript
var r = $Api.post(
  `http://remoteaddr.com:8080/orders`,
  JSON.stringify({ userId: userId, amount: 100 })
);
```

**Propagation du `X-Correlation-Id` (bonne pratique)**

Lorsqu'un script appelle `$Api`, il est recommandé de propager l'identifiant de corrélation de la requête entrante :

```javascript
$Api.setHeaders({
  "X-Correlation-Id": request.getHeader("X-Correlation-Id"),
  "X-Tenant-Id": request.getHeader("X-Tenant-Id")
});
var r = $Api.get(`/payments/${paymentId}`);
```

Cela maintient la traçabilité de bout en bout sur toute la chaîne d'appels.

## 3.4. `$Principal` : principal authentifié

`$Principal` est injecté automatiquement dans le script et contient l'utilisateur authentifié courant.
Si la requête n'est pas authentifiée, sa valeur est `null`.

Note d'alignement:

- Le principal est construit côté backend depuis le profil OIDC de session (`SecurityService.getCurrentPrincipal(...)`).
- Pour les scripts API dynamiques, `$Principal` reste la source canonique côté serveur (pas de dépendance aux cookies UI).

Champs généralement disponibles :

- `id`
- `username`
- `name`
- `email`
- `roles`
- `preferred_username` (quand présent dans le profil OIDC)

Exemple :

```javascript
if ($Principal) {
  var username = $Principal.username;
  var roles = $Principal.roles || [];
}
```

## 3.5. `$Queue` : publication vers le bus de messages

`$Queue` est injecté **uniquement si** un `IQueueClient` est enregistré dans la configuration PayOS (`PayOSConfig.setQueueClient(...)`). Si aucun client n'est configuré, `$Queue` est absent du contexte JS — tester sa présence avant usage.

Méthodes disponibles dans les scripts JS :

- `publish(message: String) : void` — publie un message sur le topic configuré
- `isConnected() : boolean` — vérifie si la connexion au broker est active

**Cas d'usage typiques** :
- Publier un événement métier après une opération (paiement créé, statut changé)
- Envoyer une notification asynchrone
- Alimenter un pipeline d'analytics ou d'audit externe

**Exemple**

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();

  var saved = $DB.save("Payment", {
    amount: body?.amount,
    status: "PENDING"
  });

  if ($Queue && $Queue.isConnected()) {
    $Queue.publish(JSON.stringify({
      event: "payment.created",
      tenantId: request.getHeader("X-Tenant-Id"),
      correlationId: request.getHeader("X-Correlation-Id"),
      paymentId: saved
    }));
  }

  $Response.setJsonBody({ status: "OK", id: saved });
  $Response.setStatusCode(201);
  return $Response;
}
```

> **Note** : `publish` est synchrone du point de vue du script. L'envoi effectif vers le broker peut être asynchrone selon l'implémentation (`queue-service-nats` ou autre). Ne pas supposer que le message est consommé avant le retour de la réponse HTTP.

## 3.6. `$WebHooks` : notifications sortantes (webhooks)

`$WebHooks` est **toujours injecté** dans le contexte d'exécution lorsque le système de webhooks est actif. Si aucun abonné n'est configuré, `hasSubscribers()` retourne `false` — il n'est pas nécessaire de tester `$WebHooks != null`.

Méthodes disponibles dans les scripts JS :

- `emit(eventName: String, payload: Object) : void` — émet un événement webhook vers tous les abonnés correspondants (URL, filtres, secret HMAC)
- `hasSubscribers(eventName: String) : boolean` — vérifie si au moins un abonné est configuré pour cet événement

**Noms d'événements custom** : tout nom de chaîne est valide pour `emit()`. Aucune restriction ne s'applique aux noms d'événements custom (`native=false`). La contrainte sur les noms d'événements système (`api.*`, `page.*`, etc.) s'applique **uniquement** aux entrées `native=true` dans `webhooks.json`, validées au chargement.

**Règle de déduplication** : si le script émet un nom d'événement qui correspond à un événement système natif (ex. `api.post-request`), le kernel supprimera son propre dispatch automatique pour les entrées dont le filtre complet (chemin + méthode + codes de statut) correspond à la requête/réponse courante. Les entrées dont le filtre ne correspond pas sont envoyées normalement.

**Cas d'usage typiques** :
- Notifier un système externe après une opération métier (paiement créé, statut changé)
- Déclencher un audit log externe
- Alimenter un bus d'intégration tiers (ERP, CRM, BI)

**Exemple**

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();

  var saved = $DB.save("Payment", {
    amount: body?.amount,
    status: "COMPLETED"
  });

  if ($WebHooks.hasSubscribers('payment.completed')) {
    $WebHooks.emit('payment.completed', {
      transactionId: saved,
      amount: body?.amount,
      tenantId: request.getHeader("X-Tenant-Id"),
      correlationId: request.getHeader("X-Correlation-Id")
    });
  }

  $Response.setJsonBody({ status: "OK", id: saved });
  $Response.setStatusCode(201);
  return $Response;
}
```

> **Note** : `emit` est non-bloquant du point de vue du script. La livraison HTTP est effectuée de façon asynchrone (threads virtuels). Le script n'attend pas la confirmation de livraison avant de retourner la réponse.

> **Configuration** : les webhooks sont déclarés dans `webhooks.json` à la racine du répertoire de l'application. Voir [hooks-webhooks.md](../../../docs/architects/hooks-webhooks.md) pour les détails de configuration, les événements système, et les options de retry/signature.

## 3.6.1. `$I18n` : localisation côté serveur

`$I18n` est injecté automatiquement dans les scripts API et les hooks. Il permet de produire des messages localisés à partir de `config/i18n.json` et des fichiers `i18n/{locale}/*.json` de l'application.

Méthodes principales :

- `$I18n.locale() : String` — retourne la locale résolue pour la requête courante.
- `$I18n.t(key: String) : String` — traduit une clé simple.
- `$I18n.t(key: String, params: Object) : String` — traduit et remplace les tokens `{name}`.
- `$I18n.t(key: String, params: Object, locale: String) : String` — force une locale pour cet appel.
- `$I18n.exists(key: String) : boolean` — vérifie si une clé existe dans les bundles résolus.
- `$I18n.withLocale(locale: String)` — crée un proxy localisé pour plusieurs appels.

Exemple avec plusieurs paramètres :

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();
  var message = $I18n.t("orders.status", {
    orderId: body?.orderId,
    status: body?.status
  });

  $Response.setStatusCode(200);
  $Response.setJsonBody({ message: message, locale: $I18n.locale() });
  return $Response;
}
```

Avec la traduction suivante :

```json
{
  "orders": {
    "status": "Commande {orderId} au statut {status}"
  }
}
```

La locale est résolue dans l'ordre suivant : locale explicite, header d'override (`X-Locale` par défaut), champs du principal (`locale`, `language`, `preferredLocale`), `Accept-Language`, `defaultLocale`, `fallbackLocale`, puis `en`.

> **Guide complet** : voir [server-side-i18n-js-guide.md](server-side-i18n-js-guide.md) pour la structure des fichiers, l'héritage via `extends`, les fallbacks et les modes de clé manquante.

## 3.7. `$Errors` : erreurs métier standardisées

`$Errors` permet à un script API de lever une erreur métier avec un code fonctionnel, un message, un statut HTTP et des détails optionnels.

Méthodes disponibles :

- `business(code: String, message: String) : void` — lève une erreur métier avec le statut `400`.
- `business(code: String, message: String, statusCode: int) : void` — lève une erreur métier avec le statut fourni.
- `business(code: String, message: String, statusCode: int, details: Object) : void` — ajoute des détails métier sérialisés en JSON.
- `badRequest(code: String, message: String) : void` — raccourci pour `400`.
- `conflict(code: String, message: String) : void` — raccourci pour `409`.
- `notFound(code: String, message: String) : void` — raccourci pour `404`.

Exemple :

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();
  var balance = controlData.balance;

  if (balance < body.amount) {
    $Errors.business("INSUFFICIENT_FUNDS", "Solde insuffisant", 422, {
      available: balance,
      requested: body.amount
    });
  }

  $Response.setStatusCode(200);
  $Response.setJsonBody({ status: "ACCEPTED" });
  return $Response;
}
```

Réponse par défaut générée par PayOS :

```json
{
  "error": "Business error",
  "code": "INSUFFICIENT_FUNDS",
  "message": "Solde insuffisant",
  "details": {
    "available": 100,
    "requested": 250
  }
}
```

Si un hook `hooks/on-error.js` existe, il peut personnaliser la réponse. Sinon, PayOS applique automatiquement le mapping JSON ci-dessus. Les statuts hors plage `400..599` sont normalisés en `400`, et un code vide devient `BUSINESS_ERROR`.

---

## 3.8. En-têtes standards et traçabilité

Deux en-têtes sont reconnus nativement par le kernel PayOS :

| En-tête | Rôle |
|---|---|
| `X-Tenant-Id` | Identifie le tenant de la requête. Utilisé automatiquement par `$DB` pour l'isolation des données. |
| `X-Correlation-Id` | Identifiant de corrélation pour la traçabilité distribuée. Propagé automatiquement dans les logs d'audit. |

**Comportement automatique (transparent pour le script)** :
- `X-Tenant-Id` est lu par le kernel et injecté dans `$DB` → le script n'a pas besoin de passer le tenant à `$DB` explicitement.
- `X-Correlation-Id` est inclus dans l'entrée d'audit PCI-DSS générée automatiquement à la fin de chaque exécution.

**Bonne pratique** : lire ces en-têtes dans le script et les propager aux appels sortants (`$Api`, `$Queue`) pour maintenir la traçabilité de bout en bout :

```javascript
var correlationId = request.getHeader("X-Correlation-Id");
var tenantId = request.getHeader("X-Tenant-Id");

$Api.setHeaders({ "X-Correlation-Id": correlationId, "X-Tenant-Id": tenantId });

if ($Queue && $Queue.isConnected()) {
  $Queue.publish(JSON.stringify({ correlationId: correlationId, event: "...", tenantId: tenantId }));
}
```

## 3.9. `$Tenant` : tenant résolu

`$Tenant` est injecté automatiquement dans le script et contient l'identifiant du tenant courant, résolu par le kernel à partir du header `X-Tenant-Id`, du contexte de la requête, ou du MDC.
Sa valeur est `null` si aucun tenant n'est identifiable pour la requête.

```javascript
if ($Tenant) {
  $Logger.info("Processing request for tenant: {}", $Tenant);
}
```

Utilisations typiques :
- Journaliser des informations de traçabilité sans lire manuellement le header ;
- Conditionner la logique métier selon le tenant (en complément de `$DB` qui l'injecte automatiquement).

## 3.10. `$Logger` : journalisation depuis les scripts

`$Logger` est un logger SLF4J (`org.slf4j.Logger`) injecté dans le contexte JS. Il permet d'écrire des logs structurés depuis les scripts, en bénéficiant des tags MDC du kernel (tenant, correlation, appId).

Méthodes disponibles :

- `$Logger.info(message, ...args)`
- `$Logger.warn(message, ...args)`
- `$Logger.error(message, ...args)`
- `$Logger.debug(message, ...args)`

Les paramètres utilisent la syntaxe de substitution SLF4J (`{}`), comme en Java.

**Exemple :**

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();
  $Logger.info("Payment received: amount={}, tenant={}", body?.amount, $Tenant);

  try {
    var result = $DB.save("Payment", { amount: body?.amount });
    $Logger.info("Payment saved with id={}", result);
    $Response.setJsonBody({ id: result });
    $Response.setStatusCode(201);
  } catch (e) {
    $Logger.error("Failed to save payment: {}", e.message);
    $Response.setStatusCode(500);
    $Response.setBody("Internal error");
  }

  return $Response;
}
```

> **Bonne pratique** : préférer `$Logger` à `print()` ou `console.log()` pour que les logs soient intégrés au système de logging structuré du kernel et corrélés avec le tenant/correlation ID.

## 3.11. `$Library` : bibliothèques JavaScript partagées

`$Library` est injecté automatiquement dans chaque script API. Il permet de charger et d'exécuter des bibliothèques JavaScript partagées stockées dans le répertoire `lib/` de l'application.

Méthode disponible :

- `$Library.load(name: String) : Object` — charge et évalue la bibliothèque `lib/<name>.js`. Le résultat (module exporté) est mis en cache pour la durée du contexte d'exécution : plusieurs appels à `load` avec le même nom retournent le même objet sans re-lecture du fichier.

**Structure du répertoire bibliothèques** :

```
{app-base-path}/
  lib/
    utils.js
    validators.js
```

**Format d'un fichier bibliothèque** (`lib/utils.js`) :

```javascript
var exports = {};

exports.formatAmount = function(amount, currency) {
  return amount.toFixed(2) + ' ' + currency;
};

exports.isValidIban = function(iban) {
  return iban && iban.length >= 15;
};

exports; // la bibliothèque doit retourner l'objet exports
```

**Utilisation dans un script API** :

```javascript
execute(request, controlData) {
  var utils = $Library.load("utils");
  var validators = $Library.load("validators");

  var body = request.getJsonBody();

  if (!validators.isValidIban(body?.iban)) {
    $Response.setStatusCode(400);
    $Response.setJsonBody({ error: "Invalid IBAN" });
    return $Response;
  }

  var formatted = utils.formatAmount(body?.amount, "EUR");
  $Logger.info("Processing payment: {}", formatted);

  $Response.setJsonBody({ status: "OK", display: formatted });
  return $Response;
}
```

> **Chiffrement** : les fichiers `lib/*.js` peuvent être chiffrés (même convention que les scripts API). `LibraryLoader` applique `CryptoService.decryptIfEncrypted` à chaque lecture.
>
> **Cache** : `LibraryLoader` maintient un cache par chemin de fichier avec invalidation par `lastModified`. Le hot-reload du kernel invalide automatiquement ce cache lors d'un rechargement de config.

## 3.12. `$HookChain`, `$Error` et `$Page` — disponibles dans les scripts de hooks

Les fichiers de hooks (`hooks/*.js`) reçoivent les mêmes bindings que les scripts API standards, plus les objets suivants selon le point du cycle de vie :

### `$HookChain` — contrôle du pipeline

`$HookChain` est injecté dans **tous** les scripts de hooks. Il permet de contrôler la progression dans la chaîne d'héritage (via `extends` dans `app.json`) :

- `$HookChain.proceed()` — délègue l'exécution au hook parent dans la chaîne `extends` (ou au script API principal pour `pre-request.js`). À appeler explicitement si l'on souhaite que l'application parente soit exécutée.
- `$HookChain.stop()` — interrompt le pipeline immédiatement ; retourne la valeur courante de `$Response` au client sans poursuivre.

### `$Error` — exception déclenchante

`$Error` est injecté **uniquement** dans les scripts `on-error.js` et `page-on-error.js`. Il contient l'exception Java qui a déclenché le hook d'erreur.

```javascript
// hooks/on-error.js
$Logger.error("API error: {}", $Error.getMessage());
$Response.setStatusCode(500);
$Response.setJsonBody({ error: "Internal error", correlationId: request.getHeader("X-Correlation-Id") });
$HookChain.stop();
```

### `$Page` — contexte de la page Vue (hooks de page uniquement)

`$Page` est injecté **uniquement** dans les scripts de hooks de page (`page-pre-serve.js`, `page-post-serve.js`, `page-on-error.js`). Il expose :

- `$Page.getRequestPath() : String` — chemin de la page demandée (ex. `/dashboard`)
- `$Page.getProps() : Map<String, Object>` — props résolus (route + query params)

> **Note** : `$Principal` n'est **pas** disponible dans les hooks de page. Les pages Vue sont des ressources servies publiquement ; l'authentification est gérée côté client.

### Tableau récapitulatif des hooks disponibles

| Fichier hook | Point du cycle | `$HookChain` | `$Error` | `$Page` | `$Principal` |
|---|---|:---:|:---:|:---:|:---:|
| `hooks/pre-request.js` | Avant le script API | ✓ | — | — | ✓ |
| `hooks/post-request.js` | Après le script API | ✓ | — | — | ✓ |
| `hooks/on-error.js` | Exception API | ✓ | ✓ | — | ✓ |
| `hooks/page-pre-serve.js` | Avant assemblage Vue | ✓ | — | ✓ | — |
| `hooks/page-post-serve.js` | Après assemblage Vue | ✓ | — | ✓ | — |
| `hooks/page-on-error.js` | Exception page | ✓ | ✓ | ✓ | — |

> **Référence complète** : voir [hooks-webhooks.md](../../../docs/architects/hooks-webhooks.md) pour la description complète du moteur de hooks, la résolution par héritage d'application, et les exemples avancés.

## 4. Fonctions du cycle de vie d'exécution

### 4.1. Collecte des données de contrôle avec `loadControlData`

#### Signature

`loadControlData(request: Request) : Json`

où 
- **request**: Objet Java de type Request
- **Json**: valeur de retour sous format Json (controlData)

#### Description

La fonction `loadControlData` est exécutée en premier, avant `execute`.
Son rôle est de préparer toutes les données de configuration / paramètres nécessaires au traitement métier, afin d'éviter de recalculer systématiquement ces informations plusieurs fois dans `execute`.

Utilisations recommandées :

- charger des paramètres métier (seuils, frais, mapping) ;
- préparer des données de référence (configuration, dictionnaires) ;
- centraliser les valeurs de contexte réutilisées dans plusieurs blocs de logique.

Contrat recommandé :

- Entrée : la requête
- Sortie : un objet '*controlData*' JSON simple, sérialisable, qui est passé par le kernel à la fonction `execute(request, controlData)`.

Bonnes pratiques :

- Garder cette fonction rapide (pas de traitement lourds inutiles) ;
- Retourner un objet plat pré-préparé et explicite ;
- Ne pas modifier la réponse HTTP dans cette étape.

Exemple :

```javascript
loadControlData(request) {
  // read data from the database or config files, numbers that are hardcoded here are just for simplicity
  return {
    feesPerMerchant: request.getJsonBody()?.fees || 0.0,
    acquirerFees: 0.6,
    maxRetries: 3,
    timeoutMs: 1500,
  };
}
```

Puis dans `execute` :

```javascript
execute(request, controlData) {
  var amount = request.getJsonBody()?.amount || 0.0;
  var totalFees = amount * (controlData.feesPerMerchant + controlData.acquirerFees);

  $Response.setJsonBody({
    status: "OK",
    amount: amount,
    totalFees: totalFees,
  });
  $Response.statusCode = 200;
  return $Response;
}
```
### 4.2. Exécution de la logique métier du Endpoint avec `execute`

#### Signature

`execute(request: Request, controlData: Json) : Response`

où
- request: Objet Java de type Request
- controlData: Objet JSON préparé par `loadControlData`
- Response: Objet Java de type `Response`

**NB**: ***L'accès aux attributs des objets Java se fait via les getters et setters et pas comme attribut Json***

```Javascript
var body = $Response.getBody(); // correct
var falseBody = $Response.body // incorrect
```

#### Description

La fonction `execute` contient la logique métier principale de l'endpoint. Elle reçoit la requête courante ainsi que `controlData` pré-calculé, applique les règles métier, puis construit et retourne la réponse.

Utilisations recommandées :

- valider les entrées fonctionnelles (payload, paramètres, path variables) ;
- exécuter les traitements métier principaux ;
- appeler `$DB`, `$Api` ou `$App` selon le besoin ;
- remplir `$Response` (body, statusCode, headers) avant retour.

Contrat :

- Entrée : `request` + `controlData`
- Sortie : un objet de type `Response` prêt à être renvoyé au client.

Bonnes pratiques :

- gérer explicitement les cas d'erreur métier/technique (définir une manière unifiée de gérer les erreurs);
- utiliser `controlData` pour éviter les recalculs inutiles ;
- encadrer les opérations DB multi-étapes avec transaction manuelle si nécessaire (utilisation de `autoCommit`);
- toujours fixer un `statusCode` cohérent avec le résultat.

Exemple :

```javascript
execute(request, controlData) {
  var body = request.getJsonBody();
  var amount = body?.amount || 0.0;

  if (amount <= 0) {
    $Response.setJsonBody({
      status: "ERROR",
      code: "VALIDATION_ERROR",
      message: "amount must be > 0",
    });
    $Response.statusCode = 400;
    return $Response;
  }

  var totalFees = amount * (controlData.feesPerMerchant + controlData.acquirerFees);
  var netAmount = amount - totalFees;

  $Response.setJsonBody({
    status: "OK",
    amount: amount,
    fees: totalFees,
    netAmount: netAmount,
  });
  $Response.statusCode = 200;
  return $Response;
}
```

### 4.3. Observabilité avec `emitInsight`

#### Signature 

`emitInsight(request: Request, response: Response, payload: Json) : Json`

où:

- request: Objet Java de type `Request`
- response: Objet Java de type `Response`
- payload: payload de la réponse de type `Json`

#### Description

`emitInsight` sert à publier des informations Analytics ou des évènements métiers pertinents :

- durée d'exécution ;
- type de requête ;
- résultat (`SUCCESS`, `ERROR`) ;
- métadonnées tenant/application.

Exemple :

```javascript
emitInsight(input, output) {
  return {
    event: "payment.api.process",
    tenant: $Tenant,
    application: $App.getApplication ? $App.getApplication() : undefined,
    status: output?.status,
    timestamp: Date.now(),
  };
}
```

## 5. Sécurité et multi-tenant

- Les accès à la base de données sont par défaut tenant-aware via `$DB`.
- Le `tenantId` est injecté automatiquement en lecture et en écriture.
- Le principal authentifié est accessible dans les scripts via `$Principal`.

**Audit automatique (PCI-DSS Req 10.2.1)**

Chaque exécution de script est automatiquement tracée par le kernel, **sans action côté script**. L'entrée d'audit contient :
- `userId` (depuis `$Principal`, ou `"anonymous"`)
- `tenantId` (résolu depuis `X-Tenant-Id` ou MDC)
- `correlationId` (depuis l'en-tête `X-Correlation-Id`)
- chemin de l'endpoint
- code de statut HTTP de la réponse

L'implémentation par défaut écrit ces événements via SLF4J/logback. Elle peut être remplacée par une implémentation custom (ex. publication NATS) via `AuditLogger.setInstance(impl)` au bootstrap.

**Bonnes pratiques** :

- Ne jamais faire confiance à une valeur client sans validation
- Éviter d'exposer des détails internes dans les erreurs (stack traces, noms de tables, etc.)
- Propager `X-Correlation-Id` et `X-Tenant-Id` dans tous les appels sortants (`$Api`, `$Queue`)
- Vérifier `$Queue != null && $Queue.isConnected()` avant tout appel de publication

## 6. Checklist de développement

- Les trois méthodes sont présentes ;
- Les accès DB se font via $DB (qui est automatiquement tenant aware) ;
- Les transactions sont correctement fermées ;
- Le flag `autoCommit` est remis à `true` systématiquement (erreur ou pas) à la fin du script si jamais il a été mis à `false` au début du script.
- La gestion des erreurs est normalisée (à définir);

## 7. Template exemple prêt à copier

```javascript

  loadControlData(request) {
    return {
      feesPerMerchant: request.getBody()?.fees || 0.0,
      acquirerFees: 0.6
    };
  }

  execute(request, controlData) {
    if (!request?.body) {
      const e = new Error("Invalid payload");
      e.code = "VALIDATION_ERROR";
      throw e;
    }

    $Response.setJsonBody(request.getJsonBody());
    $Response.statusCode = 200;

    return $Response;
  },

  emitInsight(input, output) {
    return {
      event: "payos.api.handler",
      tenant: $Tenant,
      status: output?.status || "UNKNOWN",
      at: new Date().toISOString(),
    };
  }
```
