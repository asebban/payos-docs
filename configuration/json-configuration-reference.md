# PayOS — Référence complète de la configuration JSON

> Source canonique : `IConfigSpec.java` + fichiers de configuration du runtime  
> Dernière mise à jour : 2026-06-03

---

## Table des matières

- [PayOS — Référence complète de la configuration JSON](#payos--référence-complète-de-la-configuration-json)
  - [Table des matières](#table-des-matières)
  - [1. Vue d'ensemble et séquence de chargement](#1-vue-densemble-et-séquence-de-chargement)
  - [2. payos.json — Point d'entrée](#2-payosjson--point-dentrée)
    - [`capabilityCatalog`](#capabilitycatalog)
    - [`applicationCatalog`](#applicationcatalog)
    - [`productCatalog`](#productcatalog)
  - [3. bootstrap.json — Configuration racine](#3-bootstrapjson--configuration-racine)
    - [3.1 `servers`](#31-servers)
    - [3.2 `applications`](#32-applications)
      - [Clés racine d'une application](#clés-racine-dune-application)
      - [Bloc `security` d'une application](#bloc-security-dune-application)
      - [Bloc `database-service` d'une application](#bloc-database-service-dune-application)
    - [3.3 `security` (globale)](#33-security-globale)
    - [3.4 `multitenancy`](#34-multitenancy)
      - [Clés racine `multitenancy`](#clés-racine-multitenancy)
      - [`tenantSimulator`](#tenantsimulator)
      - [`default-tenant-quotas`](#default-tenant-quotas)
      - [`tenants` — configuration par tenant](#tenants--configuration-par-tenant)
        - [Modes d'isolation](#modes-disolation)
    - [3.5 `database-service` (globale)](#35-database-service-globale)
      - [`database-service.configuration`](#database-serviceconfiguration)
    - [3.6 `queue-service`](#36-queue-service)
    - [3.7 `webhooks` (bootstrap.json)](#37-webhooks-bootstrapjson)
    - [3.8 `webhooks.json` (par application)](#38-webhooksjson-par-application)
    - [3.9 `config/i18n.json` (par application)](#39-configi18njson-par-application)
    - [3.10 `swagger-ui` (bootstrap.json)](#310-swagger-ui-bootstrapjson)
    - [3.11 `http-webhook-service` (bootstrap.json)](#311-http-webhook-service-bootstrapjson)
    - [3.12 `secret-service` (bootstrap.json)](#312-secret-service-bootstrapjson)
    - [3.13 `extensions-dir` (bootstrap.json)](#313-extensions-dir-bootstrapjson)
    - [3.14 `notification-service` (bootstrap.json)](#314-notification-service-bootstrapjson)
  - [4. manifest.json — Déclaration de capability](#4-manifestjson--déclaration-de-capability)
  - [5. Résolution hiérarchique des valeurs](#5-résolution-hiérarchique-des-valeurs)
  - [6. Constantes et valeurs par défaut](#6-constantes-et-valeurs-par-défaut)
  - [7. Exemples complets](#7-exemples-complets)
    - [Déploiement mono-tenant, HTTP simple](#déploiement-mono-tenant-http-simple)
    - [Déploiement multi-tenant, HTTPS + OIDC + NATS](#déploiement-multi-tenant-https--oidc--nats)

---

## 1. Vue d'ensemble et séquence de chargement

```
payos.json
  └─ configDir  →  {configDir}/*.json  (bootstrap.json et tout autre fichier JSON du dossier)
                       └─ applications[].base.path/config/*.json  (config par application)
```

Chaque couche **fusionne** avec la suivante ; la couche la plus proche de l'application a la priorité. Les fichiers sont surveillés à chaud via `ConfigWatcher` : toute modification déclenche un rechargement sans redémarrage.

---

## 2. payos.json — Point d'entrée

Fichier racine du runtime. Son rôle est **double** selon le lecteur :

- **Le runtime** (`BootServer`) ne lit que `configDir`. Il charge ensuite l'intégralité de la configuration depuis les fichiers `*.json` présents dans ce répertoire. 
- **Les outils `cpm` / `ppm`** lisent les clés de catalogues (`capabilityCatalog`, `applicationCatalog`, `productCatalog`) pour résoudre les packages à installer, ainsi que `bundle-id` pour identifier l'instance cible. Un bundle est une configuration de produits et d'applications 

```json
{
  "configDir": "./config",
  "bundle-id": "payos-node-1",
  "capabilityCatalog": {
    "type": "git",
    "baseUrl": "https://git.example.com/capabilities"
  },
  "applicationCatalog": {
    "type": "git",
    "baseUrl": "https://git.example.com/applications"
  },
  "productCatalog": {
    "type": "local",
    "baseUrl": "/opt/payos/products"
  }
}
```

| Clé | Lue par | Type | Défaut | Description |
|-----|---------|------|--------|-------------|
| `configDir` | Runtime + cpm | string | `"."` | Chemin absolu ou relatif vers le dossier contenant `bootstrap.json` et les autres fichiers de configuration |
| `bundle-id` | cpm / ppm | string | — | Identifiant unique de cette instance du runtime. Utilisé pour distinguer les nœuds dans un déploiement multi-instances |
| `capabilityCatalog` | cpm | object | — | Catalogue de capabilities utilisé par `cpm --from-catalog` (voir ci-dessous) |
| `applicationCatalog` | cpm / ppm | object | — | Catalogue d'applications. Même structure que `capabilityCatalog` (voir ci-dessous) |
| `productCatalog` | ppm | object | — | Catalogue de produits (bundles d'applications et capabilities). Même structure que `capabilityCatalog` (voir ci-dessous) |
| `capabilityRepository` | cpm | object | — | *(Obsolète — utiliser `capabilityCatalog`)* Alias conservé pour compatibilité ascendante |

### `capabilityCatalog`

Configure le catalogue distant ou local depuis lequel `cpm` installe et met à jour les capabilities. 

- En cas d'utilisation de git, seul le chemin git de base du repository est spécifié ici (chemin du groupe de repositories ou organisation qui regroupe tous les repositories de type capability). 
- En cas de type=local, cela veut dire qu'il faut fournir l'attribut `path` qui spécifie le répertoire racine 

> **Note de compatibilité :** La clé `capabilityRepository` est conservée comme alias de `capabilityCatalog` pour les fichiers de configuration existants.

```json
"capabilityCatalog": {
  "type": "git",
  "baseUrl": "https://git.example.com/payos/capabilities.git"
}
```

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `type` | string | `"local"` | Type de catalogue : `"local"` (copie depuis un dossier local) ou `"git"` (clone depuis un dépôt distant) |
| `path` | string | — | Requis si `type=local`. Chemin racine du catalogue. Structure attendue : `{path}/{capabilityId}/` ou `{path}/{capabilityId}/{version}/` |
| `baseUrl` | string | — | Requis si `type=git`. URL racine du dépôt Git. La version est résolue via les tags Git. |

### `applicationCatalog`

Même structure que `capabilityCatalog`. Configure le dépôt distant ou local des bundles d'applications. Même philosophie que le catalogue des capabilities. En cas de type=git, seul le chemin de base des repositories des application est fourni.

```json
"applicationCatalog": {
  "type": "git",
  "baseUrl": "https://git.example.com/applications"
}
```

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `type` | string | `"local"` | `"local"` ou `"git"` |
| `path` | string | — | Requis si `type=local`. Chemin racine du catalogue d'applications. |
| `baseUrl` | string | — | Requis si `type=git`. Racine de base de l'URL du dépôt Git des applications. |

### `productCatalog`

Configure le dépôt de produits (bundles combinant applications et capabilities).

```json
"productCatalog": {
  "type": "local",
  "path": "/opt/payos/products"
}
```

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `type` | string | `"local"` | `"local"` ou `"git"` |
| `path` | string | — | Requis si `type=local`. Chemin racine du catalogue de produits. |
| `url` | string | — | Requis si `type=git`. URL du dépôt Git de produits. |

---

## 3. bootstrap.json — Configuration racine

Fichier principal du runtime. Toutes les sections sont optionnelles sauf `servers` (au moins un serveur doit être déclaré).

### 3.1 `servers`

Tableau de serveurs à démarrer. Chaque entrée déclare un point d'écoute.

```json
"servers": [
  {
    "host": "0.0.0.0",
    "port": 8080,
    "protocol": "http",
    "keystore": "/etc/payos/keystore.p12",
    "keystorePassword": "changeit",
    "keyPassword": "changeit",
    "keystoreType": "PKCS12",
    "keyAlias": "payos"
  },
  {
    "host": "0.0.0.0",
    "port": 9090,
    "protocol": "tcp",
    "tcp-handlers-dir": "/opt/payos/tcp-handlers"
  },
  {
    "protocol": "queue",
    "type": "nats",
    "consumer-topic": "payos.requests"
  }
]
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `host` | string | Non | `"127.0.0.1"` | Adresse IP d'écoute |
| `port` | int | Oui (HTTP/TCP) | — | Port d'écoute |
| `protocol` | string | Oui | — | `"http"`, `"tcp"`, ou `"queue"` |
| `tcp-handlers-dir` | string | Non | — | Dossier contenant les handlers TCP (JAR ou scripts). Surcharge la valeur globale `tcp-handlers-dir` |
| `type` | string | Non | `"nats"` | Type de client MoM pour le serveur queue |
| `consumer-topic` | string | Non | — | Sujet NATS sur lequel le serveur écoute les requêtes entrantes |
| `keystore` | string | Non | — | Chemin vers le fichier keystore pour TLS/HTTPS |
| `keystorePassword` | string | Non | — | Mot de passe du keystore |
| `keyPassword` | string | Non | — | Mot de passe de la clé privée dans le keystore |
| `keystoreType` | string | Non | — | Type de keystore : `"JKS"` ou `"PKCS12"` |
| `keyAlias` | string | Non | — | Alias de la clé dans le keystore |

> **Note TLS :** Les champs `keystore*` s'appliquent uniquement au protocole `http`. Un serveur HTTP avec `keystore` configuré devient automatiquement HTTPS.

---

### 3.2 `applications`

Tableau des applications hébergées par le runtime.

```json
"applications": [
  {
    "id": "payment-gateway",
    "name": "Payment Gateway",
    "base.path": "/opt/payos/apps/payment-gateway",
    "version": "2.1.0",
    "authorized.tenants": ["acme", "globex"],
    "mapping-files": ["model/gateway.hbm.xml"],
    "security": {
      "provider": "nimbus",
      "clientId": "gateway-client",
      "clientSecret": "s3cr3t",
      "oidcProviderBaseUrl": "https://auth.example.com",
      "realm": "payos",
      "scope": "openid profile email",
      "sessionTtlSeconds": 3600,
      "sessionCookieSecure": true,
      "allowedOrigins": ["https://app.example.com"]
    },
    "database-service": {
      "name": "gateway_db",
      "mapping-files": ["model/gateway.hbm.xml"],
      "configuration": {
        "url": "jdbc:postgresql://db.example.com:5432/gateway",
        "username": "gw_user",
        "password": "gw_pass",
        "dialect": "org.hibernate.dialect.PostgreSQLDialect",
        "ddl-auto": "validate",
        "schema": "gateway",
        "max-pool-size": 10,
        "minimum-idle": 2
      }
    }
  }
]
```

#### Clés racine d'une application

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `id` | string | Oui | — | Identifiant unique de l'application (utilisé pour le routage URI) |
| `name` | string | Oui | — | Nom lisible de l'application |
| `base.path` | string | Non | `"./apps/{id}"` | Dossier racine de l'application |
| `version` | string | Non | — | Version de l'application |
| `authorized.tenants` ou `authorized-tenants` | array[string] | Non | Tous les tenants | Liste des `tenantId` autorisés à accéder à cette application |
| `mapping-files` | array[string] | Non | `model/*.xml` | Fichiers HBM Hibernate relatifs à l'application (charge les entités métier de l'application) |
| `security` | object | Non | Hérité du global | Configuration de sécurité spécifique à l'application (voir ci-dessous) |
| `database-service` | object | Non | Hérité du global | Configuration de source de données spécifique à l'application (voir ci-dessous) |
| `extends` | array[string] | Non | `[]` | Liste des `id` de capabilities actives à inclure dans l'application. Géré automatiquement par `cpm`. |
| `category` | string | Non | — | Marqueur de type : `"capability"` pour les entrées gérées par `cpm`. Ne pas définir manuellement. |

#### Bloc `security` d'une application

Identique au [bloc `security` global](#33-security-globale). Les valeurs ici surchargent le global pour cette application uniquement.

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `provider` | string | — | Fournisseur de sécurité : `"nimbus"` (recommandé) ou `"pac4j"` |
| `oidcProviderBaseUrl` | string | — | URL de base du fournisseur OIDC (ex. Keycloak). Utilisée pour construire `discoveryUri` si absent |
| `realm` | string | — | Nom du realm Keycloak |
| `discoveryUri` | string | `http://localhost:8080/realms/master/.well-known/openid-configuration` | URI de découverte OIDC complète. Surcharge la construction depuis `oidcProviderBaseUrl` + `realm` |
| `clientId` | string | Requis si security activé | — | Client ID OIDC |
| `clientSecret` | string | Requis si security activé | — | Client secret OIDC (PCI-DSS Req 2.1 : les valeurs par défaut sont rejetées au démarrage) |
| `callBackUri` | string | `"/callback"` | URI de callback OAuth2 |
| `scope` | string | — | Scopes OIDC demandés (ex. `"openid profile email"`) |
| `preferredJwsAlgorithm` | string | — | Algorithme JWS pour validation des tokens : `"RS256"`, `"HS256"`, etc. |
| `logoutUrl` | string | — | URL de logout personnalisée |
| `postLogoutRedirectUri` | string | — | URI de redirection après logout |
| `sessionTtlSeconds` | int | `1800` (30 min) | Durée de vie d'une session en secondes |
| `sessionMaxEntries` | int | `10000` | Nombre maximum de sessions simultanées en mémoire |
| `sessionCookieSecure` | boolean | `false` | Cookie de session transmis uniquement en HTTPS (`Secure` flag) |
| `allowedOrigins` | array[string] | — | Origins autorisées pour CORS |

#### Bloc `database-service` d'une application

Identique à la [section `database-service` globale](#35-database-service-globale). Les valeurs ici surchargent le global pour cette application uniquement.

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `name` | string | Dérivé de l'URL | Nom logique de l'instance BDD propre à l'application |
| `mapping-files` | array[string] | `model/*.xml` | Fichiers HBM Hibernate relatifs à `base.path` de l'application |
| `configuration` | object | Hérité du global | Paramètres de connexion JDBC/HikariCP (mêmes clés que [`database-service.configuration`](#database-serviceconfiguration)) |

> **Priorité :** Un bloc `database-service` au niveau de l'application prend le dessus sur le bloc global et sur la configuration du tenant courant pour toutes les requêtes routées vers cette application.

---

### 3.3 `security` (globale)

Même structure que le bloc `security` d'une application. S'applique à toutes les applications qui n'ont pas de bloc `security` propre et se situe à la racine de la configuration.

```json
"security": {
  "provider": "nimbus",
  "clientId": "payos-global",
  "clientSecret": "s3cr3t",
  "oidcProviderBaseUrl": "https://auth.example.com",
  "realm": "master",
  "scope": "openid",
  "sessionCookieSecure": true,
  "logoutUrl": "/logout"
}
```

Voir le tableau des [clés `security`](#bloc-security-dune-application) ci-dessus — toutes les clés sont identiques.

---

### 3.4 `multitenancy`

Contrôle l'isolation multi-tenant et les quotas.

```json
"multitenancy": {
  "requireTenantId": true,
  "enforceForRequestTypes": ["api", "page", "component"],
  "default-database-schema": "public",
  "default-isolation-mode": "shared-schema",
  "tenantSimulator": {
    "enabled": false,
    "tenantId": "demo-tenant"
  },
  "default-tenant-quotas": {
    "enabled": true,
    "requestsPerMinute": 1000
  },
  "tenants": {
    "default": {
      "schema": "public",
      "isolationMode": "shared-schema"
    },
    "tenant1": {
      "schema": "tenant1",
      "isolationMode": "dedicated-schema",
      "security": { },
      "database-service": { },
      "quota": { "requestsPerMinute": 500 }
    },
    "tenant3": {
      "isolationMode": "dedicated-database",
      "database-service": {
        "name": "tenant3_db",
        "configuration": {
          "url": "jdbc:postgresql://db3:5432/tenant3",
          "username": "t3user",
          "password": "t3pass"
        }
      }
    }
  }
}
```

#### Clés racine `multitenancy`

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `requireTenantId` | boolean | `true` | Exiger le header `X-Tenant-Id` sur toutes les requêtes |
| `enforceForRequestTypes` | array[string] | `["api","page","component"]` | Types de requêtes pour lesquels le tenant est obligatoire |
| `default-database-schema` | string | `"public"` | Schéma de base de données par défaut pour tous les tenants |
| `default-isolation-mode` | string | `"shared-schema"` | Mode d'isolation par défaut (voir ci-dessous) |

#### `tenantSimulator`

Outil de développement permettant de simuler un tenant sans envoyer le header.

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `enabled` | boolean | `false` | Activer le simulateur (à ne jamais activer en production) |
| `tenantId` | string | — | Tenant simulé |

#### `default-tenant-quotas`

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `enabled` | boolean | `false` | Activer l'application des quotas |
| `requestsPerMinute` | int | `1000` | Limite globale de requêtes par minute (s'applique à chaque tenant sauf surcharge) |

#### `tenants` — configuration par tenant

Chaque clé est un `tenantId`. La clé `"default"` sert de modèle pour les tenants non déclarés.

| Clé | Type | Défaut | Description |
|-----|------|--------|-------------|
| `schema` | string | `"public"` | Schéma SQL du tenant |
| `isolationMode` | string | `"shared-schema"` | Mode d'isolation : `"shared-schema"`, `"dedicated-schema"`, `"dedicated-database"` |
| `security` | object | Hérité du global | Configuration de sécurité spécifique au tenant (mêmes clés que la section `security`) |
| `database-service` | object | Hérité du global | Configuration BDD spécifique au tenant (mêmes clés que la section `database-service`) |
| `quota` | object | — | Quota spécifique : `{ "enabled": true, "requestsPerMinute": 500 }` |

##### Modes d'isolation

| Mode | Description |
|------|-------------|
| `"shared-schema"` | Tous les tenants partagent le même schéma (colonne `tenant_id` dans les tables) |
| `"dedicated-schema"` | Chaque tenant dispose de son propre schéma dans la même base de données |
| `"dedicated-database"` | Chaque tenant dispose de sa propre instance de base de données |

> **Format héritage (compatibilité) :** Le bloc `quotas` (sans le préfixe `default-`) surcharge le bloc `default-quota` pour un tenant particulier.

---

### 3.5 `database-service` (globale)

Configuration complète d'une source de données Hibernate/HikariCP partagée.

```json
"database-service": {
  "name": "payos_db",
  "configuration": {
    "url": "jdbc:postgresql://localhost:5432/payos",
    "username": "payos_user",
    "password": "s3cr3t",
    "driver_class": "org.postgresql.Driver",
    "dialect": "org.hibernate.dialect.PostgreSQLDialect",
    "ddl-auto": "validate",
    "schema": "public",
    "max-pool-size": 20,
    "minimum-idle": 5,
    "retired-session-factory-close-delay-seconds": 30
  }
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `name` | string | Non | Dérivé de l'URL | Nom logique de l'instance BDD |

#### `database-service.configuration`

| Clé | Alias acceptés | Type | Requis | Défaut | Description |
|-----|----------------|------|--------|--------|-------------|
| `url` | `jdbcUrl`, `hibernate.connection.url` | string | Oui | — | URL JDBC de connexion |
| `username` | `user`, `hibernate.connection.username` | string | Oui | — | Utilisateur BDD |
| `password` | `hibernate.connection.password` | string | Non | `""` | Mot de passe BDD |
| `driver_class` | `driverClass`, `hibernate.connection.driver_class` | string | Non | Auto-détecté | Classe du driver JDBC. Note : `driver-class` (avec tiret) n'est **pas** reconnu malgré son apparence similaire — seule la clé `driver_class` (underscore) et ses alias ci-dessus sont lus par `DatabaseServiceInitializer`. |
| `dialect` | `hibernate.dialect` | string | Non | — | Dialecte Hibernate SQL |
| `ddl-auto` | `hibernate.hbm2ddl.auto` | string | Non | Aucun (pas de gestion automatique du schéma sauf configuration explicite) | Stratégie DDL : `validate`, `update`, `create`, `create-drop` |
| `schema` | — | string | Non | `"public"` | Schéma SQL utilisé pour les entités |
| `max-pool-size` | `maximumPoolSize`, `maxPoolSize`, `pool-max` | int | Non | HikariCP défaut | Taille maximale du pool de connexions |
| `minimum-idle` | `minimumIdle`, `minIdle`, `pool-min` | int | Non | HikariCP défaut | Nombre minimum de connexions inactives |
| `retired-session-factory-close-delay-seconds` | — | int | Non | `30` | Délai avant fermeture d'une session factory remplacée (rechargement de config) |

---

### 3.6 `queue-service`

Configuration du client de messagerie asynchrone (MoM). L'implémentation NATS est dans le module séparé `queue-service-nats`.

```json
"queue-service": {
  "configuration": {
    "enabled": true,
    "type": "nats",
    "name": "nats-payos",
    "host": "nats.internal",
    "port": 4222,
    "publisher-topic": "payos.out",
    "consumer-topic": "payos.in"
  }
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `enabled` | boolean | Non | `false` | Activer le service de queue |
| `type` | string | Non | `"nats"` | Type de client MoM |
| `name` | string | Non | Dérivé du type | Nom logique de l'instance |
| `host` | string | Non | `"localhost"` | Hôte du serveur de messagerie |
| `port` | int | Non | `4222` | Port du serveur de messagerie |
| `publisher-topic` | string | Oui si enabled | — | Sujet de publication des messages sortants |
| `consumer-topic` | string | Non | — | Sujet d'écoute des messages entrants |

> **Injection dans les scripts :** Une fois configuré, le client queue est disponible dans les scripts JavaScript sous `$Queue`. Pattern identique à `$DB`.

---

### 3.7 `webhooks` (bootstrap.json)

Configuration globale du dispatcher de webhooks. Le champ `dispatcher` sélectionne l'implémentation active et détermine le bloc de configuration technique à charger (`{type}-webhook-service`). Si cette section est absente ou si `dispatcher` est omis, le type `http` est utilisé par défaut.

```json
"webhooks": {
  "dispatcher": "http",
  "timeout": 5000,
  "deadLetterTopic": "payos.webhook.dead-letter"
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `dispatcher` | string | Non | `"http"` | Implémentation du dispatcher : `"http"` (livraison HTTP POST signée) ou `"queue"` (publication MoM) |
| `timeout` | int | Non | `5000` | Timeout en millisecondes pour chaque appel de livraison HTTP |
| `deadLetterTopic` | string | Non | — | Topic MoM vers lequel les événements non livrables (toutes tentatives épuisées) sont publiés |

> **Convention de nommage :** La valeur de `dispatcher` détermine le bloc de configuration technique chargé : `{type}-webhook-service` (ex. `dispatcher: "http"` → bloc `http-webhook-service`, section [3.11](#311-http-webhook-service-bootstrapjson)).

> **Injection dans les scripts :** Une fois activé, `$WebHooks` est disponible dans tous les scripts JavaScript (API et hooks). Voir [3.8](#38-webhooksjson-par-application) pour les abonnements par application.

---

### 3.8 `webhooks.json` (par application)

Fichier de configuration des abonnements webhooks déclarés à la racine d'une application : `{app-base-path}/webhooks.json`.

**Exemple complet :**

```json
[
  {
    "id": "payment-created-ext",
    "event": "payment.created",
    "native": false,
    "url": "https://erp.example.com/webhooks/payments",
    "secret": "hmac-signing-secret",
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

#### Champs d'un abonnement webhook

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `id` | string | Oui | Identifiant unique de l'abonnement (utile pour le débogage et les logs) |
| `event` | string | Oui | Nom de l'événement écouté. Libre pour `native=false` ; doit être un événement système connu pour `native=true` |
| `native` | boolean | Non | `false` | `false` = déclenché par `$WebHooks.emit()` dans le script ; `true` = déclenché automatiquement par le kernel après chaque requête |
| `url` | string | Oui | URL de destination du webhook |
| `secret` | string | Non | Secret HMAC-SHA256 utilisé pour signer le body de chaque appel (header `X-PayOS-Signature`) |
| `headers` | object | Non | En-têtes HTTP supplémentaires à ajouter à chaque appel (ex. `{"Authorization": "Bearer token"}`) |
| `filter` | object | Non | Filtre de déclenchement (voir ci-dessous) |
| `retry` | object | Non | Politique de retry (voir ci-dessous) |

#### Champs du filtre (`filter`)

| Champ | Type | Description |
|-------|------|-------------|
| `path` | string | Filtre sur le chemin de la requête (ex. `"/payments"`). Supporte les préfixes avec `*` |
| `method` | string | Filtre sur la méthode HTTP (ex. `"POST"`) |
| `statusCodes` | array[int] | Filtre sur les codes de statut HTTP de la réponse (ex. `[200, 201]`). Appliqué uniquement pour `native=true` au moment du dispatch natif |

#### Champs du retry (`retry`)

| Champ | Type | Défaut | Description |
|-------|------|--------|-------------|
| `maxAttempts` | int | `1` | Nombre maximum de tentatives (incluant la première) |
| `backoffMs` | int | `1000` | Délai entre deux tentatives en millisecondes |

> **Contrainte `native=true` :** L'événement doit appartenir à l'ensemble des événements système connus (`api.*`, `page.*`, `security.*`, `capability.*`). Toute entrée invalide est rejetée au chargement avec un log WARN. Voir la catalogue complet dans [hooks-webhooks.md](../../architects/hooks-webhooks.md).
>
> **Règle de déduplication :** Si un script émet (`$WebHooks.emit()`) le même nom d'événement qu'un événement `native=true`, le kernel supprime son propre dispatch automatique pour les entrées dont le filtre complet (chemin + méthode + `statusCodes`) correspond à la requête/réponse courante.

---

### 3.9 `config/i18n.json` (par application)

Fichier de configuration de la localisation côté serveur. Il est placé dans `{app-base-path}/config/i18n.json` et consommé par `$I18n` dans les scripts API et hooks.

```json
{
  "defaultLocale": "fr",
  "fallbackLocale": "en",
  "supportedLocales": ["fr-FR", "fr", "en", "ar-MA"],
  "missingKeyMode": "bracket",
  "headerName": "Accept-Language",
  "overrideHeaderName": "X-Locale"
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `defaultLocale` | string | Non | `"en"` | Locale utilisée si la requête ne fournit aucune préférence exploitable |
| `fallbackLocale` | string | Non | `"en"` | Locale consultée quand une clé manque dans la locale demandée |
| `supportedLocales` | array[string] | Non | toutes | Liste blanche optionnelle des locales acceptées |
| `missingKeyMode` | string | Non | `"key"` | Rendu d'une clé absente : `"key"`, `"empty"`, ou `"bracket"` |
| `headerName` | string | Non | `"Accept-Language"` | Header de négociation linguistique standard |
| `overrideHeaderName` | string | Non | `"X-Locale"` | Header prioritaire permettant de forcer la locale de la requête |

Les messages sont stockés dans des dossiers de locale : `{app-base-path}/i18n/{locale}/*.json`. Tous les fichiers JSON directement sous un dossier de locale sont triés par nom puis fusionnés dans un bundle logique.

```text
app1/
  config/
    i18n.json
  i18n/
    fr/
      common.json
      orders.json
    en/
      common.json
```

La configuration et les messages suivent la chaîne `extends` : les parents sont fusionnés d'abord, puis l'application locale surcharge les clés héritées. Les capabilities ne contribuent que lorsqu'elles sont actives pour l'application et le tenant courants.

> **Guide développeur :** voir [server-side-i18n-js-guide.md](../guides/server-side-i18n-js-guide.md) pour les exemples `$I18n` et les règles de résolution de locale.

---

### 3.10 `swagger-ui` (bootstrap.json)

Configuration de l'intégration Swagger UI (documentation interactive OpenAPI exposée par le runtime).

```json
"swagger-ui": {
  "dist-dir": "/path/to/swagger-ui-dist",
  "openapi-yaml": "/path/to/openapi.yaml",
  "local-only": true
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `dist-dir` | string | Non | `<runtime>/node_modules/swagger-ui-dist` | Dossier contenant les fichiers statiques Swagger UI |
| `openapi-yaml` | string | Non | `<runtime>/openapi.yaml` | Chemin du fichier OpenAPI YAML exposé sur `/openapi.yaml` |
| `local-only` | boolean | Non | `true` | Si `true`, l'interface Swagger UI n'est accessible que depuis `localhost` |

> **Note :** Swagger UI n'est initialisé que si la clé `swagger-ui` est présente dans la configuration. Si `local-only: true`, toute tentative d'accès depuis une IP non-locale reçoit une réponse `403`.

---

### 3.11 `http-webhook-service` (bootstrap.json)

Configuration technique du dispatcher HTTP de webhooks (module `webhook-service-http`). Ce bloc est chargé automatiquement lorsque `webhooks.dispatcher = "http"` (valeur par défaut). Si ce bloc est absent, le dispatcher HTTP est actif avec les valeurs par défaut.

```json
"http-webhook-service": {
  "enabled": true,
  "connectTimeoutMs": 5000,
  "requestTimeoutMs": 10000
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `enabled` | boolean | Non | `true` | Activer le dispatcher de webhooks |
| `connectTimeoutMs` | long | Non | `5000` | Timeout de connexion en millisecondes |
| `requestTimeoutMs` | long | Non | `10000` | Timeout de réponse par requête en millisecondes |

> **Différence avec la section `3.7 webhooks`** : la section `webhooks` documente les abonnements de dispatch (dead-letter, dispatcher global). La section `http-webhook-service` configure le module d'implémentation HTTP sous-jacent utilisé pour la livraison effective.

---

### 3.12 `secret-service` (bootstrap.json)

Configuration du service de gestion des secrets. Ce service rend disponible le binding `$Secrets` dans les scripts JavaScript. Il est désactivé par défaut. L'implémentation est chargée via SPI depuis le répertoire `connectors-dir`.

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

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `enabled` | boolean | Non | `false` | Activer le service de secrets. Si `false`, `$Secrets` n'est pas injecté. |
| `type` | string | Non | `"filesystem"` | Type de provider : `"filesystem"` (module `secret-service-filesystem`) ou tout provider SPI disponible dans `connectors-dir` |
| `root` | string | Requis si enabled | — | Répertoire racine de stockage des secrets (pour le provider `filesystem`) |
| `keyfile` | string | Non | — | Chemin vers le fichier de clé de chiffrement (pour le provider `filesystem` avec chiffrement AES) |

#### Binding `$Secrets` dans les scripts

Une fois activé, le provider est accessible dans tout script API ou hook via `$Secrets` :

```javascript
// Lire un secret
const apiKey = $Secrets.get("stripe-api-key");

// Lister les secrets disponibles pour le tenant courant
const names = $Secrets.list();
```

| Méthode | Signature | Description |
|---------|-----------|-------------|
| `get` | `get(name: string): string` | Retourne la valeur du secret identifié par `name`. Lance une exception si le secret est introuvable. |
| `list` | `list(): string[]` | Retourne la liste des noms de secrets disponibles pour le tenant courant. |
| `tokenize` | `tokenize(value: string): string` | Remplace une valeur sensible par un token opaque (UUID v4), non réversible sans le provider. Lance `UnsupportedOperationException` si le provider ne supporte pas la tokenisation. |
| `detokenize` | `detokenize(token: string): string` | Retrouve la valeur sensible associée à un token. |

Le binding est scopé au tenant courant de la requête : `$Secrets.get("key")` résout `{tenantId}/{key}` sur le provider. Un secret manquant génère une `SecretNotFoundException` qui remonte comme erreur 500.

#### Guide d'utilisation pas à pas

##### Étape 1 — Prérequis

S'assurer que le JAR `secret-service-filesystem` est présent dans `connectors-dir` (défaut : `.connectors/`).

##### Étape 2 — Générer la clé maîtresse

Le provider `filesystem` chiffre chaque secret en AES-256-GCM. Une clé maîtresse de 32 octets est requise.

**Option A — fichier binaire (recommandé en production) :**

```bash
openssl rand -out /opt/payos/secrets/.keyfile 32
chmod 600 /opt/payos/secrets/.keyfile
mkdir -p /opt/payos/secrets
chmod 700 /opt/payos/secrets
```

**Option B — variable d'environnement (Base64) :**

```bash
export PAYOS_SECRET_MASTER_KEY=$(openssl rand -base64 32)
```

Si `keyfile` est absent de la configuration, le provider lit `PAYOS_SECRET_MASTER_KEY`. Si ni l'un ni l'autre n'est présent, le démarrage échoue.

##### Étape 3 — Configurer bootstrap.json

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

##### Étape 4 — Provisionner un secret

> **Le binding `$Secrets` dans les scripts JS n'expose que `get`, `list`, `tokenize` et `detokenize`** — pas d'écriture. Le provisionnement passe par le CLI `spm` (module `secret-service-filesystem`).

**Avec le CLI `spm`** (méthode recommandée ; JAR `spm.jar`) :

```bash
# Valeur passée en argument
java -jar spm.jar set \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme \
  --name stripe-api-key \
  --value sk_live_xxxx \
  --type api-key

# Valeur lue depuis stdin (pour éviter de l'exposer dans l'historique shell)
echo -n "sk_live_xxxx" | java -jar spm.jar set \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme \
  --name stripe-api-key \
  --type api-key

# Si la clé maîtresse est dans la variable d'environnement (pas de --keyfile)
export PAYOS_SECRET_MASTER_KEY=<base64>
java -jar spm.jar set --root /opt/payos/secrets --tenant acme --name db-password --value s3cr3t
```

Le JAR `spm.jar` est produit par `mvn package` dans le module `secret-service-filesystem`. Une fois les wrappers installés (`scripts/install.sh` / `install.ps1`), la commande `spm` est disponible directement sans `java -jar`.

Pour le provider `vault` (module `secret-service-vault`, livré avec PayOS), utiliser le CLI/API Vault natif pour provisionner — PayOS lit les secrets, il ne les écrit pas via ce provider. Pour d'autres backends externes (AWS Secrets Manager, Azure Key Vault…), un connecteur `ISecretProviderFactory` dédié doit être implémenté ; voir [configuration/secret-service.md](secret-service.md).

##### Étape 5 — Lire le secret dans un script

Dans tout script API ou hook, `$Secrets` est disponible dès que `secret-service` est activé :

```javascript
// api/payment.js
const apiKey = $Secrets.get("stripe-api-key");

if (!apiKey) {
    return $Errors.business("SECRETS_NOT_CONFIGURED", "Stripe API key not provisioned", 500);
}

// utiliser apiKey pour appeler l'API externe
$Response.json({ status: "ok" });
```

Le secret est automatiquement scopé au tenant courant de la requête. `$Secrets.get("stripe-api-key")` résout `{tenantId}/stripe-api-key` sur le provider sans que le script ait à préciser le tenant.

##### Étape 6 — Gérer l'absence d'un secret

`$Secrets.get()` lève une exception si le secret est absent, ce qui génère automatiquement une erreur 500. Pour une gestion explicite :

```javascript
// Vérification préalable via list()
const available = $Secrets.list();
if (!available.includes("stripe-api-key")) {
    return $Errors.notFound("SECRET_MISSING", "stripe-api-key non provisionné pour ce tenant");
}
const apiKey = $Secrets.get("stripe-api-key");
```

Ou avec un try/catch si `list()` n'est pas souhaité :

```javascript
let apiKey;
try {
    apiKey = $Secrets.get("stripe-api-key");
} catch (e) {
    return $Errors.business("SECRET_UNAVAILABLE", "stripe-api-key indisponible", 503);
}
```

##### Étape 7 — Rotation d'un secret

Rappeler `set` sur le même nom — la version est incrémentée automatiquement, la nouvelle valeur est chiffrée avec un nouvel IV AES. **Du côté des scripts, aucune modification n'est nécessaire** : `$Secrets.get("stripe-api-key")` retourne toujours la valeur courante.

```bash
# La version passe de 1 à 2 ; l'ancien .enc est archivé sous versions/, le nouveau remplace .enc atomiquement
java -jar spm.jar set \
  --root /opt/payos/secrets \
  --keyfile /opt/payos/secrets/.keyfile \
  --tenant acme \
  --name stripe-api-key \
  --value sk_live_yyyy
```

##### Référence rapide des commandes CLI

| Commande | Description |
|----------|-------------|
| `spm keygen --out <path>` | Générer un fichier clé AES-256 de 32 octets |
| `spm set --root <dir> --tenant <id> --name <name> [--value <val>] [--type <type>]` | Stocker ou mettre à jour un secret |
| `spm get --root <dir> --tenant <id> --name <name>` | Lire la valeur d'un secret (stdout) |
| `spm list --root <dir> --tenant <id>` | Lister les noms des secrets d'un tenant |
| `spm delete --root <dir> --tenant <id> --name <name>` | Supprimer un secret |
| `spm describe --root <dir> --tenant <id> --name <name>` | Afficher les métadonnées sans exposer la valeur |

`--keyfile` est optionnel sur toutes les commandes sauf `keygen` — si absent, lit `$PAYOS_SECRET_MASTER_KEY`. Le provider `filesystem` conserve aussi un historique de versions
(`IVersionedSecretProvider`) et supporte la tokenisation (`ITokenProvider`), mais `spm` n'expose pas encore de sous-commandes pour ces opérations (API Java directe uniquement).

---

#### Extension par SPI

Pour ajouter un provider personnalisé, implémenter `ISecretProviderFactory` et enregistrer le service via `META-INF/services/ma.s2m.payos.secret.api.ISecretProviderFactory` dans le JAR du connecteur, placé dans `connectors-dir`. La valeur `type` dans la configuration doit correspondre à la valeur retournée par `ISecretProviderFactory.type()`.

---

### 3.13 `extensions-dir` (bootstrap.json)

Répertoire de JARs Java chargés au démarrage et rendus accessibles aux scripts JavaScript via `Java.type()`. Permet d'appeler n'importe quelle classe Java publique d'une bibliothèque tierce sans déclarer de dépendance Maven dans le kernel.

```json
"extensions-dir": "./extensions"
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `extensions-dir` | string | Non | — | Chemin absolu ou relatif vers le dossier contenant les JARs d'extension |

#### Résolution de la valeur

La clé est résolue dans cet ordre (première valeur non nulle retenue) :

1. Propriété système Java : `-Dextensions-dir=/opt/payos/extensions`
2. Variable d'environnement : `PAYOS_EXTENSIONS_DIR=/opt/payos/extensions`
3. Clé `extensions-dir` dans `bootstrap.json` (ou tout fichier JSON du `configDir`)

#### Mécanisme de chargement

Au démarrage, `ExtensionLoader` :

1. Résout le chemin via la séquence ci-dessus.
2. Scanne tous les fichiers `*.jar` présents dans le répertoire.
3. Construit un `URLClassLoader` dont le parent est le classloader du fat JAR (kernel).
4. Enregistre ce classloader dans `PayOSConfig` comme `extensionClassLoader`.
5. `PolyglotScriptingEngine` injecte ce classloader dans chaque contexte GraalVM via `.hostClassLoader()`.

La chaîne de délégation garantit que `Java.type()` résout à la fois les classes des JARs d'extension **et** toutes les classes du fat JAR (kernel PayOS).

#### Rechargement à chaud

Lorsque la configuration est rechargée (hot-reload via `ConfigWatcher`), `ExtensionLoader.initialize()` est rappelé. Les contextes GraalVM créés pour les requêtes suivantes utilisent le nouveau classloader ; les contextes déjà ouverts ne sont pas affectés.

#### Utilisation dans les scripts JS

```javascript
// api/process-iso.js
var IsoParser = Java.type('com.acme.iso.IsoParser');
var parser    = new IsoParser();
var message   = parser.parse($Request.getBodyAsString());

$Response.json({ pan: message.get('PAN'), amount: message.get('AMOUNT') });
```

Les classes du fat JAR (kernel) sont également accessibles :

```javascript
var ObjectMapper = Java.type('com.fasterxml.jackson.databind.ObjectMapper');
var mapper = new ObjectMapper();
```

> **Contrainte de sécurité :** le prédicat `allowHostClassLookup` reste actif — `java.lang.System` est toujours bloquée. Toutes les autres classes accessibles via le classloader peuvent être instanciées depuis les scripts.

#### Structure de déploiement

```
payos-runtime/
  config/
    bootstrap.json         ← "extensions-dir": "./extensions"
  extensions/
    iso-parser-1.0.jar
    crypto-utils-2.1.jar
```

Aucune interface ne doit être implémentée — les JARs peuvent être des bibliothèques tierces quelconques.

---

### 3.14 `notification-service` (bootstrap.json)

Configuration du connecteur de notification (binding `$Notification`). Volontairement **distincte** de [`queue-service`](#36-queue-service) : un connecteur de notification adossé à une queue (ex. `payos-notification-connector`) établit et possède sa **propre** connexion broker à partir de ce bloc, plutôt que de réutiliser le client générique exposé aux scripts sous `$Queue` — le broker/topic de notification peut différer de celui de `$Queue`.

```json
"notification-service": {
  "configuration": {
    "type": "nats",
    "host": "nats.internal",
    "port": 4222,
    "topic": "payos.notifications"
  }
}
```

| Clé | Type | Requis | Défaut | Description |
|-----|------|--------|--------|-------------|
| `type` | string | Non | Dérivé du connecteur (`nats` pour le connecteur queue) | Type de client MoM utilisé par le connecteur |
| `host` | string | Non | `"localhost"` | Hôte du broker de notification |
| `port` | int | Non | `4222` | Port du broker de notification |
| `topic` | string | Non | `"payos.notifications"` | Sujet utilisé pour la connexion |

> Le bloc `configuration` est transmis tel quel (`Map<String, String>`) à `INotificationServiceFactory#initialize` du connecteur actif au démarrage ; les clés au-delà de `type` sont donc spécifiques au connecteur installé dans `connectors-dir`. Si aucun connecteur de notification n'est présent, ce bloc est ignoré et `$Notification` n'est pas disponible dans les scripts.

---

## 4. manifest.json — Déclaration de capability

Chaque capability (module fonctionnel) déclare son identité et ses dépendances dans un fichier `manifest.json`.

```json
{
  "id": "payment-links",
  "name": "Payment Links",
  "version": "1.0.0",
  "description": "Capability allowing merchants to create and manage payment links.",
  "tags": ["payments"],
  "category": "capability", // vs application
  "dependencies": {
    "required": ["payment-core"],
    "optional": ["customer-management"]
  },
  "lifecycle": {
    "installScript": "scripts/install.js",
    "uninstallScript": "scripts/uninstall.js",
    "activateScript": "scripts/activate.js",
    "deactivateScript": "scripts/deactivate.js"
  },
  "metadata": {
    "author": "Team Name",
    "contact": "team@example.com"
  }
}
```

| Clé | Type | Requis | Description |
|-----|------|--------|-------------|
| `id` | string | Oui | Identifiant unique de la capability |
| `name` | string | Oui | Nom lisible |
| `version` | string | Oui | Version sémantique |
| `description` | string | Non | Description fonctionnelle |
| `tags` | array[string] | Non | Tags de classification |
| `category` | string | Oui | capability ou application |
| `dependencies.required` | array[string] | Non | IDs de capabilities requises |
| `dependencies.optional` | array[string] | Non | IDs de capabilities optionnelles |
| `lifecycle.installScript` | string | Non | Script JS exécuté à l'installation |
| `lifecycle.uninstallScript` | string | Non | Script JS exécuté à la désinstallation |
| `lifecycle.activateScript` | string | Non | Script JS exécuté à l'activation |
| `lifecycle.deactivateScript` | string | Non | Script JS exécuté à la désactivation |
| `metadata.author` | string | Non | Auteur / équipe responsable |
| `metadata.contact` | string | Non | Contact de l'équipe |

---

## 5. Résolution hiérarchique des valeurs

Pour chaque clé de configuration, la résolution suit cet ordre (du plus prioritaire au moins prioritaire) :

```
1. Config spécifique à l'application  (apps/{id}/config/*.json)
2. Bloc security/database de l'application  (applications[].security / database-service)
3. Config du tenant courant  (multitenancy.tenants.{tenantId}.*)
4. Config du tenant "default"  (multitenancy.tenants.default.*)
5. Config globale racine  (bootstrap.json — clés de premier niveau)
6. Valeurs codées en dur dans IConfigSpec  (constantes Java)
```

---

## 6. Constantes et valeurs par défaut

| Constante | Valeur | Utilisation |
|-----------|--------|-------------|
| `SESSION_COOKIE_NAME` | `"PAYOS_SESSION_ID"` | Nom du cookie de session HTTP |
| `SESSION_ID_ATTRIBUTE` | `"payos.sessionId"` | Attribut de requête portant l'ID de session |
| `DEFAULT_DISCOVERY_URI` | `http://localhost:8080/realms/master/.well-known/openid-configuration` | URI OIDC si non spécifiée |
| `DEFAULT_MAX_SESSIONS` | `10 000` | Max sessions simultanées |
| `DEFAULT_TTL_SECONDS` | `1800` | TTL session (30 min) |
| `DEFAULT_REQUESTS_PER_MINUTE` | `1000` | Quota par défaut |
| `DEFAULT_DATABASE` | `"payos_db"` | Nom d'instance BDD par défaut |
| `DEFAULT_DATABASE_SCHEMA` | `"public"` | Schéma BDD par défaut |
| `DEFAULT_ISOLATION_MODE` | `"shared-schema"` | Mode isolation par défaut |
| `DEFAULT_QUEUE_CLIENT_TYPE` | `"nats"` | Type de client MoM par défaut |
| `APPLICATION_MAPPING_FILES_DIRECTORY` | `"model"` | Dossier des fichiers HBM |
| `CAPABILITIES_DIR` | `".capabilities"` | Dossier interne géré par `cpm` dans `configDir`. Contient `activation.json` et les capabilities installées |
| `APPLICATIONS_DIR` | `".applications"` | Dossier interne géré par `cpm` dans `configDir`. Contient les applications installées depuis un catalogue |

---

## 7. Exemples complets

### Déploiement mono-tenant, HTTP simple

```json
{
  "servers": [
    { "host": "0.0.0.0", "port": 8080, "protocol": "http" }
  ],
  "applications": [
    {
      "id": "myapp",
      "name": "My App",
      "base.path": "/opt/payos/apps/myapp"
    }
  ],
  "multitenancy": {
    "requireTenantId": false
  },
  "database-service": {
    "configuration": {
      "url": "jdbc:postgresql://localhost:5432/mydb",
      "username": "user",
      "password": "pass",
      "ddl-auto": "validate"
    }
  }
}
```

### Déploiement multi-tenant, HTTPS + OIDC + NATS

```json
{
  "servers": [
    {
      "host": "0.0.0.0",
      "port": 8443,
      "protocol": "http",
      "keystore": "/etc/ssl/payos.p12",
      "keystorePassword": "changeit",
      "keystoreType": "PKCS12"
    },
    {
      "protocol": "queue",
      "type": "nats",
      "consumer-topic": "payos.requests"
    }
  ],
  "applications": [
    {
      "id": "gateway",
      "name": "Payment Gateway",
      "base.path": "/opt/payos/apps/gateway",
      "version": "1.0.0"
    }
  ],
  "security": {
    "provider": "nimbus",
    "clientId": "payos-gw",
    "clientSecret": "super-secret",
    "oidcProviderBaseUrl": "https://auth.internal",
    "realm": "payos-prod",
    "scope": "openid profile",
    "sessionCookieSecure": true,
    "sessionTtlSeconds": 3600
  },
  "multitenancy": {
    "requireTenantId": true,
    "default-isolation-mode": "dedicated-schema",
    "default-tenant-quotas": { "enabled": true, "requestsPerMinute": 1000 },
    "tenants": {
      "default": { "schema": "public", "isolationMode": "shared-schema" },
      "bank-a": { "schema": "bank_a", "isolationMode": "dedicated-schema" },
      "bank-b": {
        "isolationMode": "dedicated-database",
        "database-service": {
          "name": "bank_b_db",
          "configuration": {
            "url": "jdbc:postgresql://db-b:5432/bankb",
            "username": "bankb_user",
            "password": "bankb_pass"
          }
        }
      }
    }
  },
  "database-service": {
    "name": "payos_shared",
    "configuration": {
      "url": "jdbc:postgresql://db-shared:5432/payos",
      "username": "payos",
      "password": "payos_pass",
      "dialect": "org.hibernate.dialect.PostgreSQLDialect",
      "ddl-auto": "validate",
      "max-pool-size": 20,
      "minimum-idle": 5
    }
  },
  "queue-service": {
    "configuration": {
      "enabled": true,
      "type": "nats",
      "host": "nats.internal",
      "port": 4222,
      "publisher-topic": "payos.out",
      "consumer-topic": "payos.in"
    }
  }
}
```
