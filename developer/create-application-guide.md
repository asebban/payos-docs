# Guide — Créer une application PayOS

> Ce guide couvre la création d'une application PayOS de zéro : structure de fichiers, configuration, endpoints API JavaScript, pages Vue, localisation côté serveur, et intégration avec la base de données et les librairies partagées.
>
> Prérequis : un runtime PayOS opérationnel avec un `bootstrap.json` existant.

---

## Table des matières

1. [Concepts de base](#1-concepts-de-base)
2. [Structure de fichiers d'une application](#2-structure-de-fichiers-dune-application)
3. [Scaffolding rapide](#3-scaffolding-rapide)
4. [Déclarer l'application dans bootstrap.json](#4-déclarer-lapplication-dans-bootstrapjson)
5. [Configurer les routes et mappings API](#5-configurer-les-routes-et-mappings-api)
6. [Écrire un endpoint API JavaScript](#6-écrire-un-endpoint-api-javascript)
7. [Écrire une page Vue](#7-écrire-une-page-vue)
7.3. [Servir des fichiers statiques](#73-servir-des-fichiers-statiques)
7.5. [Hooks et Webhooks](#75-hooks-et-webhooks)
7.6. [Localiser les réponses côté serveur (`$I18n`)](#76-localiser-les-réponses-côté-serveur-i18n)
8. [Partager du code avec les librairies (`lib/`)](#8-partager-du-code-avec-les-librairies-lib)
9. [Accéder à la base de données](#9-accéder-à-la-base-de-données)
10. [Sécuriser des endpoints](#10-sécuriser-des-endpoints)
11. [Héritage d'applications (`extends`)](#11-héritage-dapplications-extends)
12. [Exemple complet](#12-exemple-complet)

---

## 1. Concepts de base

Une application PayOS est un **dossier** contenant des fichiers de configuration et des ressources (scripts JS, pages Vue, librairies). Le runtime la charge à partir du chemin `base.path` déclaré dans `bootstrap.json`.

**Types de ressources :**

| Type | Dossier | Description |
|------|---------|-------------|
| Configuration | `config/` | Déclaration des mappings API et des routes page, extensions (héritage) d'applications, ... |
| Endpoint API | `api/` | Scripts JavaScript exposés en HTTP/TCP/Queue |
| Page | `page/` | Fichiers `.vue` ou `.html` servis comme pages |
| Composant | `component/` | Composants Vue réutilisables |
| Librairie | `lib/` | Modules JS partagés entre endpoints |
| Fichiers statiques | `files/` | Fichiers statiques (images, CSS, JS, PDF, etc.) |
| Localisation | `i18n/` + `config/i18n.json` | Messages localisés chargés par `$I18n` |
| Menu | `menu/` | Définition des entrées de menu |
| Hooks | `hooks/` | scripts JS des hooks |

---

## 2. Structure de fichiers d'une application

```
my-app/
├── manifest.json            ← utilisé par apm pour l'enregistrement dans bootstrap.json
├── config/
│   ├── application.json     ← métadonnées (extends, category…)
│   ├── i18n.json            ← configuration de localisation côté serveur
│   ├── mappings.json        ← mappings des endpoints API
│   └── routes.json          ← routes des pages Vue
├── api/
│   └── items/
│       ├── list.js          ← handler GET /items
│       ├── get.js           ← handler GET /items/{id}
│       └── create.js        ← handler POST /items
├── page/
│   ├── index.vue            ← page principale
│   └── home.vue             ← page d'accueil
├── component/
│   └── card.vue             ← composant réutilisable
├── lib/
│   └── utils.js             ← librairie JS partagée
├── files/
│   ├── logo.png             ← image statique
│   ├── styles.css           ← feuille de styles
│   ├── docs/
│   │   └── guide.pdf        ← documentation PDF
│   └── assets/
│       └── banner.jpg       ← ressources diverses
├── i18n/
│   ├── fr/
│   │   ├── common.json      ← messages français communs
│   │   └── orders.json      ← messages français du domaine commandes
│   └── en/
│       └── common.json      ← messages anglais
├── hooks/
│   ├── pre-request.js       ← avant le script API
│   ├── post-request.js      ← après le script API
│   ├── on-error.js          ← en cas d'exception API
│   ├── page-pre-serve.js    ← avant l'assemblage de la page Vue
│   ├── page-post-serve.js   ← après l'assemblage de la page Vue
│   └── page-on-error.js     ← en cas d'exception page
├── webhooks.json            ← abonnements aux événements sortants
└── menu/
    └── entries.json         ← entrées de navigation
```

---

## 3. Scaffolding rapide

Le runtime fournit un script pour générer la structure minimale d'une application :

**Linux / macOS :**
```bash
# Syntaxe positionnelle
./generate_app.sh my-app

# Avec répertoire de sortie
./generate_app.sh --app-id my-app --output /opt/payos/apps

# Avec template UI Nuxt embarqué
./generate_app.sh --app-id my-app --output /opt/payos/apps --template ui
```

**Windows (PowerShell) :**
```powershell
# Syntaxe positionnelle
.\generate_app.ps1 my-app

# Avec répertoire de sortie
.\generate_app.ps1 -AppId my-app -output C:\payos\apps

# Avec template UI Nuxt embarqué
.\generate_app.ps1 -AppId my-app -output C:\payos\apps -Template ui
```

Le script crée un dossier `<AppId>` dans le répertoire de base fourni (ou dans le répertoire courant par défaut).
Le mode `ui` est désormais entièrement autonome : tous les artefacts Nuxt générés sont intégrés directement dans les scripts, sans dépendance vers un template externe présent sur disque.

Fichiers générés :

| Fichier | Description |
|---------|-------------|
| `manifest.json` | Manifeste de l'application (id, version, base.path) ; utilisé par `apm` |
| `config/application.json` | Configuration de base avec `extends: ["default"]` |
| `config/mappings.json` | Mappings API exemples pour `/items` et `/items/{id}` |
| `config/routes.json` | Routes de pages : `/` → `index`, `/home` → `home` |
| `api/items/list.js` | Handler GET `/items` |
| `api/items/get.js` | Handler GET `/items/{id}` |
| `api/items/create.js` | Handler POST `/items` |
| `lib/utils.js` | Librairie JS partagée exemple |
| `page/index.vue` | Page principale |
| `page/home.vue` | Page d'accueil |
| `menu/entries.json` | Entrée de menu pointant vers `/home` |

En mode `ui`, ce socle est complété avec un frontend Nuxt : composants, plugins, assets publics, configuration Nuxt et `package.json` prêts pour `npm install` puis `npm run dev`. Les fichiers `config/routes.json` et `menu/entries.json` sont alors enrichis par le template UI.

---

## 4. Déclarer l'application dans bootstrap.json

Dans la section `applications` du `bootstrap.json`, ajoutez une entrée pour votre application :

```json
{
  "applications": [
    {
      "id": "my-app",
      "name": "My Application",
      "base.path": "/opt/payos/apps/my-app",
      "version": "1.0.0"
    }
  ]
}
```

> **Rechargement à chaud :** le runtime surveille les fichiers de configuration via `ConfigWatcher`. Il n'est pas nécessaire de redémarrer après une modification de `bootstrap.json`.

### Clés essentielles

| Clé | Requis | Description |
|-----|--------|-------------|
| `id` | Oui | Identifiant unique. Utilisé dans les URIs : `/{id}/api/...`, `/{id}/page/...` |
| `name` | Oui | Nom lisible |
| `base.path` | Non | Chemin absolu du dossier de l'application. Défaut : `./apps/{id}` |
| `version` | Non | Version de l'application |
| `extends` | Non | Liste d'IDs d'applications parentes (voir [section 11](#11-héritage-dapplications-extends)) |

---

## 5. Configurer les routes et mappings API

### 5.1 `config/mappings.json` — Endpoints API

Déclare les endpoints HTTP de l'application. Chaque clé est un template URI.

```json
{
  "mappings": {
    "api": {
      "/payments": {
        "GET": {
          "handler": "payments/list",
          "description": "Liste des paiements"
        },
        "POST": {
          "handler": "payments/create",
          "description": "Créer un paiement",
          "roles": ["ROLE_USER"]
        }
      },
      "/payments/{id}": {
        "GET": {
          "handler": "payments/get",
          "description": "Détail d'un paiement"
        },
        "DELETE": {
          "handler": "payments/delete",
          "roles": ["ROLE_ADMIN"]
        }
      }
    }
  }
}
```

- La valeur de `handler` est le **chemin relatif** du fichier JS dans `api/`, sans l'extension `.js`.
- Les variables de chemin (`{id}`) sont accessibles dans le script via `request.getPathVariables()`.
- `roles` est optionnel. Si absent, l'endpoint est public. Si présent, une authentification OIDC est requise.

### 5.2 `config/routes.json` — Routes de pages

Déclare les routes de navigation pour les pages Vue.

```json
{
  "routes": [
    { "path": "/",        "component": "index"          },
    { "path": "/home",    "component": "home"           },
    { "path": "/payments","component": "payments/list"  },
    { "path": "/profile", "component": "profile/detail" }
  ]
}
```

La valeur de `component` est le chemin relatif du fichier `.vue` dans `page/`, sans extension.

> **Limitation — pas de path variables dans les routes de pages :** le routeur de pages effectue une **comparaison exacte** de l'URL (`Map.get(path)`). Il ne supporte pas les segments dynamiques du type `:id` ou `{id}`. Pour afficher un détail avec un identifiant, passez-le en **query parameter** (`/payments/detail?id=123`) ou chargez les données depuis le composant Vue via un appel à l'endpoint API correspondant.

### 5.3 `config/application.json` — Métadonnées

```json
{
  "extends": ["default"]
}
```

Ce fichier permet de déclarer les applications parentes (voir [section 11](#11-héritage-dapplications-extends)). Il peut aussi contenir tout bloc de configuration spécifique à l'application (sécurité, base de données, etc.).

---

## 6. Écrire un endpoint API JavaScript

Chaque fichier dans `api/` expose obligatoirement **trois fonctions** :

```javascript
// api/payments/list.js

function loadControlData(request) {
    // Chargement des données de contrôle (appelé avant execute)
    // Retourner un objet plat Map<String,Object> ou null
    return {
        tenantId: request.getHeader("X-Tenant-Id")
    };
}

function execute(request, controlData) {
    // Logique principale de l'endpoint
    // Doit retourner : $Response, Map<String,Object>, ou List<Map<String,Object>>
    
    var params = $DB.newParams();
    var payments = $DB.findAll("Payment");
    $Response.setJsonBody(payments);
    
    return $Response;
}

function emitInsight(request, response, payload) {
    // Observabilité post-traitement (telemetry, audit, événements)
    // Appelé après execute, même en cas d'erreur
    // La valeur de retour est actuellement ignorée
    return null;
}
```

### Objets globaux disponibles dans les scripts

| Binding | Type | Description |
|---------|------|-------------|
| `$Response` | `Response` | Objet réponse HTTP |
| `$Api` | `ApiProxy` | Appel d'autres endpoints API de l'application |
| `$App` | `AppProxy` | Accès au contexte applicatif |
| `$Principal` | `Map` | Informations de l'utilisateur authentifié (`id`, `email`, rôles…) |
| `$Tenant` | `String` | Identifiant du tenant courant |
| `$Logger` | `Logger` | Logger SLF4J (`info`, `warn`, `error`) |
| `$DB` | `IDatabaseService` | Accès base de données (si configuré) |
| `$Queue` | `IQueueClient` | Client de messagerie (si configuré) |
| `$Library` | `LibraryProxy` | Chargement de librairies JS partagées (voir [section 8](#8-partager-du-code-avec-les-librairies-lib)) |
| `$WebHooks` | `WebhookHooksProxy` | Émission d'événements webhooks sortants (voir [section 7.5](#75-hooks-et-webhooks)) |
| `$HookChain` | `HookChainProxy` | Contrôle du pipeline dans les scripts de hooks (proceed / stop) |

### Variables de chemin

```javascript
function execute(request, controlData) {
    var id = request.getPathVariables()["id"];
    var params = $DB.newParams();
    params.put("id", id);
    var payment = $DB.unique("select p from Payment p where p.id = :id", params);
    if (payment == null) {
        $Response.setStatusCode(404);
        return $Response;
    }
    $Response.setJsonBody(payment);
    return $Response;
}
```

### Corps de requête JSON

```javascript
function execute(request, controlData) {
    var body = request.getJsonBody();
    var payment = $DB.save("Payment", {
        amount:   body.amount,
        currency: body.currency,
        status:   "PENDING"
    });
    $Response.setStatusCode(201);
    $Response.setJsonBody(payment);
    return $Response;
}
```

---

## 7. Écrire une page Vue

Les pages sont des fichiers `.vue` (Single File Component) placés dans `page/`.

```vue
<!-- page/payments/list.vue -->
<template>
  <div>
    <h1>Paiements</h1>
    <ul>
      <li v-for="p in payments" :key="p.id">{{ p.amount }} {{ p.currency }}</li>
    </ul>
  </div>
</template>

<script>
export default {
  data() {
    return { payments: [] };
  },
  async mounted() {
    const res = await fetch("/my-app/api/payments");
    this.payments = await res.json();
  }
}
</script>

<style scoped>
h1 { color: #2c3e50; }
</style>
```

> Les fichiers `.html` statiques sont également supportés dans `page/`.

---

## 7.3. Servir des fichiers statiques

Les fichiers statiques (images, CSS, JavaScript, PDF, archives, etc.) sont placés dans le dossier `files/` et accessibles via HTTP sans nécessiter de script ou de configuration supplémentaire.

### Structure et accès

```
my-app/
├── files/
│   ├── logo.png
│   ├── styles.css
│   ├── docs/
│   │   └── guide.pdf
│   └── assets/
│       └── banner.jpg
```

**Accès HTTP :**

```
GET /my-app/files/logo.png        → Retourne logo.png avec Content-Type: image/png
GET /my-app/files/styles.css      → Retourne styles.css avec Content-Type: text/css
GET /my-app/files/docs/guide.pdf  → Retourne guide.pdf avec Content-Type: application/pdf
GET /my-app/files/assets/banner.jpg → Retourne banner.jpg avec Content-Type: image/jpeg
```

### Types MIME supportés

Le runtime détecte automatiquement le type de contenu en fonction de l'extension du fichier :

| Catégorie | Extensions | Content-Type |
|-----------|------------|-------------|
| **Texte** | `.txt`, `.csv` | `text/plain`, `text/csv` |
| **HTML/Web** | `.html`, `.htm`, `.css`, `.js`, `.json`, `.xml` | `text/html`, `text/css`, `application/javascript`, etc. |
| **Images** | `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`, `.ico`, `.bmp` | `image/png`, `image/jpeg`, etc. |
| **Documents** | `.pdf`, `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx` | `application/pdf`, `application/msword`, etc. |
| **Archives** | `.zip`, `.tar`, `.gz`, `.7z`, `.rar` | `application/zip`, `application/x-tar`, etc. |
| **Audio** | `.mp3`, `.wav`, `.ogg`, `.m4a`, `.flac` | `audio/mpeg`, `audio/wav`, etc. |
| **Vidéo** | `.mp4`, `.avi`, `.mkv`, `.webm`, `.mov` | `video/mp4`, `video/x-msvideo`, etc. |
| **Polices** | `.woff`, `.woff2`, `.ttf`, `.otf`, `.eot` | `font/woff`, `font/woff2`, etc. |

### Fonctionnalités de cache HTTP

Le runtime implémente automatiquement les mécanismes de cache HTTP pour optimiser les performances :

- **Cache-Control:** `max-age=3600` (1 heure par défaut)
- **Last-Modified:** Date de dernière modification du fichier
- **ETag:** Empreinte basée sur timestamp et taille du fichier
- **304 Not Modified:** Retourné automatiquement si le fichier n'a pas été modifié depuis la dernière requête

**Exemple de requête avec cache :**

```http
# Première requête
GET /my-app/files/logo.png HTTP/1.1

HTTP/1.1 200 OK
Content-Type: image/png
Cache-Control: max-age=3600
Last-Modified: Tue, 07 Jun 2026 10:30:00 GMT
ETag: "1717754400000-4096"
Content-Length: 4096

# Requête suivante avec If-Modified-Since
GET /my-app/files/logo.png HTTP/1.1
If-Modified-Since: Tue, 07 Jun 2026 10:30:00 GMT

HTTP/1.1 304 Not Modified
Cache-Control: max-age=3600
Last-Modified: Tue, 07 Jun 2026 10:30:00 GMT
ETag: "1717754400000-4096"
```

### Sécurité

- **Protection contre directory traversal :** Les tentatives d'accès à des fichiers en dehors du dossier `files/` (ex: `../../../etc/passwd`) sont bloquées et retournent une erreur 404.
- **Décryptage automatique :** Les fichiers chiffrés avec l'en-tête magique PayOS (`P8OS`) sont automatiquement décryptés avant d'être servis.

### Utilisation depuis une page Vue

```vue
<template>
  <div>
    <img :src="logoUrl" alt="Logo" />
    <a :href="pdfUrl" target="_blank">Télécharger le guide</a>
  </div>
</template>

<script>
export default {
  data() {
    return {
      logoUrl: '/my-app/files/logo.png',
      pdfUrl: '/my-app/files/docs/guide.pdf'
    };
  }
}
</script>
```

### Utilisation depuis un endpoint API

```javascript
// api/config/get-assets.js
function execute(request) {
    var assets = {
        logo: "/" + $App.getId() + "/files/logo.png",
        styles: "/" + $App.getId() + "/files/styles.css",
        guide: "/" + $App.getId() + "/files/docs/guide.pdf"
    };
    
    $Response.setJsonBody(assets);
    return $Response;
}
```

### Limitations

- Les fichiers doivent résider physiquement dans `<app-base>/files/` ou ses sous-répertoires
- Pas de génération dynamique : pour servir du contenu généré à la volée, utilisez un endpoint API
- Pas de liste de répertoire : l'accès direct à un répertoire retourne 404

---

## 7.5. Hooks et Webhooks

### Hooks — scripts de cycle de vie

Les fichiers du répertoire `hooks/` sont chargés automatiquement par le kernel à chaque point du cycle de vie des requêtes API et des pages Vue. Aucune configuration supplémentaire n'est nécessaire : la présence du fichier suffit.

| Fichier hook | Point du cycle | Contexte |
|---|---|---|
| `hooks/pre-request.js` | Avant le script API | `$HookChain`, `$WebHooks`, `$Principal` |
| `hooks/post-request.js` | Après le script API | `$HookChain`, `$WebHooks`, `$Principal` |
| `hooks/on-error.js` | Exception API | `$HookChain`, `$WebHooks`, `$Error`, `$Principal` |
| `hooks/page-pre-serve.js` | Avant assemblage Vue | `$HookChain`, `$WebHooks`, `$Page` |
| `hooks/page-post-serve.js` | Après assemblage Vue | `$HookChain`, `$WebHooks`, `$Page` |
| `hooks/page-on-error.js` | Exception page | `$HookChain`, `$WebHooks`, `$Error`, `$Page` |

**Bindings spécifiques aux hooks :**
- `$HookChain.proceed()` — délègue au hook parent (`extends`) ou au script API principal ; appel explicite requis pour poursuivre.
- `$HookChain.stop()` — interrompt le pipeline et retourne `$Response` immédiatement.
- `$Error` — exception déclenchante (uniquement dans `on-error.js` et `page-on-error.js`).
- `$Page` — contexte de la page : `getRequestPath()` + `getProps()` (uniquement dans les hooks de page).

**Exemple — hook de validation pré-requête :**

```javascript
// hooks/pre-request.js
var body = request.getJsonBody();
if (!body || !body.amount) {
    $Response.setStatusCode(400);
    $Response.setJsonBody({ error: "Payload manquant ou incomplet" });
    $HookChain.stop();
    return;
}
$HookChain.proceed();
```

**Exemple — hook d'erreur avec notification webhook :**

```javascript
// hooks/on-error.js
$Logger.error("Erreur API : {}", $Error.getMessage());
if ($WebHooks.hasSubscribers('api.error.notified')) {
    $WebHooks.emit('api.error.notified', {
        path: request.getPath(),
        error: $Error.getMessage(),
        correlationId: request.getHeader("X-Correlation-Id")
    });
}
$Response.setStatusCode(500);
$Response.setJsonBody({ error: "Erreur interne" });
$HookChain.stop();
```

### Webhooks — notifications sortantes

Les webhooks permettent de notifier des systèmes externes de manière asynchrone (via HTTP POST signé HMAC-SHA256) lors d'événements applicatifs.

Ils sont déclarés dans `webhooks.json` à la racine de l'application :

```json
[
  {
    "id": "payment-created-notify",
    "event": "payment.created",
    "native": false,
    "url": "https://external.example.com/webhooks",
    "secret": "my-hmac-secret",
    "filter": {
      "path": "/payments",
      "method": "POST",
      "statusCodes": [201]
    },
    "retry": { "maxAttempts": 3, "backoffMs": 1000 }
  },
  {
    "id": "api-post-request-audit",
    "event": "api.post-request",
    "native": true,
    "url": "https://audit.example.com/events",
    "secret": "audit-secret"
  }
]
```

- `native: false` — déclenché explicitement par `$WebHooks.emit('payment.created', payload)` dans le script.
- `native: true` — déclenché automatiquement par le kernel après chaque requête satisfaisant le filtre. L'événement doit être un événement système reconnu (`api.*`, `page.*`).

> **Référence complète :** voir [hooks-webhooks.md](../../architects/hooks-webhooks.md) pour la liste des événements système, les règles de déduplication et la configuration de retry.

---

## 7.6. Localiser les réponses côté serveur (`$I18n`)

PayOS injecte `$I18n` dans les scripts API et les hooks. Les messages sont chargés depuis `config/i18n.json` et les dossiers `i18n/{locale}/` de l'application.

Exemple de configuration :

```json
{
  "defaultLocale": "fr",
  "fallbackLocale": "en",
  "supportedLocales": ["fr", "en", "ar-MA"],
  "headerName": "Accept-Language",
  "overrideHeaderName": "X-Locale",
  "missingKeyMode": "bracket"
}
```

Exemple de fichier `i18n/fr/orders.json` :

```json
{
  "orders": {
    "created": "Commande {orderId} créée pour {customerName}"
  }
}
```

Utilisation dans un endpoint :

```javascript
function execute(request, controlData) {
    var body = request.getJsonBody();
    var message = $I18n.t("orders.created", {
        orderId: body?.orderId,
        customerName: body?.customerName
    });

    $Response.setStatusCode(201);
    $Response.setJsonBody({ message: message, locale: $I18n.locale() });
    return $Response;
}
```

Les traductions suivent la chaîne `extends` : les ressources de l'application locale surchargent les clés héritées des applications parentes ou des capabilities actives.

> **Guide complet :** voir [server-side-i18n-js-guide.md](server-side-i18n-js-guide.md).

---

## 8. Partager du code avec les librairies (`lib/`)

Les librairies JS sont des modules réutilisables placés dans `lib/`. Elles sont chargées à la demande dans les endpoints via `$Library.load(name)`.

### Créer une librairie

```javascript
// lib/utils.js

function formatAmount(amount, currency) {
    return amount.toFixed(2) + " " + currency;
}

function validateIban(iban) {
    return iban != null && iban.length >= 15;
}

// La dernière expression est l'objet exporté (équivalent d'un return)
({ formatAmount: formatAmount, validateIban: validateIban })
```

### Utiliser une librairie dans un endpoint

```javascript
// api/payments/create.js

function execute(request, controlData) {
    var utils = $Library.load("utils");
    
    var body = request.getJsonBody();
    if (!utils.validateIban(body.iban)) {
        $Response.setStatusCode(400);
        $Response.setJsonBody({ error: "IBAN invalide" });
        return $Response;
    }
    
    var payment = $DB.save("Payment", { amount: body.amount, iban: body.iban });
    var r = { id: payment.id, formatted: utils.formatAmount(body.amount, body.currency) };
    $Response.setStatusCode(200);
    $Response.setJsonBody(r);
}
```

**Notes importantes :**
- `$Library.load(name)` est idempotent dans un même contexte de requête : le module est évalué une seule fois, les appels suivants retournent l'objet en cache.
- La résolution suit la chaîne `extends` : si `lib/utils.js` n'existe pas dans l'application courante, le runtime cherche dans les applications parentes.
- Le contenu des librairies est mis en cache disque par `lastModified` — le rechargement à chaud fonctionne automatiquement.

---

## 9. Accéder à la base de données

Pour utiliser `$DB`, l'application doit avoir un bloc `database-service` dans sa configuration (directement dans `bootstrap.json` ou dans `config/application.json`).

```json
{
  "database-service": {
    "name": "my_app_db",
    "mapping-files": ["model/payment.hbm.xml"],
    "configuration": {
      "url": "jdbc:postgresql://localhost:5432/myapp",
      "username": "myapp_user",
      "password": "secret",
      "dialect": "org.hibernate.dialect.PostgreSQLDialect",
      "ddl-auto": "validate"
    }
  }
}
```

### Opérations courantes

```javascript
// Lecture — liste paginée
var params = $DB.newParams();
params.put("status", "PENDING");
var list = $DB.list("select p from Payment p where p.status = :status", params, 0, 50);

// Lecture — objet unique
var single = $DB.unique("select p from Payment p where p.id = :id", params);

// Écriture
var created = $DB.save("Payment", { amount: 100, currency: "EUR", status: "PENDING" });

// Mise à jour
$DB.update("Payment", { id: created.id, status: "CONFIRMED" });

// Suppression
$DB.deleteById("Payment", created.id);
```

Voir le [Guide JavaScript des Endpoints API](./javascript-api-endpoint-guide.md) pour la référence complète de `$DB`.

---

## 10. Sécuriser des endpoints

### Par rôle dans les mappings

```json
{
  "mappings": {
    "api": {
      "/admin/payments": {
        "GET": {
          "handler": "admin/payments",
          "roles": ["ROLE_ADMIN"]
        }
      }
    }
  }
}
```

Si l'utilisateur n'est pas authentifié ou ne possède pas le rôle requis, le runtime retourne automatiquement `401` ou `403` sans exécuter le script.

### Accéder à l'identité dans le script

```javascript
function execute(request, controlData) {
    // $Principal est null si l'endpoint est public
    var userId   = $Principal != null ? $Principal["id"] : null;
    var userMail = $Principal != null ? $Principal["email"] : null;
    
    $Logger.info("Requête de l'utilisateur : {}", userId);
    // ...
}
```

### Configuration OIDC

La sécurité OIDC se configure dans le bloc `security` de l'application (dans `bootstrap.json` ou `config/application.json`) :

```json
{
  "security": {
    "provider": "nimbus",
    "clientId": "my-app-client",
    "clientSecret": "s3cr3t",
    "oidcProviderBaseUrl": "https://auth.example.com",
    "realm": "payos",
    "scope": "openid profile email"
  }
}
```

Voir la [Référence de configuration JSON](./json-configuration-reference.md#bloc-security-dune-application) pour toutes les clés disponibles.

---

## 11. Héritage d'applications (`extends`)

Une application peut hériter des ressources (API, pages, composants, librairies) d'autres applications via la clé `extends` :

```json
{
  "extends": ["common-capabilities", "default"]
}
```

**Comportement de résolution :**
1. Le runtime cherche la ressource dans l'application courante.
2. Si non trouvée, il parcourt la liste `extends` dans l'ordre, récursivement.
3. La première correspondance est retournée.

Cela permet de :
- Partager des librairies communes entre plusieurs applications.
- Surcharger un endpoint hérité en créant un fichier du même nom dans `api/`.
- Activer des capabilities métier optionnelles.

---

## 12. Exemple complet

### Structure

```
payment-app/
├── manifest.json
├── config/
│   ├── application.json
│   ├── mappings.json
│   └── routes.json
├── api/
│   └── payments/
│       ├── list.js
│       ├── get.js
│       └── create.js
├── page/
│   ├── index.vue
│   └── home.vue
└── lib/
    └── validation.js
```

### `config/application.json`
```json
{
  "extends": ["default"]
}
```

### `config/mappings.json`
```json
{
  "mappings": {
    "api": {
      "/payments": {
        "GET":  { "handler": "payments/list",   "roles": ["ROLE_USER"] },
        "POST": { "handler": "payments/create", "roles": ["ROLE_USER"] }
      },
      "/payments/{id}": {
        "GET":  { "handler": "payments/get",    "roles": ["ROLE_USER"] }
      }
    }
  }
}
```

### `lib/validation.js`
```javascript
function isPositiveAmount(amount) {
    return typeof amount === "number" && amount > 0;
}

function isSupportedCurrency(currency) {
    return ["EUR", "USD", "MAD"].indexOf(currency) !== -1;
}

({ isPositiveAmount: isPositiveAmount, isSupportedCurrency: isSupportedCurrency })
```

### `api/payments/create.js`
```javascript
function loadControlData(request) {
    return null;
}

function execute(request, controlData) {
    var validation = $Library.load("validation");
    var body = request.getJsonBody();

    if (!validation.isPositiveAmount(body.amount)) {
        $Response.setStatusCode(400);
        $Response.setJsonBody({ error: "Montant invalide" });
        return $Response;
    }

    if (!validation.isSupportedCurrency(body.currency)) {
        $Response.setStatusCode(400);
        $Response.setJsonBody({ error: "Devise non supportée" });
        return $Response;
    }

    var payment = $DB.save("Payment", {
        amount:   body.amount,
        currency: body.currency,
        status:   "PENDING"
    });

    $Logger.info("Paiement créé : {}", payment.id);
    $Response.setStatusCode(201);
    $Response.setJsonBody(payment);
    return $Response;
}

function emitInsight(request, response, payload) {
    return null;
}
```

### Déclaration dans `bootstrap.json`
```json
{
  "applications": [
    {
      "id": "payment-app",
      "name": "Payment Application",
      "base.path": "/opt/payos/apps/payment-app",
      "version": "1.0.0",
      "mapping-files": ["model/payment.hbm.xml"],
      "database-service": {
        "name": "payment_db",
        "configuration": {
          "url": "jdbc:postgresql://localhost:5432/payments",
          "username": "app_user",
          "password": "secret",
          "dialect": "org.hibernate.dialect.PostgreSQLDialect",
          "ddl-auto": "validate"
        }
      }
    }
  ]
}
```

### Accès aux endpoints

Une fois le runtime démarré, les endpoints sont accessibles aux URIs :

```
GET  http://localhost:8080/payment-app/api/payments
POST http://localhost:8080/payment-app/api/payments
GET  http://localhost:8080/payment-app/api/payments/123
GET  http://localhost:8080/payment-app/page/index
```

---

## Références

- [Référence complète de la configuration JSON](./json-configuration-reference.md)
- [Guide JavaScript des Endpoints API](./javascript-api-endpoint-guide.md)
- [Architecture PayOS](./architecture.md)
- [Guide de sécurité OIDC](./oidc-configuration-guide.md)
