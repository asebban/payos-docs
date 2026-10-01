Created : 2026-09-30
Dernière mise à jour : 2026-10-01
Version : v7

# Diagramme de classes — Task Scheduling System (`payos-job-scheduler`)

**Audience :** architectes, tech leads, contributeurs à `payos-job-scheduler`, `database-service` et `payos-foundation`
**Périmètre :** toutes les classes de production créées jusqu'ici pour TSS (Stories 1.1 et 1.2), plus les classes d'appui touchées dans `database-service` et `payos-foundation`
**Document vivant :** ce diagramme sera mis à jour au fil des prochaines stories (1.3 à 1.5, puis Epic 2) — voir [§9](#9-état-davancement--ce-qui-nest-pas-encore-construit) pour ce qui manque encore.

---

## Table des matières

1. [Objectif et méthodologie](#1-objectif-et-méthodologie)
2. [Légende](#2-légende)
3. [Vue d'ensemble](#3-vue-densemble)
4. [`payos-foundation` — le contrat `IDatabaseService`](#4-payos-foundation--le-contrat-idatabaseservice)
5. [`database-service` — l'implémentation Hibernate](#5-database-service--limplémentation-hibernate)
6. [`payos-job-scheduler/jobstore` — configuration du `JobStore` Quartz](#6-payos-job-schedulerjobstore--configuration-du-jobstore-quartz)
7. [`payos-job-scheduler/bootstrap` — points d'entrée et cycle de vie](#7-payos-job-schedulerbootstrap--points-dentrée-et-cycle-de-vie)
8. [`payos-job-scheduler/registration` et `execution` — chargement et exécution des jobs](#8-payos-job-schedulerregistration-et-execution--chargement-et-exécution-des-jobs)
9. [État d'avancement — ce qui n'est pas encore construit](#9-état-davancement--ce-qui-nest-pas-encore-construit)
10. [Flux d'exécution complet, pas à pas](#10-flux-dexécution-complet-pas-à-pas)
11. [Un piège de nommage à connaître](#11-un-piège-de-nommage-à-connaître)
12. [Références](#12-références)

---

## 1. Objectif et méthodologie

Ce document complète [ADR-0009](adr/0009-task-scheduling-system.md) (la décision) et `payos-job-scheduler/_bmad-output/planning-artifacts/architecture.md` (l'architecture narrative propre au module, tenue par le workflow BMad). Là où ces deux documents expliquent *pourquoi* TSS est structuré ainsi, celui-ci répond à une question différente — *quelles classes existent, où vivent-elles, et comment se relient-elles* — au niveau du code réellement écrit, pas du plan.

Chaque classe listée a été localisée par lecture directe du code source (pas d'inférence à partir des noms), dans les répertoires suivants :
- `payos-job-scheduler/src/main/java/ma/s2m/payos/scheduler/{jobstore,bootstrap,registration,execution}/`
- `database-service/src/main/java/ma/s2m/` (`DynamicDataAccessService` et sa classe imbriquée `TenantConfig`)
- `payos-foundation/src/main/java/ma/s2m/payos/database/IDatabaseService.java`

Les classes de test (`*Test.java`) sont exclues du périmètre. Les classes `payos` (kernel) et Quartz apparaissent comme dépendances de contexte (TSS les appelle) mais ne sont pas elles-mêmes détaillées membre par membre — seules leurs méthodes effectivement utilisées par TSS sont montrées.

À ce stade (fin Story 1.2 + le post-implémentation `IJobConfigSpec`), le périmètre compte **21 classes/interfaces/records de production** portées par TSS et ses deux dépendances directement modifiées pour lui (`IDatabaseService`, `DynamicDataAccessService`) :

| Module | Classes |
| --- | --- |
| `payos-foundation` | `IDatabaseService`, `IDatabaseService.DedicatedTenantSession` (interface imbriquée) |
| `database-service` | `DynamicDataAccessService`, `DynamicDataAccessService.TenantConfig` (classe imbriquée) |
| `payos-job-scheduler` / `config` | `IJobConfigSpec` |
| `payos-job-scheduler` / `jobstore` | `DriverDelegateResolver`, `TssDataSourceConfig`, `JobStoreConfigurer` |
| `payos-job-scheduler` / `bootstrap` | `SchedulerBootstrap`, `StandaloneMain`, `RuntimeIntegrationHook`, `StandaloneConfigLoader` |
| `payos-job-scheduler` / `registration` | `JobDefinition`, `TenantConfig`, `TenantJobRegistry`, `JobConfigLoader` |
| `payos-job-scheduler` / `execution` | `JobDatabaseBinding`, `JobContext`, `ScriptExecutionDependencies`, `ScriptJob`, `ScriptJobFactory` |

---

## 2. Légende

| Stéréotype de module | Repository |
| --- | --- |
| `<<foundation>>` | `payos-foundation` |
| `<<kernel>>` | `payos` (référencé pour contexte — non modifié par TSS) |
| `<<database-service>>` | `database-service` |
| `<<job-scheduler>>` | `payos-job-scheduler` |
| `<<quartz>>` | Bibliothèque externe `org.quartz-scheduler:quartz` (pas du code PayOS) |

Chaque classe porte aussi un stéréotype de nature (`<<interface>>`, `<<record>>`, aucun pour une classe concrète ordinaire).

| Notation mermaid | Signification |
| --- | --- |
| `A --|> B` | `A` hérite de `B` (extends) |
| `A ..|> B` | `A` implémente l'interface `B` |
| `A ..> B : label` | `A` utilise/instancie/dépend de `B` |
| `A *-- B` | `A` possède `B` en composition |
| `A o-- B` | `A` possède `B` en agrégation (référence à un objet qui existe indépendamment) |

---

## 3. Vue d'ensemble

```mermaid
classDiagram
    class IDatabaseService {
        <<interface>>
        <<foundation>>
    }
    class DedicatedTenantSession {
        <<interface>>
        <<foundation>>
    }
    class DynamicDataAccessService {
        <<database-service>>
    }
    class JobDatabaseBinding {
        <<job-scheduler>>
    }
    class JobContext {
        <<job-scheduler>>
    }
    class ScriptExecutionDependencies {
        <<record>>
        <<job-scheduler>>
    }
    class ScriptJob {
        <<job-scheduler>>
    }
    class ScriptJobFactory {
        <<job-scheduler>>
    }
    class TenantJobRegistry {
        <<job-scheduler>>
    }
    class JobConfigLoader {
        <<job-scheduler>>
    }
    class SchedulerBootstrap {
        <<job-scheduler>>
    }
    class DatabaseBinding {
        <<kernel>>
    }
    class Job {
        <<interface>>
        <<quartz>>
    }
    class JobFactory {
        <<interface>>
        <<quartz>>
    }
    class Scheduler {
        <<interface>>
        <<quartz>>
    }

    IDatabaseService *-- DedicatedTenantSession : interface imbriquée
    DedicatedTenantSession --|> IDatabaseService
    DynamicDataAccessService ..|> IDatabaseService
    DynamicDataAccessService ..|> DedicatedTenantSession
    JobDatabaseBinding ..> IDatabaseService : délègue tout $DB
    JobContext *-- JobDatabaseBinding : construit $DB
    JobContext ..> DatabaseBinding : construit $PlatformDB (kernel, inchangée)
    ScriptExecutionDependencies o-- IDatabaseService : multiTenantDatabaseService, platformDatabaseService
    ScriptJob ..|> Job
    ScriptJob o-- ScriptExecutionDependencies : injectée au constructeur
    ScriptJob ..> DedicatedTenantSession : openDedicatedSessionFor(tenantId)
    ScriptJob ..> JobContext : crée par exécution
    ScriptJobFactory ..|> JobFactory
    ScriptJobFactory ..> ScriptJob : instancie avec dépendances
    SchedulerBootstrap ..> Scheduler : construit/démarre
    SchedulerBootstrap ..> ScriptJobFactory : installe via setJobFactory
    SchedulerBootstrap ..> TenantJobRegistry : planifie chaque job de
    JobConfigLoader ..> TenantJobRegistry : construit
```

Ce diagramme relie les classes pivots ; le détail complet de chacune est dans les sections suivantes.

---

## 4. `payos-foundation` — le contrat `IDatabaseService`

```mermaid
classDiagram
    class IDatabaseService {
        <<interface>>
        <<foundation>>
        +newParams() Map
        +list(query, params, firstResult, maxResults) List
        +list(query, params, maxResults) List
        +list(query, params) List
        +unique(query, params) Object
        +first(query, params) Object
        +executeUpdate(query, params) int
        +get(entityName, id) Map
        +getFresh(entityName, id) Map
        +save(entityName, entity) Object
        +update(entityName, entity) void
        +delete(entityName, entity) void
        +deleteById(entityName, id) void
        +findAll(entityName) List~Map~
        +find(query, params) List
        +setCurrentTenant(tenantId) void
        +clearCurrentTenant() void
        +beginRequestScope() boolean
        +endRequestScope() void
        +commitRequestScope() boolean
        +rollbackRequestScope() void
        +markRollbackOnly() void
        +setAutoCommit(autoCommit) void «deprecated»
        +isAutoCommit() boolean «deprecated»
        +beginTransaction() void «deprecated»
        +commitTransaction() void «deprecated»
        +rollbackTransaction() void «deprecated»
        +openDedicatedSessionFor(tenantId) DedicatedTenantSession
    }
    class DedicatedTenantSession {
        <<interface>>
        <<foundation>>
        +close() void
    }
    DedicatedTenantSession --|> IDatabaseService
    IDatabaseService *-- DedicatedTenantSession : interface imbriquée
```

**`IDatabaseService`** est le contrat unique derrière `$DB`/`$PlatformDB` côté kernel, et maintenant aussi derrière TSS — délibérément dépourvu de tout import Hibernate (ou de tout autre framework de persistance) dans ses signatures, pour qu'une implémentation puisse exister sans dépendre d'Hibernate.

**`openDedicatedSessionFor(String tenantId)`** (ajoutée pour Story 1.2) ouvre une session dédiée, manuellement transactionnable, pour un tenant donné — sans jamais exposer de type Hibernate à l'appelant. Implémentation par défaut : lève `UnsupportedOperationException` — seules les implémentations capables de router vers la base d'un tenant précis (comme `DynamicDataAccessService` en mode registre multi-tenant) la surchargent.

**`DedicatedTenantSession`** est une interface imbriquée : `IDatabaseService` + `close()`. C'est la seule différence avec l'interface parente — un appelant qui a ouvert une session dédiée doit pouvoir la refermer, et rien d'autre. `close()` doit annuler (rollback) toute transaction encore active avant de libérer la ressource — l'appelant n'a jamais besoin de sortir de cette interface pour s'en assurer.

*Historique de correction :* une première version de cet ajout exposait `openSessionForTenant(String): org.hibernate.Session` directement sur la classe concrète `DynamicDataAccessService`, puis sur une interface dédiée `TenantSessionSource` — deux approches qui couplaient structurellement `payos-job-scheduler` à une implémentation Hibernate. Voir `architecture.md` de `payos-job-scheduler` (section « Long-Running Job Execution & Manual Transaction Control ») pour le détail de cette correction en deux temps.

---

## 5. `database-service` — l'implémentation Hibernate

```mermaid
classDiagram
    class IDatabaseService {
        <<interface>>
        <<foundation>>
    }
    class DedicatedTenantSession {
        <<interface>>
        <<foundation>>
    }
    class DynamicDataAccessService {
        <<database-service>>
        -session Session
        -sessionFactory SessionFactory
        -tenantRegistry Map~String, TenantConfig~
        -tenantSessionFactories Map~String, SessionFactory~
        -platformScoped boolean
        -autoCommit boolean
        +DynamicDataAccessService(session)
        +DynamicDataAccessService(hibernateMappingFiles)
        +DynamicDataAccessService(tenantRegistry)
        +DynamicDataAccessService(platformConfig)
        +setCurrentTenantContext(tenantId) void$
        +clearCurrentTenantContext() void$
        +setCurrentTenant(tenantId) void
        +clearCurrentTenant() void
        +registerTenant(tenantId, tenantConfig) void
        +openDedicatedSessionFor(tenantId) DedicatedTenantSession
        +close() void
        ...CRUD IDatabaseService...
    }
    class TenantConfig {
        <<database-service>>
        -url String
        -username String
        -password String
        -driverClass String
        -dialect String
        -ddlAuto String
        -schema String
        -dataSource DataSource
        -hibernateMappingFiles String[]
    }
    DynamicDataAccessService ..|> IDatabaseService
    DynamicDataAccessService ..|> DedicatedTenantSession
    DynamicDataAccessService *-- TenantConfig : tenantRegistry (classe imbriquée)
```

**`DynamicDataAccessService`** a quatre modes de construction, tous déjà existants avant TSS (Story 1.1/1.2 n'en ajoutent aucun) :
1. **Session-backed** (`Session` explicite) — contrôle transactionnel manuel disponible.
2. **Fichiers de mapping Hibernate** — construit sa propre `SessionFactory`, aussi session-backed.
3. **Registre multi-tenant** (`Map<String, TenantConfig>`) — mode utilisé derrière `$DB` en requête API ; transaction possédée automatiquement par le runtime pour la durée de la requête.
4. **Config plateforme** (`TenantConfig` unique) — utilisé derrière `$PlatformDB`, filtrage tenant désactivé.

**`openDedicatedSessionFor(tenantId)`** (implémentation de l'interface, ajoutée pour TSS) ouvre une `Session` via le mécanisme interne déjà existant `openTenantSession`/`resolveTenantSessionFactory` (utilisés en interne par le mode 3), puis l'enveloppe dans **une nouvelle instance** de `DynamicDataAccessService` construite en mode 1 (session-backed) — c'est cette nouvelle instance, pas la `Session` brute, qui est retournée comme `DedicatedTenantSession`.

**`close()`** (implémentation de l'interface) n'a de sens que pour une instance session-backed — lève `IllegalStateException` sinon. Elle annule d'abord toute transaction encore active, puis ferme la `Session`.

**Découverte notable (Story 1.2) :** `CURRENT_TENANT` est un `ThreadLocal` **statique**, partagé par *toutes* les instances de `DynamicDataAccessService` sur un même thread — y compris une instance session-backed retournée par `openDedicatedSessionFor`. Appeler `save`/`get`/etc. sans avoir d'abord appelé `setCurrentTenant(tenantId)` sur cette instance échoue avec `IllegalStateException`, même en mode session-backed. C'est exactement pourquoi `ScriptJob` (§8) appelle `tenantSession.setCurrentTenant(tenantId)` avant toute opération `$DB`.

---

## 6. `payos-job-scheduler/jobstore` — configuration du `JobStore` Quartz

```mermaid
classDiagram
    class DriverDelegateResolver {
        <<job-scheduler>>
        +POSTGRESQL String$
        +ORACLE String$
        +resolve(vendor) String$
    }
    class TssDataSourceConfig {
        <<job-scheduler>>
        -jdbcUrl String
        -username String
        -password String
        -driverClassName String
        -vendor String
        -schema String
        -maxPoolSize int
        +getJdbcUrl() String
        +getUsername() String
        +getDriverClassName() String
        +getVendor() String
        +getSchema() String
        +getMaxPoolSize() int
    }
    class JobStoreConfigurer {
        <<job-scheduler>>
        +buildProperties(schedulerInstanceName, threadCount, dataSourceConfig) Properties$
    }
    class Properties {
        <<quartz>>
    }
    JobStoreConfigurer ..> TssDataSourceConfig : lit
    JobStoreConfigurer ..> DriverDelegateResolver : resolve(vendor)
    JobStoreConfigurer ..> Properties : construit
```

**`TssDataSourceConfig`** est la configuration de connexion propre à TSS, pour son propre schéma isolé (tables `QRTZ_*` + `tss_job_execution` à venir) — délibérément **pas** le `TenantConfig` de `DynamicDataAccessService` (voir [§11](#11-un-piège-de-nommage-à-connaître)), car ce schéma n'appartient ni à un tenant ni à la config plateforme.

**`DriverDelegateResolver`** décide seulement quel `DriverDelegate` Quartz utiliser (`StdJDBCDelegate` par défaut, `OracleDelegate` pour Oracle) — Quartz embarque déjà tous ses délégués, cette classe ne fait que choisir.

**`JobStoreConfigurer`** assemble les `java.util.Properties` pour un `JobStoreTX` clustered, en pointant Quartz vers le `DataSource` autonome de TSS (via le `HikariCpPoolingConnectionProvider` déjà embarqué dans Quartz — aucun `ConnectionProvider` custom nécessaire) — jamais via `DynamicDataAccessService` (voir architecture.md « Why not reuse DynamicDataAccessService directly »).

---

## 7. `payos-job-scheduler/bootstrap` — points d'entrée et cycle de vie

```mermaid
classDiagram
    class SchedulerBootstrap {
        <<job-scheduler>>
        -scheduler Scheduler
        +start(schedulerInstanceName, threadCount, dataSourceConfig) SchedulerBootstrap$
        +isStarted() boolean
        +scheduler() Scheduler
        +registerScriptJobs(registry, dependencies) void
        +shutdown(waitForJobsToComplete) void
    }
    class StandaloneMain {
        <<job-scheduler>>
        +main(args) void$
    }
    class RuntimeIntegrationHook {
        <<job-scheduler>>
        -bootstrap SchedulerBootstrap
        +start(schedulerInstanceName, threadCount, dataSourceConfig) void
        +isStarted() boolean
        +stop(waitForJobsToComplete) void
    }
    class StandaloneConfigLoader {
        <<job-scheduler>>
        -envLookup Function~String,String~
        -propertyLookup Function~String,String~
        -configFileContents JsonNode
        +load() StandaloneConfigLoader$
        +load(envLookup, propertyLookup) StandaloneConfigLoader$
        +resolve(key, jsonBlockName, jsonFieldName) String
    }
    class IJobConfigSpec {
        <<interface>>
        <<job-scheduler>>
        +CONFIG_FILE_KEY String$
        +DEFAULT_CONFIG_FILE String$
    }
    class TenantJobRegistry {
        <<job-scheduler>>
    }
    class ScriptExecutionDependencies {
        <<record>>
        <<job-scheduler>>
    }
    class Scheduler {
        <<interface>>
        <<quartz>>
        +setJobFactory(factory) void
        +scheduleJob(jobDetail, trigger) Date
        +start() void
        +shutdown(waitForJobsToComplete) void
    }
    class StdSchedulerFactory {
        <<quartz>>
    }
    class JobFactory {
        <<interface>>
        <<quartz>>
        +newJob(bundle, scheduler) Job
    }
    class JobRunShell {
        <<quartz>>
        +initialize(qs) void
        +run() void
    }
    class Job {
        <<interface>>
        <<quartz>>
        +execute(context) void
    }

    StandaloneMain ..> StandaloneConfigLoader : résout la config (3 niveaux)
    StandaloneMain ..> IJobConfigSpec : lit les clés (Database.*, Scheduler.*)
    StandaloneConfigLoader ..> IJobConfigSpec : CONFIG_FILE_KEY, DEFAULT_CONFIG_FILE
    StandaloneMain ..> SchedulerBootstrap : start(...)
    RuntimeIntegrationHook o-- SchedulerBootstrap : possède (0..1, une seule fois)
    SchedulerBootstrap ..> StdSchedulerFactory : construit
    SchedulerBootstrap *-- Scheduler
    SchedulerBootstrap ..> IJobConfigSpec : usingJobData(JobData.DATA_JOB_ID, ...)
    SchedulerBootstrap ..> TenantJobRegistry : registerScriptJobs(registry, ...)
    SchedulerBootstrap ..> ScriptExecutionDependencies : registerScriptJobs(..., dependencies)
    Scheduler o-- JobFactory : setJobFactory(...) — stockée sur QuartzScheduler, rien ne s'exécute encore
    JobRunShell ..> JobFactory : qs.getJobFactory().newJob(bundle, scheduler) au déclenchement d'un trigger
    JobRunShell ..> Job : run() appelle execute(context)
```

**`SchedulerBootstrap`** est le point de construction unique du scheduler Quartz — `StandaloneMain` et `RuntimeIntegrationHook` délèguent tous deux ici, pour que les deux modes de déploiement (processus autonome vs. embarqué dans `payos-runtime`) ne puissent jamais diverger.

**Le lien entre `Scheduler` et `JobFactory` (vérifié par désassemblage du jar Quartz 2.3.2, pas supposé) :** `scheduler.setJobFactory(factory)` (appelé par `SchedulerBootstrap.registerScriptJobs`) ne fait que stocker la référence dans un champ privé `jobFactory` sur `QuartzScheduler` (le moteur interne — `Scheduler` est l'interface publique, `StdScheduler` délègue) ; rien ne s'exécute à cet instant. Le lien s'active seulement plus tard, à chaque déclenchement d'un trigger : le thread interne du scheduler acquiert le trigger dû via le `JobStore`, construit un `TriggerFiredBundle`, et obtient un `JobRunShell` — un simple conteneur `(scheduler, bundle)`, sans `Job` instancié pour l'instant. C'est `JobRunShell.initialize(qs)` qui instancie réellement le job, en appelant exactement `qs.getJobFactory().newJob(firedTriggerBundle, scheduler)` — c'est **ce point précis** où `ScriptJobFactory.newJob(...)` s'exécute et construit le `ScriptJob` avec ses dépendances déjà injectées. Le `JobRunShell` (qui implémente `Runnable`) est ensuite soumis au pool de threads ; son `run()` construit le `JobExecutionContext` puis appelle `job.execute(jec)`, refermant la boucle vers `ScriptJob.execute(...)`.

**`registerScriptJobs(registry, dependencies)`** (ajoutée Story 1.2) installe d'abord un `ScriptJobFactory` (§8) sur le scheduler, puis planifie un couple `JobDetail`/`Trigger` par entrée de `registry`, avec pour `JobKey` le tuple `(scriptPath, tenantId)` — le même couple que le PRD a déjà établi comme identité unique d'un job, donc le modèle d'identité de Quartz l'impose gratuitement.

**`StandaloneMain`** est le point d'entrée du processus autonome — ne dépend que de ce module et de Quartz, aucune classe `payos-runtime` n'est référencée. La configuration passe par `StandaloneConfigLoader`.

**`RuntimeIntegrationHook`** est le point d'intégration pour un déploiement embarqué dans `payos-runtime` — mêmes garanties de construction que `StandaloneMain`, via `SchedulerBootstrap`.

**`StandaloneConfigLoader`** résout chaque clé de config à travers trois niveaux, dans l'ordre : variable d'environnement → propriété système → fichier JSON. Les lookups sont injectés (`Function<String,String>`) plutôt que d'appeler `System.getenv`/`System.getProperty` directement, pour rester testable.

**`IJobConfigSpec`** (`config/`, ajoutée post-Story 1.2 ; renommée depuis `ITSSConfigSpec` à la demande de l'utilisateur, le nom initial ne reflétant plus bien son périmètre une fois `JobsFile` ajoutée) centralise **tous** les noms de clés utilisés par TSS, de trois natures distinctes volontairement regroupées dans une seule interface :
- **Configuration externe** (`tss-config.json`, via `StandaloneConfigLoader`) — `Database` (`name = "database"`, puis `URL`/`USER`/`PASSWORD`/`DRIVER`/`VENDOR`/`SCHEMA`/`MAX_POOL_SIZE`, chacune avec son couple clé-env-ou-propriété + nom de champ JSON correspondant, ex. `URL = "TSS_DB_URL"` / `URL_JSON_FIELD = "dbUrl"`) et `Scheduler` (`name = "scheduler"`, puis `INSTANCE_NAME`, `THREAD_COUNT`, avec leurs valeurs par défaut), plus `CONFIG_FILE_KEY`/`DEFAULT_CONFIG_FILE` au niveau racine.
- **Schéma de `jobs.json`** (via `JobConfigLoader`, §8, document totalement différent de `tss-config.json`, jamais lu via `StandaloneConfigLoader`) — `JobsFile` (`name = "jobs"`, la clé du tableau racine), avec `JobsFile.Entry` (`ID`, `PARALLELIZABLE`) puis deux blocs imbriqués `JobsFile.Entry.Implementation` (`name = "implementation"`, `TYPE`, `TYPE_SCRIPT`, `SCRIPT`) et `JobsFile.Entry.Trigger` (`name = "trigger"`, `TYPE`, `TYPE_CRON`, `EXPRESSION`). Seules les valeurs MVP (`"script"`/`"cron"`) sont déclarées — les valeurs Growth-phase (`"java"`/`"queue"`) ne sont volontairement pas ajoutées tant qu'aucun code ne les lit, pour ne pas introduire de constantes mortes.
- **Clés internes** `JobData` — `jobId`/`tenantId`/`scriptPath` du `JobDataMap` Quartz (posées par `SchedulerBootstrap.registerScriptJobs` à la planification, relues par `ScriptJob.execute` à l'exécution — jamais sourcées d'une variable d'env/propriété/fichier, donc sans champ `name` : il n'y a pas de bloc JSON à nommer).

Modélisée sur le style de `IConfigSpec` du kernel (`payos-foundation`) — **y compris la convention `name` = clé du bloc JSON imbriqué** pour toute interface qui en représente un (ex. `tss-config.json` : `{"database": {"dbUrl": "..."}, "scheduler": {"schedulerInstanceName": "..."}}`, chaque champ `Database.name`/`Scheduler.name`/`JobsFile.name`/`...Implementation.name`/`...Trigger.name` nommant le bloc qui le contient, exactement comme `IConfigSpec.DatabaseService.name = "database-service"`) — mais **délibérément pas ajoutée à `IConfigSpec` lui-même** : ce dernier est le contrat du modèle `PayOSConfig`/`bootstrap.json` du kernel, un mécanisme de résolution complètement différent du modèle à trois niveaux propre à TSS en mode standalone, qui ne lit jamais `bootstrap.json`. Coupler les clés de TSS à `payos-foundation` aurait entamé l'indépendance de déploiement de ce module.

---

## 8. `payos-job-scheduler/registration` et `execution` — chargement et exécution des jobs

```mermaid
classDiagram
    class JobDefinition {
        <<record>>
        <<job-scheduler>>
        +id String
        +tenantId String
        +scriptPath String
        +cronExpression String
        +parallelizable boolean
    }
    class TenantConfig {
        <<job-scheduler>>
        -tenantId String
        -jobsByScriptPath Map~String, JobDefinition~
        +tenantId() String
        +jobs() Collection~JobDefinition~
        +jobForScriptPath(scriptPath) JobDefinition
    }
    class TenantJobRegistry {
        <<job-scheduler>>
        -tenantConfigs Map~String, TenantConfig~
        +forTenant(tenantId) TenantConfig
        +tenants() Map
        +allJobs() List~JobDefinition~
    }
    class JobConfigLoader {
        <<job-scheduler>>
        +load(jobsRootDir) TenantJobRegistry
    }
    class JobDatabaseBinding {
        <<job-scheduler>>
        -databaseService IDatabaseService
        +JobDatabaseBinding(databaseService)
        +beginTransaction() void
        +commitTransaction() void
        +rollbackTransaction() void
        +setAutoCommit(autoCommit) void
        +isAutoCommit() boolean
        ...CRUD délégué...
    }
    class JobContext {
        <<job-scheduler>>
        -jobId String
        -tenantId String
        -scriptPath String
        -firedAt Instant
        -db JobDatabaseBinding
        -platformDb DatabaseBinding
        +JobContext(jobId, tenantId, scriptPath, firedAt, tenantDatabaseService, platformDatabaseService)
        +getJobId() String
        +getTenantId() String
        +getScriptPath() String
        +getFiredAt() Instant
        +getDb() JobDatabaseBinding
        +getPlatformDb() DatabaseBinding
    }
    class ScriptExecutionDependencies {
        <<record>>
        <<job-scheduler>>
        +multiTenantDatabaseService IDatabaseService
        +platformDatabaseService IDatabaseService
        +scriptLoader Function~String,String~
    }
    class ScriptJob {
        <<job-scheduler>>
        -dependencies ScriptExecutionDependencies
        +ScriptJob()
        +ScriptJob(dependencies)
        +execute(quartzContext) void
        ~runJob(jobId, tenantId, scriptPath) void
        -cleanup(tenantSession, tenantContextSet, jobId, tenantId) void
        -invokeScript(scriptContent, scriptPath, jobContext) void
    }
    class ScriptJobFactory {
        <<job-scheduler>>
        -dependencies ScriptExecutionDependencies
        +ScriptJobFactory(dependencies)
        +newJob(bundle, scheduler) Job
    }
    class IDatabaseService {
        <<interface>>
        <<foundation>>
    }
    class DedicatedTenantSession {
        <<interface>>
        <<foundation>>
    }
    class DatabaseBinding {
        <<kernel>>
    }
    class IScriptingEngine {
        <<interface>>
        <<foundation>>
    }
    class ScriptingEngineFactory {
        <<kernel>>
    }
    class Job {
        <<interface>>
        <<quartz>>
    }
    class JobFactory {
        <<interface>>
        <<quartz>>
    }
    class IJobConfigSpec {
        <<interface>>
        <<job-scheduler>>
    }

    TenantJobRegistry o-- TenantConfig : par tenant
    TenantConfig o-- JobDefinition : par scriptPath
    JobConfigLoader ..> TenantJobRegistry : construit
    JobConfigLoader ..> TenantConfig : construit
    JobConfigLoader ..> JobDefinition : parse chaque entrée
    JobConfigLoader ..> IJobConfigSpec : lit JobsFile.Entry.Implementation/Trigger.*
    ScriptJob ..> IJobConfigSpec : lit JobData.DATA_JOB_ID/DATA_TENANT_ID/DATA_SCRIPT_PATH du JobDataMap

    JobDatabaseBinding ..> IDatabaseService : délègue tout, y compris begin/commit/rollbackTransaction
    JobContext *-- JobDatabaseBinding : $DB
    JobContext ..> DatabaseBinding : $PlatformDB (réutilisée telle quelle du kernel)
    ScriptExecutionDependencies o-- IDatabaseService : les deux champs base de données

    ScriptJob ..|> Job
    ScriptJob o-- ScriptExecutionDependencies
    ScriptJob ..> DedicatedTenantSession : openDedicatedSessionFor(tenantId)
    ScriptJob ..> JobContext : crée
    ScriptJob ..> IScriptingEngine : createNewEngine/evalScript/getMember
    ScriptJob ..> ScriptingEngineFactory : createNewEngine(...)
    ScriptJobFactory ..|> JobFactory
    ScriptJobFactory ..> ScriptJob : instancie avec dependencies injectées
```

### `registration/` — charger `jobs.json`

**`JobDefinition`** (record) : une entrée `jobs.json` supportée (`implementation.type: "script"` + `trigger.type: "cron"` uniquement — les autres combinaisons sont ignorées, pas rejetées, pour rester compatible avec un fichier qui mélange du contenu MVP et Growth-phase).

**`TenantConfig`** (`registration`, **à ne pas confondre** avec celle de `database-service` — [§11](#11-un-piège-de-nommage-à-connaître)) : la carte chemin-de-script → `JobDefinition` d'**un seul** tenant — un tenant déclare couramment plusieurs jobs.

**`TenantJobRegistry`** : le registre à travers **tous** les tenants — un `TenantConfig` par répertoire de tenant trouvé sous `jobs/`.

**`JobConfigLoader`** : parcourt `jobs/<tenantId>/jobs.json` (pas `custom-jobs.json` — Story 1.4). Le tenant est dérivé du nom de répertoire, littéralement — aucune vérification croisée avec un champ `tenant` déclaré dans le JSON pour l'instant. Tous les noms de clés qu'il lit (`jobs`, `id`, `parallelizable`, `implementation`/`type`/`script`, `trigger`/`type`/`expression`, plus les valeurs `"script"`/`"cron"`) viennent de `IJobConfigSpec.JobsFile` — plus aucun littéral dispersé dans cette classe.

### `execution/` — exécuter un job

**`JobDatabaseBinding`** : le binding `$DB` propre à TSS (distinct du `DatabaseBinding` du kernel) — délègue toute la surface CRUD **plus** le contrôle transactionnel manuel (`beginTransaction`/`commitTransaction`/`rollbackTransaction`/`setAutoCommit`/`isAutoCommit`), volontairement exclu du `DatabaseBinding` du kernel pour les scripts API mais sûr ici puisqu'une exécution de job ne partage jamais sa transaction avec un autre script.

**`JobContext`** : l'objet de contexte passé à la fonction `execute(context)` du script — construit `$DB` (`JobDatabaseBinding`) et `$PlatformDB` (`DatabaseBinding` du kernel, réutilisée sans modification, `null`-safe si aucune base plateforme n'est configurée).

**`ScriptExecutionDependencies`** (record) : les dépendances partagées entre toutes les exécutions — `multiTenantDatabaseService`, `platformDatabaseService` (tous deux `IDatabaseService`, jamais un type Hibernate), et `scriptLoader` (chemin de script → contenu, injecté pour rester testable).

**`ScriptJob`** : l'adaptateur Quartz `Job`. `execute(JobExecutionContext)` est un adaptateur mince qui lit le `JobDataMap` puis délègue à `runJob(jobId, tenantId, scriptPath)` (package-visible, testable sans vrai `JobExecutionContext`). `runJob` ouvre une session dédiée (`openDedicatedSessionFor`), positionne le tenant ambiant, construit le `JobContext`, invoque le script, puis — **inconditionnellement**, dans un seul bloc `finally` sans chemin de sortie anticipée — ferme la session (ce qui annule toute transaction encore active) et efface le contexte tenant. C'est le point le plus critique en sécurité de Story 1.2 : Quartz réutilise ses threads de travail entre exécutions de tenants différents ; sauter ce nettoyage sur un chemin quelconque est une fuite de données inter-tenant, pas un bug mineur.

**`ScriptJobFactory`** : une `JobFactory` Quartz qui construit `ScriptJob` avec ses dépendances injectées par constructeur — nécessaire car la `JobFactory` par défaut de Quartz n'instancie les jobs que par réflexion sans argument. Installée une fois par `SchedulerBootstrap.registerScriptJobs(...)`.

---

## 9. État d'avancement — ce qui n'est pas encore construit

Ce diagramme couvre les Stories 1.1 (infrastructure de clustering) et 1.2 (enregistrement + exécution d'un job script sur planification cron). Restent, dans l'Epic 1 :

- **Câblage `registerScriptJobs` en production (trouvé par revue de code, 2026-10-01) — bloquant avant tout déploiement réel.** `SchedulerBootstrap.registerScriptJobs(registry, dependencies)` est construite et testée, mais **rien dans `StandaloneMain`/`RuntimeIntegrationHook` ne l'appelle** — un déploiement réel démarre un scheduler qui n'exécute jamais aucun job. Bloqué sur une question non résolue : d'où vient le registre tenant (`Map<String, TenantConfig>`) en mode standalone, où aucun processus PayOS n'est déjà en cours d'exécution pour le fournir ? Trois options identifiées (lire le `bootstrap.json` de PayOS directement ; un bloc dédié dans `tss-config.json` ; différer à une story ultérieure) — l'utilisateur a choisi de différer plutôt que de trancher sans plus de contexte. Voir la story 1.2 (note de gap sur la Task 5 et l'AC1) pour le détail complet.
- **Story 1.3** — enregistrement de niveau plateforme (jobs sans tenant) et accès cross-tenant en lecture seule.
- **Story 1.4** — `custom-jobs.json`, logique de surcharge tenant-scopée, `RegistrationValidationException` (mentionnée dans architecture.md mais pas encore créée).
- **Story 1.5** — application réelle du flag `parallelizable` via une deuxième classe de job (`ExclusiveScriptJob`, avec `@DisallowConcurrentExecution`) — `ScriptJob` planifie aujourd'hui **tous** les jobs comme parallélisables, quel que soit ce que dit `jobs.json`.

Et dans l'Epic 2 (observabilité) : `history/` (table `tss_job_execution`, `JobExecutionRepository`), émission `PayOSMetrics`, masquage des données sensibles dans les logs/métriques d'exécution.

Ce document sera mis à jour à chaque nouvelle story qui ajoute ou modifie des classes de production.

---

## 10. Flux d'exécution complet, pas à pas

1. **Démarrage** — `StandaloneMain.main(...)` (ou `RuntimeIntegrationHook.start(...)` si embarqué) résout la config, appelle `SchedulerBootstrap.start(...)`, qui construit les `Properties` via `JobStoreConfigurer` (lui-même appelant `DriverDelegateResolver`) et démarre un `Scheduler` Quartz clustered (`JobStoreTX`).
2. **Chargement des jobs** — l'appelant (typiquement `StandaloneMain`, étendu en Story 1.2) charge `TenantJobRegistry registry = new JobConfigLoader().load(jobsRootDir)`.
3. **Planification** — `bootstrap.registerScriptJobs(registry, dependencies)` installe `ScriptJobFactory` puis, pour chaque `JobDefinition` de `registry.allJobs()`, planifie un `JobDetail`(`ScriptJob`) + `Trigger` cron, avec pour `JobKey` `(scriptPath, tenantId)`.
4. **Déclenchement** — à l'heure du cron, le thread interne du scheduler acquiert le trigger dû et obtient un `JobRunShell` ; `JobRunShell.initialize(qs)` appelle `qs.getJobFactory().newJob(bundle, scheduler)` — c'est-à-dire `ScriptJobFactory.newJob(...)` (construit `new ScriptJob(dependencies)`, les dépendances installées à l'étape 3) — puis `JobRunShell.run()` appelle `ScriptJob.execute(quartzContext)` (détail complet et vérifié : [§7](#7-payos-job-schedulerbootstrap--points-dentrée-et-cycle-de-vie)).
5. **Résolution** — `execute` lit `jobId`/`tenantId`/`scriptPath` du `JobDataMap` et appelle `runJob(...)`.
6. **Ouverture de session** — `runJob` appelle `dependencies.multiTenantDatabaseService().openDedicatedSessionFor(tenantId)`, obtenant un `IDatabaseService.DedicatedTenantSession` dédié à cette seule exécution, puis `tenantSession.setCurrentTenant(tenantId)`.
7. **Construction du contexte** — `new JobContext(jobId, tenantId, scriptPath, Instant.now(), tenantSession, platformDatabaseService)`, qui construit en interne `$DB` (`JobDatabaseBinding`) et `$PlatformDB` (`DatabaseBinding`, si configurée).
8. **Chargement du script** — `dependencies.scriptLoader().apply(scriptPath)` résout le contenu du script (fichier, ou autre source selon l'implémentation injectée).
9. **Exécution** — `invokeScript` crée un moteur (`ScriptingEngineFactory.createNewEngine(ENGINE_GRAALVM_POLYGLOT)`), injecte `$DB`/`$PlatformDB` via `putMember`, appelle `evalScript(scriptContent, scriptPath)`, récupère la fonction `execute` via `getMember("execute")`, vérifie `canExecute()`, puis appelle `.execute(jobContext)` — jamais `executeScript()`, propre aux scripts API.
10. **Nettoyage (AC4, inconditionnel)** — que l'étape 9 réussisse, lève une exception du script (`PolyglotException`), ou que `$DB` n'ait jamais été touché : le bloc `finally` de `runJob` appelle `tenantSession.close()` (qui annule toute transaction encore active) puis `tenantSession.clearCurrentTenant()`.

---

## 11. Un piège de nommage à connaître

Deux classes distinctes, nommées `TenantConfig`, coexistent dans des packages différents — **ne pas les confondre à l'import** :

| Classe | Package | Rôle |
| --- | --- | --- |
| `ma.s2m.DynamicDataAccessService.TenantConfig` | `database-service` (classe imbriquée) | Descripteur de **connexion base de données** pour un tenant (URL, identifiants, dialecte, mappings Hibernate) |
| `ma.s2m.payos.scheduler.registration.TenantConfig` | `payos-job-scheduler` | Carte **chemin de script → `JobDefinition`** pour un tenant (rien à voir avec une connexion base de données) |

De même, `TssDataSourceConfig` (`payos-job-scheduler/jobstore`) ressemble par sa forme (URL/identifiants/pilote/dialecte/schéma) au `TenantConfig` de `database-service`, mais configure la base **propre à TSS** (schéma `QRTZ_*`), jamais une base tenant ou plateforme — les deux structures sont volontairement similaires pour la familiarité, sans lien de code entre elles.

---

## 12. Références

- [ADR-0009 — Task Scheduling System](adr/0009-task-scheduling-system.md)
- `payos-job-scheduler/_bmad-output/planning-artifacts/architecture.md` — architecture narrative complète du module, tenue par le workflow BMad (décisions, corrections, séquence d'implémentation)
- `payos-job-scheduler/_bmad-output/implementation-artifacts/1-1-resilient-clustered-scheduling-infrastructure.md` — Story 1.1 (infrastructure de clustering)
- `payos-job-scheduler/_bmad-output/implementation-artifacts/1-2-register-and-run-a-script-job-on-a-cron-schedule.md` — Story 1.2 (enregistrement + exécution), y compris le journal de correction architecturale (Debug Log) menant à `IDatabaseService.openDedicatedSessionFor`
- [Data architecture](data-architecture.md) — pour le fonctionnement complet de `DynamicDataAccessService` au-delà de ce que TSS en utilise
- [Connector framework class diagram](connector-framework-class-diagram-v1-2026-09-02.md) — document frère, même méthodologie, pour un autre sous-système
