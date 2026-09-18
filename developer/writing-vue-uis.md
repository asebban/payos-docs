# Guide — Créer une interface utilisateur Vue.js dans PayOS

Créé: 2026-09-17
Dernier alignement: 2026-09-17
Version: v1

> Ce guide couvre la création d'une UI Vue.js pour une application PayOS : structure d'une application UI, comment écrire et enregistrer une page appelable via `.../page/...`, comment créer un composant réutilisable asynchrone, et comment importer des composants Vue "normaux" (non asynchrones, au sens `import` classique). Pour la partie backend générale d'une application PayOS (endpoints API, `bootstrap.json`, base de données, sécurité), voir [Créer une application PayOS](create-application-guide.md), section 7 en particulier pour la déclaration de routes.
>
> Prérequis : un runtime PayOS opérationnel (voir [Getting started](getting-started.md)) et une application déjà déclarée dans `bootstrap.json`.

---

## Sommaire

1. [Vue d'ensemble et composants principaux](#1-vue-densemble-et-composants-principaux)
2. [Créer une page Vue appelable via `.../page/...`](#2-créer-une-page-vue-appelable-via-page)
3. [Créer un composant réutilisable asynchrone](#3-créer-un-composant-réutilisable-asynchrone)
4. [Importer des composants Vue "normaux" (non asynchrones)](#4-importer-des-composants-vue-normaux-non-asynchrones)
5. [Pour aller plus loin](#5-pour-aller-plus-loin)

---

## 1. Vue d'ensemble et composants principaux

Une UI PayOS n'a **aucune étape de build** (pas de Vite/Webpack, pas de `npm install`) : le navigateur exécute directement les fichiers servis par le runtime. Ça repose sur deux mécanismes de rendu bien distincts, qu'il faut absolument distinguer avant d'écrire quoi que ce soit :

| | Shell applicatif (`page/app/`) | Pages et composants "dynamiques" (`page/*.vue`, `component/*.vue`) |
|---|---|---|
| **Rôle** | Infrastructure de l'app : point de montage, menu, navigation, runtime JS | Le contenu métier : vos pages et vos composants réutilisables gérés côté backend |
| **Qui les compile** | Le **navigateur**, via `vue3-sfc-loader` (vrai parsing de fichier `.vue`) | Le **backend Java** : découpe par regex de `<template>`/`<script>`/`<style>`, renvoie du JSON |
| **`<script>`** | Vrais `import` ES natifs, Composition ou Options API | Pas d'`import` — variables injectées dans le scope (`loadPage`, `onMounted`, `createDynamicComponent`, ...), style Options API uniquement |
| **Exécuté via** | Compilation SFC réelle par `vue3-sfc-loader` | `new Function(...)` côté navigateur, après découpage côté serveur |

Ce guide couvre exclusivement la 2e colonne (c'est le travail quotidien pour ajouter une page ou un composant métier) et le mécanisme d'enregistrement global qui permet de faire le pont entre les deux (sections 3 et 4).

### Arborescence type d'une application UI

```
mon-app/
├── config/
│   ├── application.json     ← métadonnées, extends, defaultUrl
│   ├── routes.json          ← routes de pages (voir §2)
│   └── mappings.json        ← mappings API (voir writing-apis.md)
├── menu/
│   └── entries.json         ← items de menu (optionnel)
├── page/
│   ├── Home.vue              ← une page (gabarit backend)
│   ├── Dashboard.vue          ← une autre page
│   └── app/                  ← le shell applicatif (fourni par le template, rarement modifié)
│       ├── index.html
│       ├── vendor/            ← Vue et vue3-sfc-loader vendorisés
│       └── src/
│           ├── main.js        ← bootstrap : createApp, enregistrement global
│           ├── config.js      ← appBase, basePath, tenantId, defaultPage
│           ├── runtime.js     ← loadPage, createDynamicComponent, session
│           ├── AppRoot.vue    ← composant racine (compilé par vue3-sfc-loader)
│           └── components/
│               ├── AppMenu.vue
│               └── CustomLink.vue
└── component/
    └── MonWidget.vue          ← composant "dynamique" réutilisable (voir §3a)
```

### Cycle de démarrage, en une phrase

Le navigateur charge `page/index.html`, qui exécute `page/app/src/main.js` comme module ES natif. `main.js` utilise `vue3-sfc-loader` pour compiler `AppRoot.vue` (qui compile lui-même `AppMenu.vue`), crée l'app Vue, l'enregistre avec un runtime partagé (`provide('runtime', ...)`), puis monte le tout. `AppRoot` charge alors la page par défaut (`config.defaultPage`, `"home"` par défaut) via `loadPage("home")`. **Il n'y a pas de `vue-router`** : toute la navigation applicative passe par `loadPage(nom)`, jamais par l'URL du navigateur.

### Les composants principaux du shell

- **`AppRoot.vue`** — composant racine, affiche `<AppMenu />` et la page courante via `<component :is="currentPage" />`.
- **`AppMenu.vue`** — menu de navigation, alimenté par `GET /{appId}/menu` (source : `menu/entries.json`) ou par des `props` fournies.
- **`CustomLink.vue`** — composant global de navigation interne, utilisable dans n'importe quelle page (`<custom-link to="dashboard">...</custom-link>`), sans déclaration.
- **`runtime.js`** — le moteur : `loadPage(nom)`, `createDynamicComponent(nom)`, état de session (`$Principal` équivalent côté frontend), cache mémoire des pages/composants déjà chargés.
- **`config.js`** — configuration du frontend (`appBase` doit correspondre à l'`id` de l'application dans `bootstrap.json`, `basePath`, `tenantId`, `defaultPage`), surchargeable via `window.__APP_CONFIG__` dans `index.html`.

En pratique, vous ne modifiez ce shell que pour y enregistrer un nouveau composant réutilisable global (§3b/§4) — la suite de ce guide (§2, §3a) concerne le travail habituel : ajouter des pages et des composants métier.

---

## 2. Créer une page Vue appelable via `.../page/...`

Une page = un fichier `.vue` dans `page/` + une entrée dans `config/routes.json`. C'est ce qui répond côté serveur à `GET /{appId}/page/{nom}`.

### 2.1. Créer le fichier

`page/MaPage.vue` :

```html
<template>
  <div class="ma-page">
    <h1>{{ title }}</h1>
    <button @click="goHome">Retour à l'accueil</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // Valeurs par défaut — surchargées par les "props" de routes.json
      title: "Ma nouvelle page",
    };
  },
  methods: {
    goHome() {
      // loadPage est injecté automatiquement dans le scope du script —
      // pas d'import, pas de this.$root.loadPage.
      loadPage("home");
    },
  },
};
</script>

<style>
/* Pas de <style scoped> ici : l'attribut est toléré syntaxiquement mais
   n'a AUCUN effet — le backend extrait le CSS par simple regex et
   l'injecte globalement dans la page. Préfixez toujours vos classes
   (.ma-page plutôt que .container) pour éviter les collisions avec
   les autres pages/composants. */
.ma-page {
  padding: 20px;
}
</style>
```

**Contraintes du `<script>`**, parce qu'il est extrait côté serveur (regex, pas un vrai compilateur SFC) puis évalué côté navigateur via `new Function(...)` :

- **Pas d'`import`** — une instruction `import` provoquerait une erreur de syntaxe à l'exécution (`import` n'est légal qu'en tête de module ES, pas dans un corps de fonction). Utilisez uniquement les identifiants déjà injectés dans le scope : `useRuntimeLoader`, `useRuntimeConfig`, `onMounted`, `loadPage`, `goBack`, `createDynamicComponent`.
- Style **Options API** (`data()`, `methods`, `mounted()`, `setup()` si besoin) — pas de `<script setup>`.
- Pour utiliser un composant réutilisable dans le `<template>` : soit un composant **global** (§3b/§4, aucune déclaration nécessaire), soit un composant **dynamique backend** (§3a, à déclarer dans `components: {}` via `createDynamicComponent(...)`).

### 2.2. Déclarer la route

Dans `config/routes.json` :

```json
{
  "routes": [
    {
      "path": "/ma-page",
      "component": "MaPage",
      "extension": ".vue",
      "props": { "title": "Titre personnalisé pour cette instance" },
      "roles": ["admin"]
    }
  ]
}
```

- `path` : nom logique de la route, utilisé en interne par `loadPage(...)` — **pas** l'URL du navigateur (il n'y a pas de `vue-router`).
- `component` : chemin du fichier relatif à `page/`, **sans extension** (ex. `MaPage`, ou `payments/detail` si le fichier est dans un sous-dossier).
- `extension` *(optionnel, défaut `.vue`)* : `.html` uniquement pour un shell statique (déjà utilisé pour `/`, ne pas dupliquer).
- `props` *(optionnel)* : fusionné dans le résultat de `data()` de la page — **les clés de `props` gagnent** sur les valeurs par défaut du script. C'est le mécanisme normal pour injecter des données spécifiques à l'environnement sans toucher au fichier `.vue`.
- `roles` *(optionnel, tableau de strings)* : si présent, la page nécessite une authentification et l'un des rôles listés — sinon **403 Forbidden**, avant même que le contenu de la page ne soit renvoyé.

> **Limitation** : le routeur de pages fait une **comparaison exacte** de `path` (`Map.get`) — pas de segments dynamiques type `:id`. Pour un détail avec identifiant, passez-le en query parameter (`loadPage("payments/detail?id=123")`) ou chargez la donnée depuis un appel à votre endpoint API.

### 2.3. (Optionnel) Ajouter une entrée de menu

Dans `menu/entries.json` :

```json
{ "id": "ma-page", "label": "Ma page", "page": "ma-page" }
```

`page` correspond au `path` de `routes.json`, **sans le `/`** initial. `AppMenu.vue` appelle alors `loadPage("ma-page")` au clic.

### 2.4. Ce qui se passe réellement quand la page est demandée

1. Navigation (clic menu, `<custom-link>`, ou appel direct à `loadPage("ma-page")`) — jamais un changement d'URL du navigateur.
2. `runtime.js` fait d'abord un appel léger : `GET /{appId}/page/ma-page?version=true` → réponse texte brute (un hash/timestamp), comparée au cache mémoire local. Si identique, aucun autre appel n'est fait.
3. Sinon : `GET /{appId}/page/ma-page` → réponse **JSON** `{ template, script, style, props }`. Le script est passé à `new Function(...)`, le style est injecté (dédupliqué par id), le tout est assemblé en objet composant Vue et mis en cache.
4. `<component :is="currentPage">` (dans `AppRoot.vue`) affiche le nouveau composant.

**Point d'attention** : une requête `GET /{appId}/page/ma-page` faite par une **navigation directe du navigateur** (pas un `fetch`/XHR du frontend, donc sans header `X-Requested-With: XMLHttpRequest`) reçoit le **shell HTML complet** (l'app SPA elle-même), pas le JSON — c'est ce qui permet de recharger l'app depuis n'importe quelle URL de page sans 404. Seul le `fetch` interne du frontend reçoit le JSON.

### 2.5. Vérifier

```bash
curl http://localhost:8080/mon-app/page/ma-page?version=true    # texte brut (version)
curl -H "X-Requested-With: XMLHttpRequest" \
     http://localhost:8080/mon-app/page/ma-page                 # JSON {template,script,style,props}
```

---

## 3. Créer un composant réutilisable asynchrone

Il existe **deux mécanismes distincts** pour un composant réutilisable async — ne pas les confondre. **Les deux sont en réalité des `Vue.defineAsyncComponent(...)`** : `createDynamicComponent(name)` *retourne elle-même* un `defineAsyncComponent(...)` (voir `runtime.js`, la fonction est littéralement `const createDynamicComponent = (name) => defineAsyncComponent(async () => {...})`). Ce n'est donc **pas** ce qui les distingue. Ce qui diffère, c'est ce qui se passe **à l'intérieur** de la fonction async :

| | (a) Composant dynamique backend | (b) Composant frontend `.vue` (vue3-sfc-loader) |
|---|---|---|
| **Les deux sont** | `defineAsyncComponent(...)` | `defineAsyncComponent(...)` |
| **Ce qui est async à l'intérieur** | Un `fetch` de **données** (JSON) vers le backend PayOS | Un `fetch` + **parsing + compilation** d'un vrai fichier `.vue`, par `loadModule(url, sfcOptions)` |
| **Emplacement** | `component/MonWidget.vue`, à la racine de l'app (**pas** `page/app/src/components/`) | `page/app/src/components/MonWidget.vue` |
| **Qui découpe le fichier source** | Le backend Java, par regex — renvoie `{template, script, style, props}` en JSON (comme une page, §2) | `vue3-sfc-loader`, par un vrai parseur SFC — jamais de passage par le backend |
| **`<script>`** | Objet JS littéral, exécuté par `new Function()` — pas d'`import` (mêmes contraintes que §2.1) | Vrai module ES, transpilé par un Babel embarqué — `import` fonctionne |
| **`<template>`** | Chaîne de caractères ; compilée en fonction de rendu **par Vue lui-même**, au premier affichage | Compilé en fonction de rendu **directement par `vue3-sfc-loader`**, avant même le premier affichage |
| **Appel** | `createDynamicComponent('MonWidget')` | `Vue.defineAsyncComponent(() => loadModule(url, sfcOptions))` |

### Quand utiliser lequel ?

- **Composant dynamique backend (a)** : le composant doit pouvoir être **modifié/versionné/personnalisé sans redéployer le frontend** — ex. un widget dont le contenu doit différer par tenant, ou dont le contenu métier est piloté par une équipe backend/ops plutôt que par l'équipe frontend. Contrepartie : pas d'`import`, style Options API imposé, aucun outillage de build/lint sur ce fichier.
- **Composant frontend `.vue` (b)** : c'est le cas **par défaut** pour un composant réutilisable "classique" (un bouton, un tableau, un sélecteur...) qui fait partie de votre bibliothèque de composants frontend, développé avec de vrais outils Vue (imports, Composition API, éventuellement testable unitairement). Choisissez ce mécanisme sauf besoin explicite du point précédent.

### 3a. Composant dynamique backend

Créer `component/MonWidget.vue` (mêmes règles que pour une page, §2.1 : pas d'`import`, Options API) :

```html
<template>
  <div class="mon-widget">{{ label }}</div>
</template>
<script>
export default {
  data() {
    return { label: "Widget" };
  },
};
</script>
<style>
.mon-widget { font-weight: bold; }
</style>
```

L'utiliser dans une page (`page/*.vue`) :

```html
<template>
  <Suspense>
    <template #default>
      <MonWidget />
    </template>
    <template #fallback>
      <div>Chargement...</div>
    </template>
  </Suspense>
</template>
<script>
export default {
  components: {
    MonWidget: createDynamicComponent('MonWidget'),
  },
};
</script>
```

`createDynamicComponent` va chercher `GET /{appId}/component/MonWidget` (avec le même contrôle de version `?version=true` que les pages) et assemble le composant à partir du JSON `{template, script, style, props}` renvoyé par le backend. `<Suspense>` est recommandé (pas obligatoire) pour afficher un `fallback` pendant le chargement.

> **Cache partagé** : le cache mémoire des pages et des composants dynamiques est le même — évitez qu'une page et un composant portent exactement le même nom.

### 3b. Composant frontend `.vue` (le cas par défaut)

Créer `page/app/src/components/MonComposant.vue` — un vrai fichier `.vue`, avec de vrais `import` :

```html
<template>
  <span class="mon-composant">{{ label }}</span>
</template>

<script>
export default {
  name: "MonComposant",
  props: {
    label: { type: String, required: true },
  },
};
</script>

<style>
.mon-composant { font-weight: bold; }
</style>
```

L'enregistrer globalement dans `page/app/src/main.js`, sur le modèle de `CustomLink`/`PlayfulCheckbox` déjà présents :

```js
// Près des autres *Url :
const monComposantUrl = new URL("./components/MonComposant.vue", import.meta.url).href;

// Près des autres *Async :
const MonComposantAsync = Vue.defineAsyncComponent(() => loadModule(monComposantUrl, sfcOptions));

// Près des autres app.component(...) :
app.component("MonComposant", MonComposantAsync);
```

Une fois enregistré globalement, il est utilisable **sans aucune déclaration locale**, dans n'importe quel template — pages backend (`page/*.vue`, §2) ou composants frontend :

```html
<mon-composant label="Bonjour" />
```

`defineAsyncComponent()` gère seul son état de chargement (rien ne s'affiche tant que la Promise n'est pas résolue) — `<Suspense>` n'est utile que pour coordonner **plusieurs** composants asynchrones autour d'un fallback commun ; pour un composant global enregistré comme ci-dessus, ce n'est pas nécessaire.

---

## 4. Importer des composants Vue "normaux" (non asynchrones)

Quatre cas de figure selon l'endroit où vous voulez utiliser un composant/une bibliothèque avec un `import` classique et synchrone. **Le cas §4a est le plus fréquent en pratique** : c'est celui à lire en premier si vous voulez utiliser une bibliothèque de composants Vue tierce (design system, lib de graphiques, etc.) dans vos pages (`page/*.vue`, §2) sans passer par `vue3-sfc-loader`/`loadModule`.

### 4a. Bibliothèque de composants Vue tierce déjà compilée (cas le plus courant)

La quasi-totalité des bibliothèques de composants Vue publiées (design systems, kits de graphiques, etc.) sont distribuées sous forme de **fichiers déjà compilés** — pas de `.vue` brut à parser. Vous n'avez donc **pas besoin de `vue3-sfc-loader`/`loadModule` du tout** pour les utiliser : elles se chargent comme n'importe quel autre fichier `.js`.

Il y a une seule difficulté pratique à connaître : la plupart de ces bibliothèques attendent Vue comme **peer dependency**. Deux formats de build existent, et le bon choix dépend de ça — cette app n'a pas de bundler ni d'`importmap` par défaut, donc :

**Option recommandée — build UMD/global de la bibliothèque** (le fichier auto-s'enregistre sur `window`, et s'attend à trouver Vue sur `window.Vue`) :

```html
<!-- page/index.html, avant le <script type="module" src="./app/src/main.js"> -->
<script src="./app/vendor/ma-lib.umd.js"></script>
```

```js
// page/app/src/main.js
import * as Vue from "../vendor/vue.js";
window.Vue = Vue; // la bibliothèque UMD lit Vue depuis le global, pas via import

const app = Vue.createApp(AppRootAsync);
app.use(window.MaLib); // seulement si la lib s'enregistre comme plugin Vue — voir ci-dessous sinon
```

Aucun `defineAsyncComponent`, aucun fetch/compile : c'est un `<script>` classique chargé une fois au démarrage, exactement comme le `vue.js` vendorisé.

**Si la bibliothèque ne s'enregistre pas comme plugin Vue** (pas de méthode `install(app)` — le cas le plus courant pour une bibliothèque de composants qui ne fait "que" exposer des composants, sans logique de plugin) : `app.use(...)` ne fonctionne pas, il n'y a **pas de raccourci** — chaque composant doit être enregistré individuellement avec `app.component(nom, Composant)`, exactement comme pour vos propres composants (§3b).

1. **Trouvez le nom de variable globale exposé par le build.** Il n'est pas toujours `window.MaLib` — regardez la doc de la bibliothèque, ou ouvrez le fichier UMD et cherchez la ligne du type `global.NomDeLaLib = {}` tout en haut (c'est le pattern UMD standard). Une fois le script chargé, inspectez-le simplement dans la console du navigateur : `console.log(window.NomDeLaLib)`.
2. **Listez ce qu'il contient** : la plupart des UMD de composants exposent un objet plat `{ Button, Input, DataTable, ... }` — chaque valeur est un composant Vue directement utilisable par `app.component(...)`.

```js
// page/app/src/main.js, après le chargement du <script> UMD (voir plus haut)
app.component("MaLibButton", window.MaLib.Button);
app.component("MaLibInput", window.MaLib.Input);
app.component("MaLibDataTable", window.MaLib.DataTable);
```

Pour éviter de lister chaque composant à la main, une simple boucle fonctionne aussi bien si vous voulez tout enregistrer d'un coup (à condition que `window.MaLib` ne contienne *que* des composants, pas d'utilitaires divers — vérifiez son contenu au préalable, §1 ci-dessus) :

```js
Object.entries(window.MaLib).forEach(([name, component]) => {
  app.component(`MaLib${name}`, component); // ex. <ma-lib-button>, préfixé pour éviter les collisions
});
```

> **CSS de la bibliothèque** : un design system livre presque toujours une feuille de style séparée (le CSS n'est jamais inclus dans le bundle JS UMD). Comme ce projet n'a pas de bundler pour l'inliner, ajoutez-la simplement en `<link>` dans `page/index.html` :
> ```html
> <link rel="stylesheet" href="./app/vendor/ma-lib.css" />
> ```
> Sans ça, les composants s'affichent mais sans aucun style — un oubli fréquent, facile à confondre avec un problème d'enregistrement du composant.

**Alternative — build ESM de la bibliothèque**, si vous préférez un vrai `import` et n'avez pas de build UMD disponible : son fichier ESM contient très probablement lui-même `import { h, ... } from "vue"`, un specifier "bare" que le navigateur ne sait résoudre nativement que via un **import map** (absent par défaut dans ce template). Ajoutez-le dans `page/index.html`, avant le `<script type="module">` qui charge `main.js` :

```html
<script type="importmap">
{
  "imports": {
    "vue": "./app/vendor/vue.esm-browser.js"
  }
}
</script>
<script type="module" src="./app/src/main.js"></script>
```

Puis dans `main.js`, un `import` tout à fait normal :

```js
import { Button, DataTable } from "../vendor/ma-lib.esm.js";
app.component("MaLibButton", Button);
app.component("MaLibDataTable", DataTable);
```

Dans les deux cas, une fois les composants enregistrés **globalement** via `app.component(...)`, ils sont utilisables **sans aucun `import`** dans vos pages (`page/*.vue`, §2) — voir §4d, c'est exactement le même mécanisme déjà utilisé par `<custom-link>`.

### 4b. Composant `.js` maison, sans `vue3-sfc-loader`

Pour votre **propre** petit composant (pas une bibliothèque tierce), si vous n'avez pas besoin de la syntaxe SFC (`<template>`/`<style>` séparés) et voulez un `import` **normal et synchrone**, écrivez-le comme un fichier `.js` (pas `.vue`), avec le template en chaîne de caractères :

```js
// page/app/src/components/MonComposant.js
export default {
  name: "MonComposant",
  props: {
    label: { type: String, required: true },
  },
  template: `<span class="mon-composant">{{ label }}</span>`,
};
```

```js
// Dans main.js, ou dans n'importe quel .js importé normalement :
import MonComposant from "./components/MonComposant.js";
app.component("MonComposant", MonComposant); // pas de defineAsyncComponent
```

Ça fonctionne car le build Vue vendorisé embarque le compilateur de template runtime (pas juste le runtime-only) — une chaîne `template: "..."` est donc compilée normalement par Vue lui-même, sans passer par `vue3-sfc-loader`.

**Compromis** : pas de bloc `<style>` séparé (injectez le CSS à la main, voir `utils/inject-style.js`), et le template est une chaîne plutôt qu'un vrai bloc HTML dans l'éditeur.

### 4c. `import` classique *à l'intérieur* d'un `.vue` compilé par `vue3-sfc-loader`

Pour garder la syntaxe SFC (§3b) tout en important une bibliothèque externe depuis le `<script>` d'un `.vue` compilé par `vue3-sfc-loader` : `vue3-sfc-loader` embarque son propre Babel, mais **ne sait pas parser un fichier `.js` externe qui contient lui-même des `import`/`export`** (limitation connue de la bibliothèque). La solution est `sfcOptions.moduleCache`, dans `main.js` :

```js
// 1. Import normal du vendor dans main.js (ESM natif)
import * as MaLib from "../vendor/ma-lib.esm.js";

// 2. L'exposer sous un nom stable aux .vue compilés par vue3-sfc-loader
const sfcOptions = {
  moduleCache: {
    vue: Vue,
    "app-runtime": RuntimeModule,
    "app-config": ConfigModule,
    "ma-lib": MaLib,          // ← nouvelle entrée
  },
  getFile,
  addStyle,
};
```

N'importe quel `.vue` compilé avec ce `sfcOptions` peut alors écrire, dans son `<script>` :

```js
import { Button } from "ma-lib";
```

Résolution instantanée (pas de fetch/compile), exactement le même principe que pour `vue`/`app-runtime`/`app-config` déjà présents. Notez que ce cas est **redondant** avec §4a pour une bibliothèque tierce : n'utilisez §4c que si vous écrivez vous-même un `.vue` du shell (§3b) qui a besoin d'importer cette bibliothèque directement dans son propre `<script>`, plutôt que de simplement l'enregistrer globalement.

### 4d. Dans une page ou un composant backend (`page/*.vue`, `component/*.vue`)

**Aucun `import` n'est possible ici**, quelle que soit la méthode — le script est exécuté via `new Function(...)` côté navigateur (§2.1), et `import` n'est légal qu'en tête de module ES, jamais dans un corps de fonction ; ça produirait une erreur de syntaxe immédiate.

La seule voie qui fonctionne pour ce contenu : enregistrer le composant **globalement** (§3b, §4a ou §4b) une fois dans `main.js`, puis le référencer **par son nom de tag**, sans aucune déclaration ni import, dans n'importe quelle page/composant backend :

```html
<!-- page/MaPage.vue -->
<template>
  <mon-composant label="Bonjour" />
  <ma-lib-button>Valider</ma-lib-button>
</template>
```

C'est exactement le mécanisme déjà utilisé par `<custom-link>` dans vos pages — Vue résout `<mon-composant>`/`<ma-lib-button>` → le composant global enregistré, indépendamment de tout système de module. **C'est donc la réponse à "comment utiliser une bibliothèque de composants tierce dans mes pages"** : enregistrez-la une fois (§4a), utilisez-la partout par tag, sans jamais l'importer dans le fichier de la page elle-même.

### Résumé

| Où voulez-vous l'utiliser ? | Mécanisme |
|---|---|
| Bibliothèque tierce déjà compilée, quel que soit l'endroit final d'usage | Enregistrement global (§4a) puis usage par tag — voir §4d pour l'usage dans une page |
| Votre propre composant, dans le shell (`page/app/src/*.vue`, compilé par `vue3-sfc-loader`) | `sfcOptions.moduleCache` (§4c) → `import` classique dans le `.vue` |
| Votre propre composant, n'importe où, sans passer par `vue3-sfc-loader` | Composant `.js` avec `template` en chaîne + `import` natif (§4b) |
| Dans une page/composant backend (`page/*.vue`, `component/*.vue`) | `app.component(...)` global (§3b, §4a ou §4b), puis usage par tag — **pas d'`import` possible ici** |

---

## 5. Pour aller plus loin

- [Créer une application PayOS](create-application-guide.md) — structure complète d'une application (API, base de données, sécurité, hooks), section 7 pour le rappel côté page.
- [Guide JavaScript des Endpoints API](javascript-api-endpoint-guide.md) — le contrat des scripts API (`$Api`, `$Request`, `$Response`, ...), notamment la propagation automatique des en-têtes par `$Api` entre appels API.
- [Scripting bindings reference](scripting-bindings.md) — chaque binding `$...` injecté côté script.
- [Architecture : request processing](../architecture/request-processing.md) — le pipeline complet côté kernel.
- [Sécurité — inventaire](../architecture/security/security-inventory.md) — RBAC, dont le champ `roles` sur une route de page (§2.2 de ce document).
