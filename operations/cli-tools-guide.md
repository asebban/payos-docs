# PayOS CLI Tools — Guide d'utilisation

Last Updated: 2026-05-31

---

## Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Installation des outils](#2-installation-des-outils)
3. [generate_app — Scaffolding d'application](#3-generate_app--scaffolding-dapplication)
4. [BootServer — Démarrage du runtime](#4-bootserver--démarrage-du-runtime)
5. [cpm — Capability Package Manager](#5-cpm--capability-package-manager)
6. [apm — Application Package Manager](#6-apm--application-package-manager)
7. [ppm — Product Package Manager](#7-ppm--product-package-manager)
8. [spm — Secret Package Manager](#8-spm--secret-package-manager)
9. [edc — Encryption Decryption Command](#9-edc--encryption-decryption-command)
10. [pdoc — OpenAPI Documentation Generator](#10-pdoc--openapi-documentation-generator)
11. [Configuration des catalogues](#11-configuration-des-catalogues)
12. [Tableau comparatif](#12-tableau-comparatif)
13. [Codes d'erreur](#13-codes-derreur)

---

## 1. Vue d'ensemble

PayOS fournit un ensemble d'outils CLI couvrant le cycle de vie complet d'une application, du scaffolding au déploiement.

| Outil | Rôle |
|---|---|
| `generate_app` | Génère le squelette d'une nouvelle application PayOS |
| `BootServer` | Démarre le runtime PayOS (HTTP, TCP, Queue) |
| `cpm` | Installe, désinstalle, active et désactive des **capabilities** |
| `apm` | Installe et désinstalle une **application** individuelle dans le runtime |
| `ppm` | Installe et désinstalle un **produit** — bundle de plusieurs applications |
| `spm` | Provisionne et gère les **secrets chiffrés** du provider filesystem |
| `edc` | Chiffre et déchiffre les **bundles** (pack/unpack) avec clé AES |
| `pdoc` | Génère la documentation **OpenAPI 3.1** statique depuis les annotations |

`cpm`, `apm` et `ppm` agissent sur le fichier `bootstrap.json` du runtime et lisent leur configuration depuis `payos.json`. `spm` est un outil d'administration indépendant qui opère directement sur le répertoire de stockage des secrets. `edc` chiffre/déchiffre les fichiers d'un bundle de manière récursive. `pdoc` scanne les scripts API pour générer des spécifications OpenAPI sans exécuter le runtime.

**Pré-requis** : Java 21+

---

## 2. Installation des outils

### Construction de cpm / apm / ppm

Depuis le répertoire `payos-pm` :

```bash
mvn package -DskipTests
```

Produit dans `target/` :
- `cpm.jar`
- `apm.jar`
- `ppm.jar`

### Installation système (Linux / macOS)

```bash
# Depuis le répertoire payos-pm
./install-payos-tools.sh [SOURCE_DIR [LIB_DIR [BIN_DIR]]]
```

| Argument | Défaut (utilisateur) | Défaut (root) | Description |
|---|---|---|---|
| `SOURCE_DIR` | répertoire du script | — | Racine du module `payos-pm` |
| `LIB_DIR` | `$HOME/.local/lib/payos-pm` | `/usr/local/lib/payos-pm` | Destination des JARs |
| `BIN_DIR` | `$HOME/.local/bin` | `/usr/local/bin` | Destination des wrappers shell |

Le script crée les répertoires nécessaires, copie les JARs, génère les wrappers shell et ajoute `BIN_DIR` au `PATH` dans le profil shell.

### Installation système (Windows)

```powershell
# Depuis le répertoire payos-pm
.\install-payos-tools.ps1

# En spécifiant les répertoires source et cible
.\install-payos-tools.ps1 -BuildDir "C:\build\payos-pm" -InstallDir "C:\tools\payos"
```

| Paramètre | Défaut | Description |
|---|---|---|
| `-BuildDir` | répertoire courant | Répertoire contenant les JARs buildés |
| `-InstallDir` | `$HOME\.payos\bin` | Répertoire de destination des JARs et wrappers |

Le script localise les JARs, génère des wrappers `.cmd` et ajoute `InstallDir` au `PATH` utilisateur (registre, sans droits administrateur).

### Invocation directe (sans installation)

```bash
# Linux / macOS
./cpm --help
./apm --help
./ppm --help

# Windows
cpm.cmd --help
apm.cmd --help
ppm.cmd --help
```

### Construction de spm

Depuis le répertoire `secret-service-filesystem` :

```bash
mvn package -DskipTests
```

Produit dans `target/` :
- `spm.jar` — fat JAR autonome (inclut picocli + logback)

### Installation système de spm (Linux / macOS)

```bash
# Depuis le répertoire secret-service-filesystem
cp target/spm.jar scripts/
./scripts/install.sh                         # → ~/.payos/bin (défaut)
./scripts/install.sh /usr/local/bin          # → répertoire personnalisé
PAYOS_HOME=/opt/payos ./scripts/install.sh   # → $PAYOS_HOME/bin
```

| Argument | Défaut | Description |
|---|---|---|
| `$1` (optionnel) | `${PAYOS_HOME:-$HOME/.payos}/bin` | Répertoire d'installation |

Le script copie `spm.jar`, génère le wrapper shell `spm` (résolution de symlinks incluse), et ajoute le répertoire cible au `PATH` dans `~/.bashrc`, `~/.zshrc` ou `~/.profile`.

### Installation système de spm (Windows)

```powershell
# Depuis le répertoire secret-service-filesystem
Copy-Item target\spm.jar scripts\
.\scripts\install.ps1                                          # → %USERPROFILE%\.payos\bin
.\scripts\install.ps1 -InstallDir "C:\Tools\payos\bin"        # → répertoire personnalisé
.\scripts\install.ps1 -JarPath "D:\build\spm.jar" -InstallDir "C:\Tools\payos\bin"
```

| Paramètre | Défaut | Description |
|---|---|---|
| `-InstallDir` | `$env:PAYOS_HOME\bin` ou `$env:USERPROFILE\.payos\bin` | Répertoire d'installation |
| `-JarPath` | `scripts\spm.jar` | Chemin vers le fat JAR |

Le script copie `spm.jar`, génère `spm.ps1` (wrapper PowerShell) et `spm.cmd` (wrapper cmd.exe), et met à jour le `PATH` utilisateur dans le registre Windows. La commande est disponible immédiatement dans la session courante.

### Invocation directe de spm (sans installation)

```bash
# Linux / macOS
java -jar secret-service-filesystem/target/spm.jar --help

# Windows
java -jar secret-service-filesystem\target\spm.jar --help
```

---

## 3. generate_app — Scaffolding d'application

`generate_app` crée un squelette d'application PayOS complet, prêt à être enregistré via `apm`.

### Syntaxe

```bash
# Linux / macOS
./generate_app.sh --app-id <id> [--output <base-dir>] [--template standard|ui]
./generate_app.sh <app-id> [<base-dir>]          # forme positionnelle

# Windows
.\generate_app.ps1 -AppId <id> [-OutputDir <base-dir>] [-Template standard|ui]
.\generate_app.ps1 <app-id> [<base-dir>]         # forme positionnelle
```

### Options

| Option | Alias PowerShell | Description |
|---|---|---|
| `--app-id <id>` | `-AppId` (obligatoire) | Identifiant unique de l'application (ex. `my-app`) |
| `--output <dir>` | `-OutputDir` / `-output` | Répertoire parent où créer le dossier `<app-id>`. Défaut : répertoire courant |
| `--template standard|ui` | `-Template` | Type de squelette à générer. `standard` conserve le comportement actuel ; `ui` ajoute un template Nuxt UI |

### Exemples

```bash
# Linux / macOS
./generate_app.sh --app-id payment-portal
./generate_app.sh --app-id payment-portal --output /opt/payos/apps
./generate_app.sh --app-id payment-portal --output /opt/payos/apps --template ui

# Windows
.\generate_app.ps1 -AppId payment-portal
.\generate_app.ps1 -AppId payment-portal -output C:\payos\apps
.\generate_app.ps1 -AppId payment-portal -output C:\payos\apps -Template ui
```

### Structure générée

```
<app-id>/
  manifest.json               # Descripteur de l'application
  config/
    application.json          # Configuration applicative
    mappings.json             # Mappings de ressources
    routes.json               # Définition des routes
  api/
    items/
      list.js                 # GET /items
      get.js                  # GET /items/:id
      create.js               # POST /items
  lib/
    utils.js                  # Utilitaires partagés
  page/
    index.vue                 # Page d'accueil
    home.vue                  # Page principale
  menu/
    entries.json              # Définition du menu
```

En mode `ui`, le script conserve ce squelette backend et y ajoute un frontend Nuxt entièrement embarqué dans le script : `app/`, `components/`, `composables/`, `plugins/`, `public/`, `nuxt.config.ts`, `package.json`, `tsconfig.json`, ainsi qu'une version enrichie de `config/routes.json` et `menu/entries.json`. Aucun template externe n'est requis sur disque.

---

## 4. BootServer — Démarrage du runtime

`BootServer` est le point d'entrée principal du runtime PayOS. Il charge la configuration, initialise les services et démarre les serveurs déclarés dans `bootstrap.json`.

### Syntaxe

```bash
java -jar payos.jar [--bundle-path <payos.json>]
```

### Options

| Option | Description |
|---|---|
| `--bundle-path <file>` | Chemin vers le fichier `payos.json`. Si absent, résout `payos.json` dans le répertoire courant puis dans le classpath |

### Exemple

```bash
# Démarrage standard (payos.json dans le répertoire courant)
java -jar payos.jar

# Démarrage avec un fichier de configuration explicite
java -jar payos.jar --bundle-path /opt/payos/payos.json
```

### Comportement au démarrage

1. Chargement de `payos.json` (fichier explicite ou détection automatique)
2. Initialisation du service de base de données
3. Initialisation du client queue (NATS) si configuré
4. Initialisation de l'`ActivationStore` pour les capabilities
5. Démarrage des serveurs déclarés dans `bootstrap.json` (HTTP, TCP, Queue)
6. Activation du `ConfigWatcher` : hot-reload automatique lors de modifications de `payos.json`, du répertoire `configDir`, du répertoire `.capabilities/` et des configurations applicatives

---

## 5. cpm — Capability Package Manager

### Qu'est-ce qu'une capability ?

Une capability est une extension fonctionnelle versionnée et indépendante d'une application PayOS. Elle embarque :

- Des scripts JavaScript d'endpoint (même format que les scripts applicatifs)
- Des composants et des pages UI
- Des descripteurs de menu
- Un descripteur `manifest.json`
- Des fichiers de configuration optionnels
- Des mappings Hibernate optionnels
- Des hooks de cycle de vie optionnels (`hooks/lifecycle.js`)

Une fois installée, une capability est enregistrée dans `bootstrap.json` comme une application ordinaire avec `"category": "capability"`. L'activer ajoute son `id` dans le tableau `extends` des application(s) cibles.

### Syntaxe générale

```
cpm [--bundle <path>] --install    --id <id> [--version <v>] (--path <dir> | --from-catalog)
cpm [--bundle <path>] --uninstall  --id <id> [--cascade] [--drop-schema]
cpm [--bundle <path>] --activate   --id <id> [--app <appId>] [--tenant <tenantId>]
cpm [--bundle <path>] --deactivate --id <id> [--app <appId>] [--tenant <tenantId>]
cpm [--bundle <path>] --status     --id <id>
```

### Options globales

| Option | Défaut | Description |
|---|---|---|
| `--bundle <path>` | `.` (répertoire courant) | Chemin vers le répertoire runtime contenant `payos.json` |
| `--help` / `-h` | — | Affiche l'aide et quitte |
| `--version` / `-V` | — | Affiche la version et quitte |

### install

Installe une capability dans le runtime.

```bash
# Depuis un répertoire local
cpm --bundle /opt/payos --install --id payment-links --path /packages/payment-links

# Depuis le catalogue configuré (dernière version)
cpm --bundle /opt/payos --install --id payment-links --from-catalog

# Depuis le catalogue, version spécifique
cpm --bundle /opt/payos --install --id payment-links --version 1.2.0 --from-catalog
```

| Option | Description |
|---|---|
| `--id <id>` | Identifiant de la capability (doit correspondre à `manifest.json:id`) |
| `--path <dir>` | Répertoire local contenant le package. Mutuellement exclusif avec `--from-catalog` |
| `--from-catalog` | Télécharge le package depuis le catalogue configuré dans `payos.json` |
| `--version <v>` | Version semver à installer. Si absente, résout la dernière version disponible |

**Séquence d'installation :**

1. Résolution du package (chemin local ou téléchargement depuis le catalogue)
2. Lecture de `manifest.json` et vérification des dépendances déclarées
3. Abandon avant tout effet de bord si une dépendance est introuvable
4. Installation récursive des dépendances manquantes
5. Copie du package vers `{configDir}/.capabilities/{id}/`
6. Écriture du descripteur dans `bootstrap.json` sous `applications`
7. Écriture de l'entrée dans `registry.json`
8. Exécution de `hooks/lifecycle.js#install(ctx)` — en cas d'échec : marque `FAILED`, skip l'auto-activation
9. Ajout d'un événement `INSTALL` dans `events.ndjson`
10. Auto-activation globale (équivalent à `cpm --activate --id <id>`)

### uninstall

Désinstalle une capability du runtime.

```bash
cpm --bundle /opt/payos --uninstall --id payment-links

# Avec désinstallation en cascade des capabilities dépendantes
cpm --bundle /opt/payos --uninstall --id payment-links --cascade

# En supprimant le schéma base de données associé
cpm --bundle /opt/payos --uninstall --id payment-links --drop-schema
```

| Option | Description |
|---|---|
| `--id <id>` | Capability à désinstaller |
| `--cascade` | Désinstalle récursivement toutes les capabilities qui en dépendent. Sans ce flag, la commande échoue s'il existe des dépendants |
| `--drop-schema` | Passe `dropSchema=true` au hook `uninstall` (le hook décide comment l'utiliser, ex : suppression de tables) |

### activate

Rend une capability disponible pour une ou toutes les applications, avec restriction optionnelle à un tenant.

```bash
# Activation globale (toutes applications, tous tenants)
cpm --bundle /opt/payos --activate --id payment-links

# Activation pour une application spécifique
cpm --bundle /opt/payos --activate --id payment-links --app my-app

# Activation pour une application et un tenant spécifiques
cpm --bundle /opt/payos --activate --id payment-links --app my-app --tenant bank-a
```

| Option | Description |
|---|---|
| `--id <id>` | Capability à activer. Doit être déjà installée |
| `--app <appId>` | Restreint l'activation à une seule application. Si absent, cible toutes les applications non-capability |
| `--tenant <tenantId>` | Restreint l'activation à un tenant unique dans l'application ciblée |

### deactivate

Retire une capability du scope applicatif sans la désinstaller.

```bash
cpm --bundle /opt/payos --deactivate --id payment-links

# Désactivation ciblée application + tenant
cpm --bundle /opt/payos --deactivate --id payment-links --app my-app --tenant bank-a
```

Les options sont identiques à `activate`. La désactivation ajoute une entrée négative (`active: false`) dans `activation.json` pour neutraliser une activation de portée plus large.

### status

Affiche l'état d'installation et d'activation d'une capability.

```bash
cpm --bundle /opt/payos --status --id payment-links
```

### Structure d'un package capability

```
my-capability/
  manifest.json           # Obligatoire — descripteur de la capability
  hooks/
    lifecycle.js          # Optionnel — hooks de cycle de vie
  scripts/
    api/
      orders.GET.js       # Script API exemple
  model/
    orders.hbm.xml        # Optionnel — fichiers de mapping Hibernate
  config/
    settings.json         # Optionnel — configuration additionnelle
```

**manifest.json :**

```json
{
  "id": "payment-links",
  "name": "Payment Links",
  "version": "1.2.0",
  "description": "Permet aux marchands de créer et partager des liens de paiement.",
  "dependencies": [
    { "id": "payment-core" },
    { "id": "notifications", "version": "2.0.0" }
  ]
}
```

**hooks/lifecycle.js :**

```javascript
exports.install = function(ctx) {
    // ctx.capabilityId, ctx.version, ctx.configDir
    // Lancer une exception pour signaler un échec
};

exports.activate = function(ctx) {
    // ctx.capabilityId, ctx.configDir
    // ctx.appId    — présent si activation pour une app spécifique
    // ctx.tenantId — présent si activation pour un tenant spécifique
};

exports.deactivate = function(ctx) { /* même ctx qu'activate */ };

exports.uninstall = function(ctx) {
    // ctx.dropSchema — booléen, transmis depuis --drop-schema
};
```

> Les hooks s'exécutent dans un sandbox GraalVM JS (`allowAllAccess=false`). Une exception dans un hook marque l'opération comme échouée. Un fichier hook absent ou une fonction manquante est ignoré silencieusement.

### Fichiers d'état

Tous les fichiers d'état sont stockés sous `{configDir}/.capabilities/`. **Ne pas les éditer manuellement.**

| Fichier | Description |
|---|---|
| `registry.json` | Enregistrements d'installation : `id`, `version`, `installedAt`, `status` (`INSTALLED` / `FAILED`) |
| `activation.json` | Entrées de scope d'activation : `capabilityId`, `applicationId`, `tenants`, `active` |
| `events.ndjson` | Journal d'audit append-only : événements `INSTALL`, `ACTIVATE`, `DEACTIVATE`, `UNINSTALL` |
| `.capabilities/{id}/` | Répertoire du package installé |

---

## 6. apm — Application Package Manager

### Qu'est-ce qu'une application PayOS ?

Une application PayOS est un répertoire contenant des scripts, des configurations et des ressources qui sont enregistrés dans `bootstrap.json` et chargés au démarrage du runtime. `apm` gère l'enregistrement et le désenregistrement d'une application individuelle.

### Syntaxe générale

```
apm [--bundle <path>] --install   --app <id[@version]|path> [--base-path <dir>]
apm [--bundle <path>] --uninstall --app <id>
apm [--bundle <path>] --status    --app <id>
```

### Options globales

| Option | Défaut | Description |
|---|---|---|
| `--bundle <path>` | `.` (répertoire courant) | Chemin vers le répertoire runtime contenant `payos.json` |

### install

Enregistre une application dans le runtime.

```bash
# Depuis un répertoire local (contient un manifest.json)
apm --bundle /opt/payos --install --app ./my-app

# Depuis le catalogue (dernière version)
apm --bundle /opt/payos --install --app my-app

# Depuis le catalogue, version spécifique
apm --bundle /opt/payos --install --app my-app@2.1.0

# Enregistrement minimal sans manifest (base-path déjà présent sur disque)
apm --bundle /opt/payos --install --app my-app --base-path /opt/payos/apps/my-app
```

| Option | Description |
|---|---|
| `--app <spec>` | Spécification de l'application. Voir le tableau de résolution ci-dessous |
| `--base-path <dir>` | Chemin de base de l'application. Fallback utilisé quand aucun manifest n'est trouvé et qu'aucun catalogue n'est configuré |

**Résolution de `--app <spec>` :**

| Forme de `<spec>` | Résolution |
|---|---|
| Chemin local (contient `/`, `\` ou commence par `.`) | Utilise le `manifest.json` trouvé dans ce répertoire ou ce fichier |
| `id` simple | Résout depuis l'`applicationCatalog` (dernière version) |
| `id@version` | Résout depuis l'`applicationCatalog` à la version exacte |

**Fallback sans catalog :** si aucun `manifest.json` n'est trouvé et qu'aucun catalogue n'est configuré, `--base-path` permet un enregistrement minimal (génère un manifest avec `id` + `base.path` + `version`).

**Le `manifest.json` d'une application est l'entrée `bootstrap.json` complète :**

```json
{
  "id": "my-app",
  "base.path": "/opt/payos/apps/my-app",
  "version": "1.0.0",
  "name": "My Application",
  "authorized.tenants": ["tenant1", "tenant2"],
  "extends": [],
  "security": {
    "provider": "nimbus"
  }
}
```

### uninstall

Retire l'entrée de l'application depuis `bootstrap.json`. **Les fichiers sur disque ne sont pas supprimés.**

```bash
apm --bundle /opt/payos --uninstall --app my-app
```

### status

Affiche l'état d'enregistrement de l'application dans `bootstrap.json`.

```bash
apm --bundle /opt/payos --status --app my-app
```

---

## 7. ppm — Product Package Manager

### Qu'est-ce qu'un produit ?

Un produit est un bundle qui regroupe plusieurs applications PayOS. Il est décrit par un manifest produit qui liste les applications à installer et les serveurs à configurer. `ppm` installe ou désinstalle toutes ces applications en une seule commande.

### Syntaxe générale

```
ppm [--bundle <path>] --install   --product <name[@version]|path>
ppm [--bundle <path>] --uninstall --product <name>
ppm [--bundle <path>] --status    --product <name>
```

### Options globales

| Option | Défaut | Description |
|---|---|---|
| `--bundle <path>` | `.` (répertoire courant) | Chemin vers le répertoire runtime contenant `payos.json` |

### install

Installe toutes les applications déclarées dans le manifest produit.

```bash
# Depuis un répertoire local (contient un manifest produit)
ppm --bundle /opt/payos --install --product ./acquiring

# Depuis le catalogue (dernière version)
ppm --bundle /opt/payos --install --product acquiring

# Version spécifique depuis le catalogue
ppm --bundle /opt/payos --install --product acquiring@3.0.0
```

| Option | Description |
|---|---|
| `--product <spec>` | Spécification du produit. Même convention que `--app` pour `apm` : chemin local, nom simple ou `name@version` |

**Format du manifest produit (`acquiring.json`) :**

```json
{
  "id": "acquiring",
  "version": "3.0.0",
  "name": "Acquiring Platform",
  "applications": [
    { "id": "acquiring-core",    "basePath": "/opt/payos/apps/acquiring-core",    "version": "3.0.0" },
    { "id": "acquiring-portal",  "basePath": "/opt/payos/apps/acquiring-portal",  "version": "3.0.0" }
  ],
  "servers": [
    { "host": "0.0.0.0", "port": 8082, "protocol": "http" }
  ]
}
```

**Séquence d'installation pour chaque application du manifest :**

- Si l'application est déjà enregistrée et son `basePath` existe sur disque → ajout du tag contributeur uniquement
- Si le `basePath` existe mais l'app n'est pas enregistrée → enregistrement sans téléchargement
- Si le `basePath` n'existe pas → téléchargement depuis l'`applicationCatalog` puis enregistrement
- Pour chaque entrée `servers[]` → fusion dans `bootstrap.json` (skip si le port existe déjà)

> **Tracking contributeur** : `ppm` inscrit le produit dans un champ interne `_contributors[]` sur chaque entrée d'application dans `bootstrap.json`. Cela permet une désinstallation multi-produits sûre.

### uninstall

Retire le tag contributeur du produit sur chaque application. Supprime les entrées d'application dont `_contributors` devient vide.

```bash
ppm --bundle /opt/payos --uninstall --product acquiring
```

> Les fichiers applicatifs sur disque ne sont pas supprimés.

### status

Affiche l'état d'installation du produit et de ses applications.

```bash
ppm --bundle /opt/payos --status --product acquiring
```

---

## 8. spm — Secret Package Manager

`spm` est l'outil d'administration du provider de secrets filesystem. Il permet de provisionner, lire, lister, décrire et supprimer des secrets chiffrés, et de générer les clés maîtresses AES-256.

> **Prérequis** : le service de secrets doit être configuré dans `bootstrap.json` avec `secret-service.configuration.enabled: true` et `type: "filesystem"`. La commande `spm` opère directement sur le répertoire de stockage (`root`) avec la même clé maîtresse (`keyfile` ou `$PAYOS_SECRET_MASTER_KEY`).

### Syntaxe générale

```
spm <command> [options]
spm --help
spm --version
```

### Options communes à toutes les commandes (sauf keygen)

| Option | Requis | Description |
|---|---|---|
| `--root <dir>` | Oui | Répertoire racine des secrets — même valeur que `secret-service.configuration.root` dans `bootstrap.json` |
| `--keyfile <path>` | Non | Chemin vers le fichier clé de 32 octets. Si absent, lit `$PAYOS_SECRET_MASTER_KEY` |
| `-h`, `--help` | Non | Affiche l'aide de la commande et quitte |

---

### keygen — Générer une clé maîtresse

Génère un fichier de clé aléatoire de 32 octets (AES-256). À exécuter une seule fois lors de la mise en place du service de secrets.

```bash
secrets keygen --out <path>
```

| Option | Description |
|---|---|
| `--out <path>` | Chemin de sortie du fichier clé. Échoue si le fichier existe déjà (protection contre l'écrasement accidentel) |

**Exemple :**

```bash
spm keygen --out /opt/payos/secrets/.keyfile
chmod 600 /opt/payos/secrets/.keyfile
```

> **Sécurité** : protéger le fichier clé avec `chmod 600` (Linux/macOS) ou des ACL restrictives (Windows). Sa perte rend tous les secrets irrécupérables.

---

### set — Stocker ou mettre à jour un secret

Chiffre et stocke la valeur d'un secret pour un tenant. Si le secret existe déjà, sa version est incrémentée automatiquement.

```bash
spm set --root <dir> --tenant <id> --name <name> [--value <val>] [--type <type>] [--keyfile <path>]
```

| Option | Requis | Description |
|---|---|---|
| `--tenant <id>` | Oui | Identifiant du tenant (`[a-zA-Z0-9-]+`) |
| `--name <name>` | Oui | Nom du secret (`[a-zA-Z0-9_.-]+`) |
| `--value <val>` | Non | Valeur en clair. Si absent, lit les octets bruts depuis stdin |
| `--type <type>` | Non | Tag de type métadonnée : `api-key`, `password`, `certificate`, etc. (défaut : `string`) |

**Exemples :**

```bash
# Valeur passée en argument
spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key --value sk_live_xxxx --type api-key

# Valeur lue depuis stdin (évite l'exposition dans l'historique shell)
echo -n "sk_live_xxxx" | spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key --type api-key

# Via variable d'environnement pour la clé maîtresse
export PAYOS_SECRET_MASTER_KEY=<base64-32-bytes>
spm set --root /opt/payos/secrets --tenant acme --name db-password --value s3cr3t --type password
```

---

### get — Lire la valeur d'un secret

Déchiffre et écrit la valeur brute du secret sur la sortie standard. Utilisable en pipeline.

```bash
spm get --root <dir> --tenant <id> --name <name> [--keyfile <path>]
```

| Option | Requis | Description |
|---|---|---|
| `--tenant <id>` | Oui | Identifiant du tenant |
| `--name <name>` | Oui | Nom du secret |

**Exemples :**

```bash
# Afficher la valeur
spm get --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key

# Sauvegarder dans un fichier
spm get --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name ssl-certificate > /tmp/cert.pem

# Utiliser en pipeline (injection dans une variable)
API_KEY=$(spm get --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key)
```

---

### list — Lister les secrets d'un tenant

Affiche les noms des secrets disponibles pour un tenant. Les valeurs ne sont pas exposées.

```bash
spm list --root <dir> --tenant <id> [--keyfile <path>]
```

**Exemple :**

```bash
spm list --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile --tenant acme
# stripe-api-key
# db-password
# jwt-signing-key
```

---

### describe — Afficher les métadonnées

Affiche les métadonnées d'un secret (type, version, dates) sans exposer sa valeur.

```bash
spm describe --root <dir> --tenant <id> --name <name> [--keyfile <path>]
```

**Exemple :**

```bash
spm describe --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key
# name     : stripe-api-key
# tenant   : acme
# type     : api-key
# version  : 3
# created  : 2026-05-31T10:00:00Z
# modified : 2026-05-31T16:26:53Z
```

---

### delete — Supprimer un secret

Supprime définitivement un secret et ses métadonnées. Irréversible.

```bash
spm delete --root <dir> --tenant <id> --name <name> [--keyfile <path>]
```

**Exemple :**

```bash
spm delete --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name old-api-key
```

---

### Flux complets

#### Mise en place initiale d'un secret

```bash
# 1. Générer la clé maîtresse
spm keygen --out /opt/payos/secrets/.keyfile
chmod 600 /opt/payos/secrets/.keyfile
mkdir -p /opt/payos/secrets
chmod 700 /opt/payos/secrets

# 2. Configurer bootstrap.json
#    "secret-service": { "configuration": { "enabled": true, "type": "filesystem",
#                        "root": "/opt/payos/secrets", "keyfile": "/opt/payos/secrets/.keyfile" } }

# 3. Provisionner les secrets nécessaires
spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key --value sk_live_xxxx --type api-key
spm set --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name db-password    --value s3cr3t        --type password

# 4. Vérifier
spm list --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile --tenant acme
```

#### Rotation d'un secret (sans impact sur les scripts)

```bash
# Appeler set sur le même nom — version incrémentée automatiquement
echo -n "sk_live_yyyy" | spm set --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile --tenant acme --name stripe-api-key

# Vérifier la nouvelle version
spm describe --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme --name stripe-api-key
# version  : 2
```

#### Audit avant rotation : lister tous les secrets d'un tenant

```bash
spm list --root /opt/payos/secrets --keyfile /opt/payos/secrets/.keyfile --tenant acme
```

### Codes de sortie

| Code | Signification |
|---|---|
| `0` | Succès |
| `1` | Erreur opérationnelle (secret introuvable, accès refusé, erreur de chiffrement, JAR manquant) |
| `2` | Erreur d'utilisation (option obligatoire manquante, valeur invalide) |

---

## 9. edc — Encryption Decryption Command

`edc` (**E**ncryption **D**ecryption **C**ommand) est l'outil de chiffrement/déchiffrement de bundles PayOS. Il permet de protéger l'intégralité d'un bundle (configuration, scripts, ressources) via AES-256, avec un marqueur magique `P8OS` pour détecter les fichiers chiffrés.

> **Prérequis** : Java 21+. Le module est fourni par `payosv2-packer`.

### Syntaxe générale

```
edc --encryption <pack|unpack> --inputdir <dir> [--key <key> | --generatekey <n>]
edc --encryption <pack|unpack> --inputdir <dir> --secret-provider <type> [options-provider]
edc --generatekey <n>
edc --help
```

### Options globales

| Option | Description |
|---|---|
| `--encryption <mode>` | Mode d'opération : `pack` (chiffrer) ou `unpack` (déchiffrer) |
| `--inputdir <dir>` | Répertoire du bundle à traiter (traitement récursif) |
| `--key <key>` | Clé de chiffrement en hexadécimal (16, 24 ou 32 octets pour AES-128/192/256) |
| `--generatekey <n>` | Génère une clé aléatoire de `n` octets (affiche en hexadécimal sur stdout) |
| `--secret-provider <type>` | Utilise un provider de secrets pour résoudre la clé (`filesystem`, `vault`, etc.) |
| `--help` / `-h` | Affiche l'aide et quitte |

### Options du secret provider (mode `--secret-provider`)

#### Options communes

| Option | Défaut | Description |
|---|---|---|
| `--secret-tenant <id>` | `default` | Tenant du secret contenant la clé |
| `--secret-name <name>` | `encryptionKey` | Nom du secret contenant la clé |
| `--connectors-dir <dir>` | — | Répertoire contenant les JARs de connecteurs pour découverte SPI (vault and filesystem secret providers) |
| `--secret-config <config>` | — | Configuration générique au format `key=value,key2=value2` pour configurations avancées |

#### Options spécifiques `filesystem`

| Option | Défaut | Description |
|---|---|---|
| `--root <dir>` | — | Répertoire racine des secrets |
| `--keyfile <path>` | — | Fichier clé maîtresse |

#### Options spécifiques `vault`

| Option | Défaut | Description |
|---|---|---|
| `--vault-address <url>` | — | URL du serveur Vault (ex: https://vault.internal:8200) |
| `--vault-namespace <ns>` | — | Namespace Vault (optionnel) |
| `--vault-kv-mount <mount>` | `secret` | Chemin de montage KV v2 |
| `--vault-auth-method <method>` | `token` | Méthode d'authentification : `token` ou `approle` |
| `--vault-token <token>` | — | Token d'authentification Vault (pour auth `token`) |
| `--vault-approle-mount <mount>` | `approle` | Chemin de montage AppRole |
| `--vault-role-id <id>` | — | Role ID AppRole (pour auth `approle`) |
| `--vault-secret-id <id>` | — | Secret ID AppRole (pour auth `approle`) |
| `--vault-timeout <sec>` | `10` | Timeout des requêtes HTTP vers Vault (secondes) |

#### Configuration via variables d'environnement

Toutes les options peuvent être passées via variables d'environnement :

| Variable | Description |
|---|---|
| `PAYOS_SECRET_PROVIDER_TYPE` | Type de provider (`filesystem`, `vault`, etc.) |
| `PAYOS_SECRET_TENANT` | Tenant du secret |
| `PAYOS_SECRET_NAME` | Nom du secret |
| `PAYOS_SECRET_ROOT` | Répertoire racine (filesystem) |
| `PAYOS_SECRET_KEYFILE` | Fichier clé maîtresse (filesystem) |
| `PAYOS_VAULT_ADDRESS` | URL du serveur Vault |
| `PAYOS_VAULT_NAMESPACE` | Namespace Vault |
| `PAYOS_VAULT_KV_MOUNT` | Chemin de montage KV v2 |
| `PAYOS_VAULT_AUTH_METHOD` | Méthode d'authentification Vault |
| `PAYOS_VAULT_TOKEN` | Token Vault |
| `PAYOS_VAULT_APPROLE_MOUNT` | Chemin de montage AppRole |
| `PAYOS_VAULT_ROLE_ID` | Role ID AppRole |
| `PAYOS_VAULT_SECRET_ID` | Secret ID AppRole |
| `PAYOS_VAULT_TIMEOUT` | Timeout HTTP Vault |
| `PAYOS_SECRET_CONFIG` | Configuration générique (format `key=value,key2=value2`) |
| `PAYOS_SECRET_CFG_*` | Configuration par clé individuelle (ex: `PAYOS_SECRET_CFG_CUSTOM_KEY` → `custom-key`) |
| `PAYOS_CONNECTORS_DIR` | Répertoire des connecteurs |

### Modes d'opération

#### pack — Chiffrer un bundle

Chiffre récursivement tous les fichiers du répertoire `--inputdir`. Chaque fichier chiffré reçoit un magic header `P8OS` (4 octets) suivi des données AES-GCM. Les fichiers déjà chiffrés (détectés via `P8OS`) sont ignorés.

```bash
# Chiffrer avec clé inline
edc --encryption pack --inputdir /opt/payos/bundle --key "0123456789abcdef0123456789abcdef"

# Chiffrer avec clé depuis secret provider filesystem
edc --encryption pack --inputdir /opt/payos/bundle \
  --secret-provider filesystem \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --secret-tenant default \
  --secret-name encryptionKey

# Chiffrer avec clé depuis Vault (authentification token)
edc --encryption pack --inputdir /opt/payos/bundle \
  --secret-provider vault \
  --vault-address https://vault.internal:8200 \
  --vault-auth-method token \
  --vault-token hvs.CAESIJ... \
  --secret-tenant default \
  --secret-name encryptionKey

# Chiffrer avec clé depuis Vault (authentification AppRole)
edc --encryption pack --inputdir /opt/payos/bundle \
  --secret-provider vault \
  --vault-address https://vault.internal:8200 \
  --vault-auth-method approle \
  --vault-role-id "${VAULT_ROLE_ID}" \
  --vault-secret-id "${VAULT_SECRET_ID}" \
  --secret-tenant default \
  --secret-name encryptionKey
```

#### unpack — Déchiffrer un bundle

Déchiffre récursivement tous les fichiers chiffrés (détectés via magic header `P8OS`) dans `--inputdir`. Les fichiers non chiffrés sont ignorés.

```bash
# Déchiffrer avec clé inline
edc --encryption unpack --inputdir /opt/payos/bundle --key "0123456789abcdef0123456789abcdef"

# Déchiffrer avec clé depuis secret provider filesystem
edc --encryption unpack --inputdir /opt/payos/bundle \
  --secret-provider filesystem \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --secret-tenant default \
  --secret-name encryptionKey

# Déchiffrer avec clé depuis Vault (authentification AppRole)
edc --encryption unpack --inputdir /opt/payos/bundle \
  --secret-provider vault \
  --vault-address https://vault.internal:8200 \
  --vault-auth-method approle \
  --vault-role-id "${VAULT_ROLE_ID}" \
  --vault-secret-id "${VAULT_SECRET_ID}" \
  --secret-tenant default \
  --secret-name encryptionKey
```

#### generatekey — Générer une clé

Génère une clé aléatoire de `n` octets et l'affiche en hexadécimal sur stdout. Utile pour générer des clés de chiffrement de bundle.

```bash
# Générer une clé AES-256 (32 octets)
edc --generatekey 32

# Stocker dans une variable
BUNDLE_KEY=$(edc --generatekey 32)
```

### Construction et installation

#### Construction

Depuis le répertoire `payosv2-packer` :

```bash
mvn package -DskipTests
```

Produit dans `target/` :
- `edc.jar` — fat JAR autonome (shade plugin)

#### Installation système (Linux / macOS)

```bash
# Depuis le répertoire payosv2-packer
./install-edc.sh                        # → ~/.payos/bin (défaut)
./install-edc.sh /usr/local/bin         # → répertoire personnalisé
PAYOS_HOME=/opt/payos ./install-edc.sh  # → $PAYOS_HOME/bin
```

Le script copie `edc.jar`, génère le wrapper shell `edc`, et ajoute le répertoire cible au `PATH`.

#### Installation système (Windows)

```powershell
# Depuis le répertoire payosv2-packer
.\install-edc.ps1                                          # → %USERPROFILE%\.payos\bin
.\install-edc.ps1 -InstallDir "C:\Tools\payos\bin"        # → répertoire personnalisé
.\install-edc.ps1 -JarPath "D:\build\edc.jar" -InstallDir "C:\Tools\payos\bin"
```

Le script copie `edc.jar`, génère `edc.ps1` et `edc.cmd`, et met à jour le `PATH` utilisateur.

#### Invocation directe (sans installation)

```bash
# Linux / macOS
java -jar payosv2-packer/target/edc.jar --help

# Windows
java -jar payosv2-packer\target\edc.jar --help
```

### Flux complets

#### Chiffrement d'un bundle pour déploiement sécurisé (filesystem)

```bash
# 1. Générer une clé de chiffrement et la stocker dans spm
BUNDLE_KEY=$(edc --generatekey 32)
echo -n "$BUNDLE_KEY" | spm set --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --tenant default \
  --name encryptionKey \
  --type encryption-key

# 2. Chiffrer le bundle
edc --encryption pack --inputdir /opt/payos/bundle \
  --secret-provider filesystem \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --secret-tenant default \
  --secret-name encryptionKey

# 3. Le bundle est maintenant chiffré en place (magic header P8OS)
# Transférer le bundle chiffré vers le serveur cible

# 4. Sur le serveur cible, déchiffrer avant démarrage
edc --encryption unpack --inputdir /opt/payos/bundle \
  --secret-provider filesystem \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --secret-tenant default \
  --secret-name encryptionKey
```

#### Chiffrement d'un bundle pour déploiement sécurisé (Vault)

```bash
# 1. Générer une clé de chiffrement et la stocker dans Vault
BUNDLE_KEY=$(edc --generatekey 32)
vault kv put secret/default/encryptionKey value="$BUNDLE_KEY"

# 2. Chiffrer le bundle avec Vault AppRole
edc --encryption pack --inputdir /opt/payos/bundle \
  --secret-provider vault \
  --vault-address https://vault.internal:8200 \
  --vault-auth-method approle \
  --vault-role-id "${VAULT_ROLE_ID}" \
  --vault-secret-id "${VAULT_SECRET_ID}" \
  --secret-tenant default \
  --secret-name encryptionKey

# 3. Le bundle est maintenant chiffré en place (magic header P8OS)
# Transférer le bundle chiffré vers le serveur cible

# 4. Sur le serveur cible, déchiffrer avant démarrage
edc --encryption unpack --inputdir /opt/payos/bundle \
  --secret-provider vault \
  --vault-address https://vault.internal:8200 \
  --vault-auth-method approle \
  --vault-role-id "${VAULT_ROLE_ID}" \
  --vault-secret-id "${VAULT_SECRET_ID}" \
  --secret-tenant default \
  --secret-name encryptionKey
```

### Magic Header et détection

`edc` utilise le magic header `P8OS` (4 octets ASCII : `0x50 0x38 0x4F 0x53`) au début de chaque fichier chiffré. Cela permet :
- De détecter automatiquement les fichiers déjà chiffrés (évite le double chiffrement)
- De valider l'intégrité du format lors du déchiffrement
- D'être utilisé par `ConfigLoader` pour déchiffrer automatiquement les configs au démarrage

### Codes de sortie

| Code | Signification |
|---|---|
| `0` | Succès |
| `1` | Erreur opérationnelle (erreur I/O, clé invalide, provider introuvable) |
| `2` | Erreur d'utilisation (option requise manquante, combinaison invalide) |

---

## 10. pdoc — OpenAPI Documentation Generator

`pdoc` est l'outil de génération de documentation OpenAPI 3.1 statique pour les applications, capabilities et produits PayOS. Il scanne les annotations `@payos.openapi` dans les scripts API JavaScript sans exécuter le runtime, garantissant une génération sûre et rapide.

> **Prérequis** : Java 21+. Le module est fourni par `pdoc`.

### Syntaxe générale

```
pdoc --app <id> --bundle-path <path> [--output <dir>] [--format yaml|json]
pdoc --capability <id> --bundle-path <path> [--output <dir>] [--format yaml|json]
pdoc --product <id> --bundle-path <path> [--output <dir>] [--format yaml|json]
pdoc --help
pdoc --version
```

### Options globales

| Option | Défaut | Description |
|---|---|---|
| `--bundle-path <path>` | `.` | Chemin vers le répertoire runtime contenant `payos.json` |
| `--app <id>` | — | ID de l'application à documenter (mutuellement exclusif avec `--capability` et `--product`) |
| `--capability <id>` | — | ID de la capability à documenter |
| `--product <id>` | — | ID du produit à documenter (génère un document par application du produit) |
| `--output <dir>` | `.` | Répertoire de sortie pour les fichiers OpenAPI générés |
| `--format <fmt>` | `yaml` | Format de sortie : `yaml` ou `json` |
| `--help` / `-h` | — | Affiche l'aide et quitte |
| `--version` / `-V` | — | Affiche la version et quitte |

### Annotations @payos.openapi

Les annotations sont des blocs de commentaires JavaScript multi-lignes avec le tag `@payos.openapi` placés **avant** la déclaration de l'endpoint. Format attendu :

```javascript
/**
 * @payos.openapi
 * summary: Create a new payment
 * description: |
 *   Creates a payment transaction for the specified merchant.
 *   Requires valid authentication and merchant authorization.
 * tags:
 *   - Payments
 * requestBody:
 *   required: true
 *   content:
 *     application/json:
 *       schema:
 *         type: object
 *         required: [amount, currency, merchantId]
 *         properties:
 *           amount:
 *             type: number
 *             description: Transaction amount in cents
 *           currency:
 *             type: string
 *             enum: [USD, EUR, MAD]
 *           merchantId:
 *             type: string
 * responses:
 *   201:
 *     description: Payment created successfully
 *     content:
 *       application/json:
 *         schema:
 *           type: object
 *           properties:
 *             id:
 *               type: string
 *             status:
 *               type: string
 *   400:
 *     description: Invalid request
 */

// Endpoint implementation
var amount = JSON.parse($Request.getBody()).amount;
// ...
```

### Génération pour une application

```bash
# Génération YAML (défaut)
pdoc --app payment-gateway --bundle-path /opt/payos --output ./docs

# Génération JSON
pdoc --app payment-gateway --bundle-path /opt/payos --output ./docs --format json
```

Produit : `./docs/payment-gateway.openapi.yaml` (ou `.json`)

### Génération pour une capability

```bash
pdoc --capability payment-links --bundle-path /opt/payos --output ./docs
```

Produit : `./docs/payment-links.openapi.yaml`

### Génération pour un produit

```bash
pdoc --product acquiring --bundle-path /opt/payos --output ./docs
```

Produit un document OpenAPI par application du produit :
- `./docs/acquiring-core.openapi.yaml`
- `./docs/acquiring-portal.openapi.yaml`

### Structure générée (exemple)

```yaml
openapi: 3.1.0
info:
  title: Payment Gateway API
  version: 2.1.0
  description: Payment processing API for merchant transactions
servers:
  - url: http://localhost:8080/payment-gateway
    description: Local development
paths:
  /payments:
    post:
      summary: Create a new payment
      description: |
        Creates a payment transaction for the specified merchant.
        Requires valid authentication and merchant authorization.
      tags:
        - Payments
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [amount, currency, merchantId]
              properties:
                amount:
                  type: number
                  description: Transaction amount in cents
                currency:
                  type: string
                  enum: [USD, EUR, MAD]
                merchantId:
                  type: string
      responses:
        '201':
          description: Payment created successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: string
                  status:
                    type: string
        '400':
          description: Invalid request
```

### Construction et installation

#### Construction

Depuis le répertoire `pdoc` :

```bash
mvn package -DskipTests
```

Produit dans `target/` :
- `pdoc.jar` — fat JAR autonome

#### Installation système (Linux / macOS)

```bash
# Depuis le répertoire pdoc
cp target/pdoc.jar scripts/
./scripts/install-pdoc-tools.sh                        # → ~/.payos/bin (défaut)
./scripts/install-pdoc-tools.sh /usr/local/bin        # → répertoire personnalisé
PAYOS_HOME=/opt/payos ./scripts/install-pdoc-tools.sh # → $PAYOS_HOME/bin
```

#### Installation système (Windows)

```powershell
# Depuis le répertoire pdoc
Copy-Item target\pdoc.jar scripts\
.\scripts\install-pdoc-tools.ps1                                          # → %USERPROFILE%\.payos\bin
.\scripts\install-pdoc-tools.ps1 -InstallDir "C:\Tools\payos\bin"        # → répertoire personnalisé
```

#### Invocation directe (sans installation)

```bash
# Linux / macOS
java -jar pdoc/target/pdoc.jar --help

# Windows
java -jar pdoc\target\pdoc.jar --help
```

### Garanties de sécurité

`pdoc` offre une **garantie runtime-safety** :
- ✅ Aucun script n'est exécuté
- ✅ Aucune connexion base de données n'est ouverte
- ✅ Aucun service externe n'est appelé
- ✅ Analyse purement lexicale des annotations

Cela permet de générer la documentation sur des environnements CI/CD restreints sans risque d'effet de bord.

### Codes de sortie

| Code | Signification |
|---|---|
| `0` | Succès |
| `1` | Erreur opérationnelle (application introuvable, erreur I/O, parsing YAML invalide) |
| `2` | Erreur d'utilisation (option requise manquante, combinaison invalide) |

---

## 11. Configuration des catalogues

`cpm`, `apm` et `ppm` supportent le téléchargement depuis un catalogue configuré dans `payos.json`.

### Catalogue de capabilities (`capabilityCatalog`)

Utilisé par `cpm --from-catalog`.

```json
{
  "capabilityCatalog": {
    "type": "local",
    "path": "/opt/payos/catalog/capabilities"
  }
}
```

```json
{
  "capabilityCatalog": {
    "type": "git",
    "baseUrl": "https://git.example.com/payos/capabilities"
  }
}
```

Layout attendu pour un catalogue local :
```
/opt/payos/catalog/capabilities/
  payment-links/            ← version non-versionnée (latest)
    manifest.json
    ...
  payment-links/1.2.0/      ← version spécifique
    manifest.json
    ...
```

### Catalogue d'applications (`applicationCatalog`)

Utilisé par `apm` (résolution depuis un nom ou un `id@version`) et par `ppm` (pour les applications du produit).

```json
{
  "applicationCatalog": {
    "type": "git",
    "baseUrl": "https://git.example.com/payos/applications"
  }
}
```

Pour `type=git`, le `baseUrl` est la racine de base. L'URL d'une application est `{baseUrl}/{appId}` (dépôt Git par application).

### Catalogue de produits (`productCatalog`)

Utilisé par `ppm` (résolution depuis un nom ou un `name@version`).

```json
{
  "productCatalog": {
    "type": "local",
    "path": "/opt/payos/catalog/products"
  }
}
```

```json
{
  "productCatalog": {
    "type": "git",
    "baseUrl": "https://git.example.com/payos/products"
  }
}
```

---

## 12. Tableau comparatif

| Dimension | `cpm` | `apm` | `ppm` | `spm` | `edc` | `pdoc` |
|---|---|---|---|---|---|---|
| Unité gérée | Capability | Application unique | Produit (bundle d'apps) | Secret chiffré | Bundle chiffré | Documentation OpenAPI |
| Option principale | `--id <id>` | `--app <id[@v]\|path>` | `--product <name[@v]\|path>` | `--name <name>` + `--tenant <id>` | `--inputdir <dir>` + `--encryption pack\|unpack` | `--app <id>` ou `--capability <id>` ou `--product <id>` |
| Fichier manifest | `<sourceDir>/manifest.json` | `manifest.json` dans le répertoire spécifié ou catalogue | `<id>.json` dans catalogue ou répertoire | — | — | `manifest.json` (lecture) |
| Clé catalogue dans `payos.json` | `capabilityCatalog` | `applicationCatalog` | `productCatalog` + `applicationCatalog` | — | — | — |
| Écrit dans | `.capabilities/`, `bootstrap.json`, `registry.json`, `activation.json`, `events.ndjson` | `bootstrap.json` uniquement | `bootstrap.json` uniquement | `{root}/{tenantId}/{name}.enc` + `.meta.json` | Fichiers en place (magic header `P8OS`) | `{output}/*.yaml` ou `*.json` |
| Hooks de cycle de vie | Oui (`hooks/lifecycle.js`) | Non | Non | — | — | — |
| Résolution de dépendances | Oui (transitive) | Non | Non | — | — | — |
| Tracking contributeur | Non | Non | Oui (`_contributors[]`) | — | — | — |
| Fallback sans manifest | Non | Oui (`--base-path`) | Non | — | — | — |
| Commandes disponibles | `install`, `uninstall`, `activate`, `deactivate`, `status` | `install`, `uninstall`, `status` | `install`, `uninstall`, `status` | `keygen`, `set`, `get`, `list`, `describe`, `delete` | `pack`, `unpack`, `generatekey` | Aucune (argument principal) |
| Module source | `payos-pm` | `payos-pm` | `payos-pm` | `secret-service-filesystem` | `payosv2-packer` | `pdoc` |
| Scope | Gestion du runtime | Gestion du runtime | Gestion du runtime | Administration des secrets | Chiffrement de bundle | Génération de documentation |

---

## 13. Codes d'erreur

| Code de sortie | Signification |
|---|---|
| `0` | Succès |
| `1` | Erreur opérationnelle (échec de hook, incohérence d'état, erreur I/O) |
| `2` | Erreur d'utilisation (option requise manquante, combinaison d'options invalide) |

---

## Exemples de flux complets

### Créer et enregistrer une nouvelle application

```bash
# 1. Générer le squelette
./generate_app.sh --app-id payment-portal --output /opt/payos/apps

# 2. Enregistrer dans le runtime
apm --bundle /opt/payos --install --app /opt/payos/apps/payment-portal
```

### Installer et activer une capability sur un tenant spécifique

```bash
# 1. Installer depuis un package local
cpm --bundle /opt/payos --install --id payment-links --path ./packages/payment-links

# 2. L'auto-activation globale a eu lieu — désactiver si nécessaire
cpm --bundle /opt/payos --deactivate --id payment-links

# 3. Activer uniquement pour l'app "merchant-portal" et le tenant "bank-a"
cpm --bundle /opt/payos --activate --id payment-links --app merchant-portal --tenant bank-a
```

### Déployer un produit complet depuis un catalogue Git

```bash
# payos.json doit contenir productCatalog et applicationCatalog configurés
ppm --bundle /opt/payos --install --product acquiring@3.0.0
```

### Enregistrer une application déjà déployée sur disque

```bash
# L'app existe déjà dans /opt/payos/apps/my-app mais n'est pas encore dans bootstrap.json
apm --bundle /opt/payos --install --app my-app --base-path /opt/payos/apps/my-app
```

### Désinstaller un produit puis nettoyer une capability dépendante en cascade

```bash
ppm --bundle /opt/payos --uninstall --product acquiring
cpm --bundle /opt/payos --uninstall --id payment-links --cascade --drop-schema
```

### Vérifier l'état d'une capability avant activation

```bash
cpm --bundle /opt/payos --status --id payment-links
```
