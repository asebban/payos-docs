Created: 2026-11-09
Last updated: 2026-11-13
Version: 2

# Démarrer TSS : prérequis, commande de démarrage et ligne de commande

Ce guide décrit, de A à Z, comment démarrer le Task Scheduling System (`payos-job-scheduler`, « TSS ») en mode standalone :
- tout ce qui doit exister avant le premier démarrage ;
- la commande de démarrage exacte ;
- ce que le démarrage affiche quand il réussit ;
- toutes les possibilités de la ligne de commande.

Un résumé du mode embarqué (TSS dans `payos-runtime`) figure en [§13](#13-mode-embarqué-tss-dans-payos-runtime).

Tous les comportements décrits ici ont été vérifiés dans le code de `StandaloneMain` et `StandaloneConfigLoader` (version 1.1.3-RELEASE, build du 2026-11-09 : `tss-config.json` y est résolu par rapport au répertoire de base). Les cas limites (fichier de configuration introuvable, `--help`, commande inconnue, échec au démarrage) ont été vérifiés par une exécution réelle du jar.

Pour le détail de chaque fonctionnalité (réconciliation, hot reload, API d'administration, historique, métriques), voir le [guide des opérations](task-scheduling-system-v21-2026-11-15.md). Pour l'écriture des scripts de job, voir le [guide développeur](../developer/task-scheduler-guide-v21-2026-11-19.md).

## 1. Vue d'ensemble : la liste à suivre dans l'ordre

| # | Étape | Section |
| --- | --- | --- |
| 1 | Java 21 ou supérieur sur la machine | [§2](#2-prérequis-logiciels) |
| 2 | Le jar `payos-job-scheduler-<version>-standalone.jar` | [§2](#2-prérequis-logiciels) |
| 3 | Une base (ou un schéma) dédiée à TSS, avec les tables Quartz et `tss_job_execution` créées **une seule fois** | [§3](#3-la-base-de-données-de-tss) |
| 4 | Un répertoire de base avec la configuration PayOS, les jobs et leurs scripts | [§4](#4-le-répertoire-de-base) |
| 5 | La configuration propre à TSS : `tss-config.json` et/ou variables d'environnement | [§5](#5-la-configuration-propre-à-tss) |
| 6 | La configuration PayOS lue par TSS : tenants, bases, connecteurs | [§6](#6-la-configuration-payos-lue-par-tss) |
| 7 | Les déclarations de jobs et leurs scripts | [§7](#7-les-jobs) |
| 8 | Le lancement, puis la vérification des logs | [§8](#8-la-commande-de-démarrage) et [§9](#9-séquence-de-démarrage-et-logs-attendus) |

## 2. Prérequis logiciels

- **Java 21 ou supérieur.** Le module est compilé pour Java 21 (`java.version` du `payos-parent`). Vérifiez avec `java -version`.
- **Le jar standalone.** Il est produit par `mvn clean package` dans `payos-job-scheduler`. Ce build génère deux jars ; seul le jar avec le suffixe `-standalone` est exécutable :
  - `target/payos-job-scheduler-<version>-standalone.jar` : jar autonome qui contient toutes les dépendances ;
  - `target/payos-job-scheduler-<version>.jar` : les classes de TSS seules, utilisé par `payos-runtime` pour le mode embarqué.
- **Ce que le jar standalone contient déjà :**
  - le moteur de scripts GraalVM, Hibernate, Quartz et HikariCP ;
  - les drivers JDBC **H2** et **PostgreSQL** ;
  - le fournisseur de bases `dynamic-database-service` et le chargeur `tenant-config-loader-database` ;
  - le mapping `mappings/PayosQueueOutbox.hbm.xml` ;
  - un `logback.xml` (logs sur la console) ;
  - les scripts SQL de création des tables (voir [§3](#3-la-base-de-données-de-tss)).
- **Ce qu'il ne contient pas :**
  - le driver JDBC **Oracle**, ni aucun autre driver. Il faut alors l'ajouter au classpath et lancer avec `java -cp` au lieu de `java -jar` : voir [§8.4](#84-ajouter-un-driver-jdbc-par-exemple-oracle) ;
  - le module de sessions Redis : l'API d'administration force donc le stockage des sessions en mémoire.
- **Accès réseau** depuis la machine TSS :
  - la base de TSS ;
  - la base de chaque tenant qui a des jobs ;
  - la base plateforme, si `platform-database-service` est déclaré ;
  - le fournisseur d'identité (par exemple Keycloak), uniquement si l'API d'administration est activée.

## 3. La base de données de TSS

TSS a **sa propre base**, ou son propre schéma. Elle contient les tables Quartz `QRTZ_*` (définitions de jobs, triggers, verrous de cluster) et la table d'historique `tss_job_execution`. Ce n'est jamais une base de tenant ni la base plateforme. **TSS ne crée jamais ces tables** : appliquez les scripts une fois, avant le premier démarrage.

### 3.1 Créer la base et l'utilisateur

Créez une base ou un schéma vide, et un utilisateur qui peut y lire, écrire et créer des tables. En H2 fichier, le fichier de base est créé automatiquement à la première connexion.

### 3.2 Extraire les scripts du jar

Les scripts sont dans le jar standalone :

| Script | Chemin dans le jar |
| --- | --- |
| Tables Quartz, PostgreSQL | `org/quartz/impl/jdbcjobstore/tables_postgres.sql` |
| Tables Quartz, Oracle | `org/quartz/impl/jdbcjobstore/tables_oracle.sql` |
| Tables Quartz, H2 | `org/quartz/impl/jdbcjobstore/tables_h2.sql` |
| Table d'historique TSS (tous éditeurs) | `db/migration/tss_job_execution.sql` |
| Tables de l'état des dépendances (tous éditeurs, **facultatif** — seulement si un job déclare un déclencheur `dependency`) | `db/migration/tss_dependency.sql` |

Extraction avec l'outil `jar` du JDK, ici pour PostgreSQL :

```bash
jar xf payos-job-scheduler-<version>-standalone.jar org/quartz/impl/jdbcjobstore/tables_postgres.sql db/migration/tss_job_execution.sql db/migration/tss_dependency.sql
```

Les fichiers sont extraits avec leur arborescence (`org/quartz/...`, `db/migration/...`) dans le répertoire courant.

### 3.3 Appliquer les scripts

> **Attention :** `tables_postgres.sql` et `tables_oracle.sql` commencent par supprimer toutes les tables `QRTZ_*` (11 instructions `DROP`). Les rejouer sur une base en service **efface tous les jobs planifiés**. `tables_h2.sql` n'a pas de `DROP` : le rejouer échoue, car les tables existent déjà. N'appliquez ces scripts que sur une base neuve.

**PostgreSQL.** Si vous utilisez un schéma dédié, par exemple `tss`, passez-le avec `PGOPTIONS` pour que les tables y soient créées :

```bash
export PGOPTIONS="-c search_path=tss"
psql -h db.internal -U tss_svc -d tss -f org/quartz/impl/jdbcjobstore/tables_postgres.sql
psql -h db.internal -U tss_svc -d tss -f db/migration/tss_job_execution.sql
```

**Oracle :**

```bash
sqlplus tss_svc/<mot_de_passe>@//db.internal:1521/TSS @org/quartz/impl/jdbcjobstore/tables_oracle.sql
sqlplus tss_svc/<mot_de_passe>@//db.internal:1521/TSS @db/migration/tss_job_execution.sql
```

**H2 (fichier).** On utilise l'outil `RunScript` d'H2, inclus dans le jar. L'URL doit être **exactement** celle que TSS utilisera, options comprises (voir [§5](#5-la-configuration-propre-à-tss)) :

```bash
java -cp payos-job-scheduler-<version>-standalone.jar org.h2.tools.RunScript \
  -url "jdbc:h2:file:./.h2/tss_db;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE" \
  -user sa -password "" -script org/quartz/impl/jdbcjobstore/tables_h2.sql
java -cp payos-job-scheduler-<version>-standalone.jar org.h2.tools.RunScript \
  -url "jdbc:h2:file:./.h2/tss_db;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;AUTO_SERVER=TRUE" \
  -user sa -password "" -script db/migration/tss_job_execution.sql
```

Une URL H2 `file:` relative est résolue par rapport au **répertoire de travail** du processus. Lancez donc cette commande depuis le répertoire d'où TSS sera démarré, ou utilisez un chemin absolu.

Si un job déclare un déclencheur `dependency`, appliquez aussi `db/migration/tss_dependency.sql` de la même façon (même outil, même base, après `tss_job_execution.sql`) — sans lui, chaque fin de job amont est journalisée en erreur et jamais comptée.

### 3.4 Vérifier

Une fois la configuration du [§5](#5-la-configuration-propre-à-tss) en place, `list` doit répondre `[ ]` sur une base neuve. Cette commande ne fait que lire la base (voir [§11](#11-la-ligne-de-commande)) :

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss list
```

## 4. Le répertoire de base

TSS résout tous ses fichiers par rapport à **un répertoire de base**. Celui-ci est passé par `--scheduler-base-dir <chemin>` ; par défaut, c'est le répertoire de travail du processus. Les fichiers concernés sont :
- `jobs/` ;
- les scripts de job ;
- le fichier de configuration PayOS ;
- et, à travers ce dernier, les connecteurs.

Disposition typique :

```
/opt/payos-tss/                      ← --scheduler-base-dir (et répertoire de travail recommandé)
  tss-config.json                    ← configuration propre à TSS (§5)
  payos.json                         ← fichier PayOS lu pour les tenants et les bases (§6)
  multitenancy.json                  ← autres *.json du configDir de PayOS (§6)
  database.json
  platform-database.json             ← optionnel
  connectors/                        ← service-adapters-dir de PayOS, optionnel (§6)
    payos-database-audit-trail-<v>.jar
  jobs/
    platform-jobs.json               ← jobs plateforme, optionnel (§7)
    template.json                    ← modèle pour les nouveaux tenants, optionnel
    tenant1/
      jobs.json                      ← jobs du tenant tenant1 (§7)
      custom-jobs.json               ← surcharges du tenant, optionnel
    tenant1-reconciliation.js        ← scripts : « pool » partagé, chemins relatifs au répertoire de base
```

> **Attention — certains chemins dépendent du répertoire de travail, pas de `--scheduler-base-dir` :** les URL H2 `file:` relatives et les fichiers de mapping Hibernate relatifs (limitation héritée du kernel). La règle la plus simple : **démarrez TSS depuis son répertoire de base**, ou utilisez des chemins absolus pour ces deux éléments. `tss-config.json`, lui, est résolu par rapport au répertoire de base (voir [§5](#5-la-configuration-propre-à-tss)).

## 5. La configuration propre à TSS

Chaque clé est résolue sur trois niveaux. La première valeur non vide l'emporte :

1. une **variable d'environnement** (par exemple `TSS_DB_URL`) ;
2. une **propriété système** de même nom (`-DTSS_DB_URL=...`, placée **avant** `-jar`) ;
3. un **champ du fichier JSON** `tss-config.json`, dans le bloc indiqué.

**Emplacement du fichier.** Il est donné par `TSS_CONFIG_FILE`, en variable d'environnement ou en propriété système uniquement. Par défaut, c'est `tss-config.json`. Un chemin absolu est utilisé tel quel ; un chemin relatif, comme la valeur par défaut, est résolu par rapport au **répertoire de base** (`--scheduler-base-dir`), comme `jobs/` et `payos.json`.

Les versions antérieures au 2026-11-09 résolvaient ce chemin par rapport au répertoire de travail. Par compatibilité, un fichier absent du répertoire de base mais présent à cet ancien emplacement est encore utilisé, avec un `WARN` « TSS config file found at '…', relative to the working directory » qui invite à le déplacer.

Le fichier est **facultatif** : tout peut venir de variables d'environnement. S'il est introuvable, TSS l'indique au démarrage par la ligne `No TSS config file at '<chemin>' — configuration comes from environment variables and system properties only`. Ce fichier ne développe **pas** les `${...}` : passez les secrets par variable d'environnement.

| Variable / propriété | Bloc et champ JSON | Défaut | Rôle |
| --- | --- | --- | --- |
| `TSS_CONFIG_FILE` | — | `tss-config.json` | Emplacement du fichier JSON, relatif au répertoire de base s'il n'est pas absolu. |
| `TSS_DB_URL` | `database.dbUrl` | **obligatoire** | URL JDBC de la base TSS. |
| `TSS_DB_DRIVER` | `database.dbDriver` | **obligatoire** | Classe du driver, par exemple `org.postgresql.Driver` ou `org.h2.Driver`. |
| `TSS_DB_USER` | `database.dbUser` | — | Utilisateur. |
| `TSS_DB_PASSWORD` | `database.dbPassword` | — | Mot de passe, de préférence par variable d'environnement. |
| `TSS_DB_VENDOR` | `database.dbVendor` | SQL générique | `postgresql` ou `oracle` choisit le dialecte Quartz adapté ; toute autre valeur, dont `h2`, utilise le dialecte générique. |
| `TSS_DB_SCHEMA` | `database.dbSchema` | — | Schéma, s'il n'est pas celui par défaut. |
| `TSS_DB_MAX_POOL_SIZE` | `database.dbMaxPoolSize` | `5` | Taille du pool de connexions de TSS. |
| `TSS_SCHEDULER_INSTANCE_NAME` | `scheduler.schedulerInstanceName` | `TSS-Cluster` | Nom du scheduler. Tous les nœuds d'un même cluster doivent avoir le **même**, et la CLI doit utiliser ce même nom. |
| `TSS_THREAD_COUNT` | `scheduler.threadCount` | `5` | Nombre d'exécutions simultanées sur ce nœud. |
| `TSS_MAX_CONCURRENT_EXECUTIONS_PER_TENANT` | `scheduler.maxConcurrentExecutionsPerTenant` | `0` (désactivé) | Quota d'exécutions simultanées par tenant, à l'échelle du cluster. |
| `TSS_MAX_PARALLEL_WORKERS_PER_JOB` | `scheduler.maxParallelWorkersPerJob` | `4` | Nombre maximal de workers par appel `$Parallel.forEach`. |
| `TSS_METRICS_HTTP_PORT` | `scheduler.metricsHttpPort` | `0` (désactivé) | Port du serveur Prometheus `GET /metrics`. |
| `TSS_HOT_RELOAD_ENABLED` | `scheduler.hotReloadEnabled` | `true` | Applique les modifications de `jobs/` sans redémarrage. |
| `TSS_EVENT_TRIGGER_REFRESH_SECONDS` | `scheduler.eventTriggerRefreshSeconds` | `30` | Fréquence à laquelle chaque nœud relit et démarre/arrête les déclencheurs événementiels (`manual`/`queue`/`dependency`) pour les faire correspondre à ce qui est déclaré. |
| `TSS_QUEUE_TYPE` | `queue.queueType` | *(aucun — pas de client queue)* | Type de broker pour les déclencheurs `queue`, propre à TSS, indépendant de la connexion queue éventuelle de l'application hôte. |
| `TSS_QUEUE_HOST` | `queue.queueHost` | **obligatoire si `TSS_QUEUE_TYPE` est défini** | Hôte du broker. |
| `TSS_QUEUE_PORT` | `queue.queuePort` | `4222` | Port du broker. |
| `TSS_QUEUE_ADAPTERS_DIR` | `queue.queueAdaptersDir` | *(aucun — classpath seul)* | Répertoire de JARs `IQueueClientFactory` supplémentaires, chargés en plus du classpath. Chemin relatif résolu par rapport à `--scheduler-base-dir`. |
| `TSS_JOBS_ROOT_DIR` | `jobs.rootDir` | `jobs` | Racine des jobs, relative au répertoire de base. |
| `TSS_PAYOS_CONFIG_FILE` | `jobs.payosConfigFile` | `payos.json` | Fichier de configuration PayOS, relatif au répertoire de base. |
| `TSS_ADMIN_HTTP_PORT` | `admin.httpPort` | `0` (désactivé) | Port de l'API HTTP d'administration. |
| `TSS_ADMIN_PLATFORM_ROLE` | `admin.platformRole` | `tss-platform-admin` | Rôle du token qui donne tous les droits. |
| `TSS_ADMIN_TENANT_ROLE` | `admin.tenantRole` | `tss-tenant-admin` | Rôle du token limité au tenant de l'appelant. |

Une valeur numérique illisible retombe sur le défaut sans message. Deux propriétés système peuvent aussi être utiles :

| Propriété système | Défaut | Rôle |
| --- | --- | --- |
| `payos.metrics.tenant-tags-enabled` | `true` en standalone | Ajoute le tag `tenant_id` aux métriques. Mettez `false` pour le retirer. |
| `logback.configurationFile` | `logback.xml` du jar | Votre propre configuration de logs. |

Exemple de `tss-config.json` :

```json
{
  "database": {
    "dbUrl": "jdbc:postgresql://db.internal:5432/tss",
    "dbUser": "tss_svc",
    "dbDriver": "org.postgresql.Driver",
    "dbVendor": "postgresql",
    "dbSchema": "tss",
    "dbMaxPoolSize": 10
  },
  "scheduler": {
    "schedulerInstanceName": "TSS-Cluster",
    "threadCount": 10,
    "metricsHttpPort": 9404,
    "hotReloadEnabled": true
  },
  "queue": {
    "queueType": "nats",
    "queueHost": "broker.internal"
  },
  "jobs": {
    "rootDir": "jobs",
    "payosConfigFile": "payos.json"
  },
  "admin": {
    "httpPort": 8095
  }
}
```

Le bloc `queue` est facultatif : omettez-le si aucun job ne déclare de déclencheur `queue`.

## 6. La configuration PayOS lue par TSS

TSS ne duplique pas la configuration des tenants. Il lit **la vraie configuration PayOS**, avec les mêmes règles que le runtime :

- **Le fichier PayOS** (`payos.json` par défaut) n'est lu que pour sa clé `configDir` (défaut `"."`, relatif au fichier) et pour `service-adapters-dir`. Tous les `*.json` **directement** dans `configDir` sont fusionnés ; les sous-répertoires ne sont pas lus. `custom.json` est appliqué en dernier.
- **`multitenancy.tenants`** déclare les tenants. Chaque tenant qui a des jobs doit y être déclaré. **Tous** les tenants déclarés sont validés au démarrage, même ceux qui n'ont aucun job : une déclaration invalide bloque le démarrage.
- **`database-service`** fournit la ou les bases des tenants : référence nommée, ou repli sur la base par défaut, comme dans le runtime.
- **`platform-database-service`** (optionnel) fournit la base plateforme utilisée par `$PlatformDB`. Son tableau `mapping-files` est obligatoire s'il est déclaré.
- **`service-adapters-dir`** (optionnel) désigne le répertoire de connecteurs à charger, relatif au fichier PayOS. `PAYOS_SERVICE_ADAPTERS_DIR` est prioritaire. Ses JARs fournissent des chargeurs de configuration, des fabriques de bases, et des fichiers de mapping livrés comme ressources, par exemple `mappings/PayosAuditEvent.hbm.xml` dans `payos-database-audit-trail`.
- **`tenant-config-loader`** (optionnel) lit les tenants dans une table de la base plateforme, au lieu de les copier en JSON.
- **`security`** n'est nécessaire que si l'API d'administration est activée : c'est la configuration OIDC qui sert à vérifier les tokens.

Trois règles propres au standalone :
- les `${VAR}`, `${VAR:-défaut}` et `${file:/chemin}` **sont** développés dans ces fichiers PayOS ;
- le service de secrets PayOS n'est **pas** initialisé : un secret doit venir d'une variable d'environnement ou d'un fichier monté ;
- les fichiers chiffrés par `payosv2-packer` ne sont **pas** pris en charge.

Exemple minimal, dans `configDir` :

```json
{
  "database-service": {
    "name": "main",
    "configuration": {
      "url": "jdbc:postgresql://db.internal:5432/payos",
      "username": "payos_svc",
      "password": "${PAYOS_DB_PASSWORD}",
      "driver_class": "org.postgresql.Driver",
      "dialect": "org.hibernate.dialect.PostgreSQLDialect"
    }
  },
  "multitenancy": {
    "tenants": {
      "tenant1": { "schema": "tenant1", "isolationMode": "dedicated-schema" }
    }
  }
}
```

## 7. Les jobs

- **`jobs/<tenant>/jobs.json`** déclare les jobs d'un tenant. Seuls les jobs `implementation.type: "script"` sont chargés. Chaque job déclare un tableau `triggers` (plutôt qu'un objet `trigger` unique, supprimé) — `cron`, `interval`, `once`, `manual`, `queue` ou `dependency`, en tout nombre et en tout mélange ; schéma complet dans le [guide développeur](../developer/task-scheduler-guide-v21-2026-11-19.md#trigger-types).
- **`jobs/platform-jobs.json`** déclare les jobs sans tenant.
- **Les scripts** sont résolus par rapport au **répertoire de base**, qui forme un « pool » partagé entre tenants : `"script": "jobs/reconciliation.js"` désigne `<répertoire de base>/jobs/reconciliation.js`. Un script manquant n'empêche pas le démarrage : seule l'exécution concernée échoue.

```json
{
  "jobs": [
    {
      "id": "tenant1-reconciliation",
      "implementation": { "type": "script", "script": "jobs/tenant1-reconciliation.js" },
      "triggers": [{ "id": "cron", "type": "cron", "expression": "0 0 2 * * ?" }],
      "parallelizable": false,
      "arguments": { "batchSize": 500 }
    }
  ]
}
```

Les déclencheurs `manual`/`queue`/`dependency` sont démarrés et arrêtés en mémoire sur chaque nœud, pas par Quartz : un changement les concernant met jusqu'à `TSS_EVENT_TRIGGER_REFRESH_SECONDS` (30 s par défaut) à atteindre tous les nœuds. `queue` a besoin de la connexion broker propre à TSS ([§5](#5-la-configuration-propre-à-tss), clés `TSS_QUEUE_*`) ; `dependency` a besoin du script `db/migration/tss_dependency.sql` appliqué sur la base TSS, en plus de `tss_job_execution.sql`.

Un `jobs/` vide ou absent n'est pas une erreur : TSS démarre sans job. Dans ce cas, le hot reload et l'API d'administration restent **désactivés jusqu'au prochain redémarrage**.

Au démarrage, TSS crée `jobs/<tenant>/` pour chaque tenant déclaré qui n'en a pas encore. Le répertoire est vide, ou contient un `jobs.json` tiré de `jobs/template.json` si ce fichier existe.

Format complet : [guide développeur](../developer/task-scheduler-guide-v21-2026-11-19.md). Changements à chaud : [opérations](task-scheduling-system-v21-2026-11-15.md#changing-jobs-without-a-restart).

## 8. La commande de démarrage

### 8.1 Syntaxe

```
java [-D<propriété>=<valeur> ...] -jar payos-job-scheduler-<version>-standalone.jar [--scheduler-base-dir <chemin>] [run]
```

- `run` est la commande **par défaut** : `java -jar <jar>` et `java -jar <jar> run` sont strictement équivalents. Les autres commandes (`sync`, `list`, `trigger`) sont décrites au [§11](#11-la-ligne-de-commande).
- `--scheduler-base-dir` peut être placé avant ou après la commande. Sans lui, le répertoire de base est le répertoire de travail. Un chemin relatif est résolu par rapport au répertoire de travail.
- Les propriétés système `-D...` doivent précéder `-jar`. Placées après, elles deviennent des arguments du programme et ne sont pas prises en compte.

### 8.2 Linux

```bash
cd /opt/payos-tss
export TSS_DB_PASSWORD='<secret>'
export PAYOS_DB_PASSWORD='<secret>'          # utilisé par ${PAYOS_DB_PASSWORD} dans la configuration PayOS
java -jar payos-job-scheduler-1.1.3-RELEASE-standalone.jar --scheduler-base-dir /opt/payos-tss
```

### 8.3 Windows (PowerShell)

```powershell
Set-Location C:\payos-tss
$env:TSS_DB_PASSWORD = '<secret>'
$env:TSS_ADMIN_HTTP_PORT = '8095'           # optionnel : active l'API d'administration
java -jar payos-job-scheduler-1.1.3-RELEASE-standalone.jar --scheduler-base-dir C:\payos-tss
```

Les variables `$env:` restent positionnées dans la console après la commande. Une même console qui lance ensuite une autre configuration, par exemple la CLI contre une autre base, doit les retirer avec `Remove-Item Env:<NOM>`.

### 8.4 Ajouter un driver JDBC (par exemple Oracle)

`java -jar` ignore `-cp`. Pour ajouter un JAR, nommez la classe principale. Le séparateur de classpath est `:` sous Linux et `;` sous Windows :

```bash
java -cp "payos-job-scheduler-1.1.3-RELEASE-standalone.jar:/opt/drivers/ojdbc11.jar" \
  ma.s2m.payos.scheduler.bootstrap.StandaloneMain --scheduler-base-dir /opt/payos-tss
```

La même forme permet d'ajouter un chargeur de configuration ou une fabrique de bases. Le plus simple reste toutefois de déposer ces JARs dans le `service-adapters-dir` de PayOS ([§6](#6-la-configuration-payos-lue-par-tss)).

### 8.5 Comme service

Le processus reste au premier plan et se gère comme n'importe quel service Java (systemd, service Windows, conteneur). Sous systemd, mettez `WorkingDirectory` sur le répertoire de base, à cause des chemins relatifs du [§4](#4-le-répertoire-de-base), et placez les secrets dans `Environment=` ou `EnvironmentFile=`.

## 9. Séquence de démarrage et logs attendus

Dans l'ordre, un démarrage réussi affiche les lignes suivantes (préfixe de log omis) :

| Étape | Ligne de log |
| --- | --- |
| Répertoire de base | `TSS scheduler base directory: /opt/payos-tss` |
| Construction du scheduler | `Starting TSS standalone scheduler 'TSS-Cluster'` |
| Connecteurs (si `service-adapters-dir`) | `Service-adapter JAR registered: <jar>` (une ligne par JAR) |
| Base plateforme (si déclarée) | `Platform database service initialized with <n> mapping file(s)` |
| Réconciliation des jobs | `Job reconciliation (STARTUP): added=[...], rescheduled=[...], updated=[...], deleted=[...], skipped=[...], failed=[...]`, ou `scheduler already matches the jobs files` |
| Enregistrement | `Registered <n> job(s) across <m> tenant(s)` |
| Démarrage effectif | `TSS standalone scheduler started (isStarted=true)` |
| Hot reload (si activé) | `TSS jobs hot reload watching '<jobs>'` |
| API d'administration (si port > 0) | `TSS admin HTTP endpoint listening on port <port>` |
| Métriques (si port > 0) | `TSS standalone metrics HTTP server listening on port <port>` |

Sans aucun job déclaré, la ligne `No job declared at startup — jobs hot reload and the admin endpoint stay off until the next restart` remplace les trois dernières.

Au premier démarrage sur une base qui contient déjà des triggers (après un arrêt prolongé), Quartz rattrape les exécutions manquées : quelques exécutions immédiates, et éventuellement des lignes `MISFIRE` dans l'historique.

L'API d'administration et le serveur de métriques sont optionnels : s'ils ne peuvent pas démarrer (par exemple parce que le port est occupé), un `WARN` est logué et le scheduler continue sans eux.

## 10. Arrêt

`Ctrl+C` ou `SIGTERM` déclenche le hook d'arrêt, qui ferme dans l'ordre :
- le hot reload ;
- l'API d'administration ;
- le scheduler, en **attendant la fin des exécutions en cours** ;
- le serveur de métriques.

Évitez `kill -9` : il interrompt une exécution en cours sans passer par ce hook.

## 11. La ligne de commande

### 11.1 Synopsis

```
java -jar payos-job-scheduler-<version>-standalone.jar [--scheduler-base-dir <chemin>] <commande> [options]

  run                                               démarre le scheduler (défaut)
  sync                                              réconcilie le scheduler avec les fichiers jobs/ de cette machine
  list    [--tenant <id>]                           liste les jobs planifiés
  trigger (--tenant <id> | --platform) --script <chemin>    déclenche un job une fois, maintenant
```

### 11.2 Options

| Option | Commandes | Rôle |
| --- | --- | --- |
| `--scheduler-base-dir <chemin>` | toutes | Répertoire de base ([§4](#4-le-répertoire-de-base)). Sans valeur, il provoque une erreur. |
| `--tenant <id>` | `list`, `trigger` | `list` : filtre sur un tenant. `trigger` : tenant du job. |
| `--platform` | `trigger` | Le job est un job plateforme. Incompatible avec `--tenant`. Ignoré par `list`. |
| `--script <chemin>` | `trigger` | Chemin du script tel que déclaré dans `jobs.json`, par exemple `jobs/tenant1-reconciliation.js`. C'est lui qui identifie le job, avec le tenant. |

Les options peuvent être placées n'importe où. La commande est le premier argument qui n'est ni une option ni la valeur d'une option.

**Il n'existe pas d'option `--help`.** Une option inconnue, c'est-à-dire tout argument commençant par `--`, est ignorée sans message : `java -jar <jar> --help` **démarre donc le scheduler**. Pour afficher l'usage, passez une commande inconnue, par exemple `java -jar <jar> help`. Le processus s'arrête alors avec le code `2` en affichant :

```
Unknown command 'help'. Usage: [--scheduler-base-dir <path>] [run | sync | list [--tenant <id>] | trigger (--tenant <id> | --platform) --script <path>]
```

### 11.3 Fonctionnement commun de `sync`, `list` et `trigger`

- **Même configuration que le service.** Ces commandes lisent `tss-config.json` et les `TSS_*` exactement comme `run` ([§5](#5-la-configuration-propre-à-tss)). Elles visent donc la base et le nom de scheduler configurés : avec un autre nom, elles ne voient aucun job.
- **Elles ne démarrent jamais de scheduler.** Elles ne déclenchent rien elles-mêmes et ne rejoignent pas le cluster. On peut donc les lancer depuis n'importe quelle machine qui accède à la base TSS, pendant que les nœuds tournent. Leur contrôle d'accès est celui des identifiants de la base TSS.
- **Audit.** Chaque `sync` et `trigger` est audité (`TSS_JOBS_SYNCHRONIZED`, `TSS_JOB_TRIGGERED`, canal `cli`) sous le nom de l'utilisateur du système d'exploitation.
- **Sortie.** Le résultat est écrit en JSON indenté sur la sortie standard. Les logs et les erreurs vont sur la console.

### 11.4 `list`

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss list --tenant tenant1
```

```json
[ {
  "tenantId" : "tenant1",
  "scriptPath" : "jobs/tenant1-db-touch.js",
  "jobId" : "tenant1-db-touch",
  "triggers" : [ {
    "id" : "cron",
    "type" : "cron",
    "definition" : { "id" : "cron", "type" : "cron", "expression" : "15 * * * * ?" },
    "nextFireTime" : "2026-11-13T10:13:15Z",
    "previousFireTime" : "2026-11-13T10:12:15Z"
  } ],
  "parallelizable" : true,
  "nextFireTime" : "2026-11-13T10:13:15Z",
  "previousFireTime" : "2026-11-13T10:12:15Z"
} ]
```

`triggers` liste chaque déclencheur déclaré sur ce job — un déclencheur événementiel (`manual`/`queue`/`dependency`) y apparaît avec `"nextFireTime": null`/`"previousFireTime": null`, puisqu'il n'a pas d'échéance propre dans Quartz. Les champs `nextFireTime`/`previousFireTime` au niveau du job sont le plus proche/le plus récent parmi ses seuls déclencheurs temporels. Sans `--tenant`, tous les jobs sont listés. Un job plateforme a `"tenantId" : null`.

### 11.5 `trigger`

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss trigger --tenant tenant1 --script jobs/tenant1-reconciliation.js
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss trigger --platform --script jobs/platform-heartbeat.js
```

```json
{
  "status" : "TRIGGERED",
  "tenantId" : "tenant1",
  "scriptPath" : "jobs/tenant1-reconciliation.js"
}
```

- Le job s'exécute une fois, immédiatement, **sur un nœud en marche**. La commande elle-même n'exécute rien : sans nœud démarré, rien ne s'exécute.
- Toutes les règles d'une exécution planifiée s'appliquent : exclusivité d'un job `parallelizable: false`, quota par tenant, historique, et métriques avec le tag `triggerType=manual`.
- Le cron du job n'est pas modifié.
- Il faut `--script` et **exactement un** de `--tenant` ou `--platform`, sinon le code de sortie est `2`.
- Un job introuvable (mauvais tenant, mauvais chemin, autre scheduler) donne le code `3`.

### 11.6 `sync`

```bash
java -jar payos-job-scheduler-<version>-standalone.jar --scheduler-base-dir /opt/payos-tss sync
```

`sync` relit les fichiers `jobs/` **de la machine où la commande est lancée** et applique les différences au scheduler partagé :
- ajout des nouveaux jobs ;
- application des changements de cron, de `parallelizable` et d'`arguments` ;
- suppression des jobs qui ne sont plus déclarés.

Il crée aussi les répertoires des nouveaux tenants. C'est l'alternative au hot reload quand celui-ci est désactivé, ou quand les fichiers sont sur un volume qui ne signale pas les modifications (NFS, ConfigMap Kubernetes).

```json
{
  "added" : [ "tenant1/jobs/tenant1-reconciliation.js" ],
  "rescheduled" : [ ],
  "updated" : [ ],
  "deleted" : [ ],
  "skipped" : [ ],
  "failed" : [ ]
}
```

Les jobs d'un tenant qui n'a encore aucun job dans le scheduler sont classés dans `skipped`, avec le motif `new tenant: restart the scheduler nodes to open its database`. Ils ne sont planifiés qu'au redémarrage des nœuds. Le code de sortie vaut `1` si `failed` n'est pas vide.

### 11.7 Codes de sortie

| Code | Signification |
| --- | --- |
| `0` | Succès. |
| `1` | `sync` a échoué sur au moins un job, ou bien une erreur de configuration a été levée avant la connexion à la base (par exemple `Missing required configuration 'TSS_DB_URL'`). |
| `2` | Erreur d'usage : commande inconnue, ou `trigger` sans `--script` ou sans exactement un de `--tenant`/`--platform`. |
| `3` | `trigger` sur un job inconnu. |

`run` ne rend pas la main tant que le service tourne.

## 12. Plusieurs nœuds (cluster)

Pour un cluster, démarrez plusieurs processus avec :
- la **même base TSS** ;
- le **même `TSS_SCHEDULER_INSTANCE_NAME`** ;
- un **contenu `jobs/` identique** sur tous les nœuds.

Un seul nœud déclenche un trigger donné, et les jobs d'un nœud arrêté sont repris par les autres. Un nœud dont les fichiers `jobs/` diffèrent réconcilierait tout le cluster vers sa propre version à son démarrage. Utilisez un volume partagé, ou un artefact unique déployé par le même pipeline. Détails : [opérations, Clustering](task-scheduling-system-v21-2026-11-15.md#clustering).

## 13. Mode embarqué (TSS dans `payos-runtime`)

TSS peut aussi tourner dans le processus du runtime. Il n'y a alors pas de commande propre à TSS :

1. **Base TSS.** Créez-la et appliquez les mêmes scripts qu'au [§3](#3-la-base-de-données-de-tss).
2. **Bloc de configuration.** Ajoutez un bloc `task-scheduler` dans `payos.json`, ou dans un `*.json` **directement** dans le `configDir` du runtime. Un fichier placé dans un sous-répertoire est ignoré sans message et TSS ne démarre pas :

   ```json
   {
     "task-scheduler": {
       "enabled": true,
       "database": { "dbUrl": "jdbc:postgresql://db.internal:5432/tss", "dbUser": "tss_svc", "dbPassword": "${TSS_DB_PASSWORD}", "dbDriver": "org.postgresql.Driver", "dbVendor": "postgresql" },
       "scheduler": { "schedulerInstanceName": "TSS-Cluster", "threadCount": 5, "hotReloadEnabled": true },
       "admin": { "httpPort": 8095 }
     }
   }
   ```

3. **Jobs.** Ils sont dans `jobs/`, sous le répertoire du bundle (`--bundle-path`). Les tenants et les bases sont ceux déjà chargés par le runtime.
4. **Démarrage.** Il se fait avec le runtime : `java -jar payos-runtime-<version>.jar --bundle-path <bundle>`. Les lignes de log de TSS apparaissent dans celles du runtime. Un problème propre à TSS est logué sans arrêter le runtime.
5. **CLI.** Utilisez le jar standalone avec `TSS_DB_URL` (et les identifiants) **et** `TSS_SCHEDULER_INSTANCE_NAME` égaux aux valeurs du bloc `task-scheduler`.

Référence des clés : [configuration](../configuration/task-scheduling-system-v11-2026-11-15.md#embedded-mode--the-task-scheduler-block-of-payosjson).

## 14. Dépannage du démarrage

| Symptôme | Cause et solution |
| --- | --- |
| `Missing required configuration 'TSS_DB_URL'` (code 1) | Aucun fichier de configuration n'a été trouvé (la ligne `No TSS config file at '<chemin>'` le précède), ou bien il n'a pas de `database.dbUrl`. Vérifiez `--scheduler-base-dir` et `TSS_CONFIG_FILE` ([§5](#5-la-configuration-propre-à-tss)), ou définissez `TSS_DB_URL`. |
| `WARN TSS config file found at '…', relative to the working directory` | Le fichier a été trouvé à l'ancien emplacement (le répertoire de travail), pas sous le répertoire de base. Il est utilisé : déplacez-le sous le répertoire de base, ou rendez `TSS_CONFIG_FILE` absolu. |
| `Failed to parse TSS config file '<fichier>'` | JSON invalide dans `tss-config.json`. |
| `Table "QRTZ_..." not found` ou `relation "qrtz_..." does not exist` | Les scripts du [§3](#3-la-base-de-données-de-tss) n'ont pas été appliqués sur cette base ou ce schéma. Vérifiez aussi `TSS_DB_SCHEMA`. |
| `MappingNotFoundException: Mapping (RESOURCE) not found : <mapping>` | Un fichier de mapping déclaré pour un tenant ou pour la base plateforme n'existe ni sur disque, ni dans le jar, ni dans les JARs de `service-adapters-dir` ([§6](#6-la-configuration-payos-lue-par-tss)). |
| Un tenant « not declared under multitenancy.tenants » | Un répertoire `jobs/<tenant>/` existe pour un tenant absent de la configuration PayOS. Déclarez le tenant ou supprimez son répertoire. |
| **Après une exception au démarrage, le processus ne se termine pas** | Défaut connu : les threads de Quartz sont créés avant l'enregistrement des jobs et ne sont pas arrêtés quand celui-ci échoue. La JVM reste donc vivante après l'exception (`MappingNotFoundException`, tenant invalide, base inaccessible...). Arrêtez le processus (`Ctrl+C`), corrigez, puis relancez. Sous un gestionnaire de services, ne vous fiez pas à une sortie du processus pour détecter cet échec : surveillez la ligne `TSS standalone scheduler started`. |
| `java -jar <jar> --help` démarre le scheduler | Normal : il n'y a pas d'option d'aide ([§11.2](#112-options)). |
| `list` vide ou `trigger` en code `3` alors que des jobs tournent | La CLI vise une autre base ou un autre nom de scheduler ([§11.3](#113-fonctionnement-commun-de-sync-list-et-trigger)). |
| Hot reload ou API d'administration absents | Aucun job n'était déclaré au démarrage ([§7](#7-les-jobs)), ou bien `TSS_HOT_RELOAD_ENABLED=false` ou `TSS_ADMIN_HTTP_PORT` absent. Pour l'API, cherchez aussi un `WARN` « Failed to start the TSS admin HTTP endpoint ». |
| Driver Oracle introuvable | Il n'est pas inclus dans le jar : voir [§8.4](#84-ajouter-un-driver-jdbc-par-exemple-oracle). |

Autres symptômes en fonctionnement : [opérations, Troubleshooting](task-scheduling-system-v21-2026-11-15.md#troubleshooting).

## 15. Voir aussi

- [Opérations TSS](task-scheduling-system-v21-2026-11-15.md) — modes de déploiement, réconciliation, hot reload, administration, historique, métriques, cluster.
- [Configuration TSS](../configuration/task-scheduling-system-v11-2026-11-15.md) — référence des clés.
- [Guide développeur TSS](../developer/task-scheduler-guide-v21-2026-11-19.md) — écrire et configurer des jobs.
- [Diagramme de classes TSS](../architecture/job-scheduler-class-diagram-v24-2026-11-15.md) — conception interne.
