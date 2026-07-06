# Architecture de messagerie PayOS (Queue Server / Queue Client / Broker NATS)

> **Source de vérité : le code.** Chaque affirmation ci-dessous est ancrée sur un fichier/ligne précis. Quand la documentation existante (`payos-docs`) contredit le code, c'est signalé explicitement en fin de document (section [Divergences doc/code](#divergences-docscode-à-connaître)).

## 1. Le point clé à retenir

Il n'y a **pas** de serveur HTTP, un serveur TCP et un "Queue Server" qui tournent dans des process séparés qui s'appellent entre eux. Il y a **un seul process JVM** (`payos-runtime`), qui démarre en interne plusieurs "listeners" (HTTP, TCP, Queue) selon la configuration. Le "Queue Server" est juste un de ces listeners.

```mermaid
graph TB
    subgraph JVM["Un seul process JVM — payos-runtime (uber-jar shaded)"]
        BOOT["BootServer.main()
        (classe payos-kernel)"]

        subgraph SRV["Listeners démarrés selon config servers[]"]
            HTTP["HttpServer
            (payos-server-http)"]
            TCP["TcpServer
            (payos-server-tcp)"]
            QSRV["QueueServer
            (payos-server-queue)"]
        end

        subgraph QC["Clients de queue (IQueueClient)"]
            QC1["NatsQueueClient #1
            détenu par PayOSConfig
            → exposé aux scripts comme $Queue"]
            QC2["NatsQueueClient #2
            détenu par QueueServer
            → écoute consumer-topic"]
        end

        BOOT -->|"Servers.start(protocol,...)
        1 fois par entrée servers[]"| HTTP
        BOOT --> TCP
        BOOT --> QSRV
        BOOT -->|"QueueServiceInitializer.initialize()
        au boot + à chaque reload config"| QC1
        QSRV -->|"détient et pilote"| QC2
    end

    QC1 -->|"nats://host:port"| NATS[("Broker NATS
    (externe, ex. nats.prod.internal:4222)")]
    QC2 -->|"nats://host:port"| NATS

    style NATS fill:#2b2b2b,color:#fff,stroke:#888
```

**Conséquence importante :** il existe généralement **deux instances distinctes** de `NatsQueueClient` dans le même process, chacune avec sa propre connexion et son propre topic :

| Instance | Détenteur | Topic utilisé | Alimenté par |
|---|---|---|---|
| Client "publisher" | `PayOSConfig` (statique) | `publisher-topic` | `queue-service.configuration` (config globale) |
| Client "consumer" | `QueueServer` | `consumer-topic` | l'entrée `servers[]` de type `"queue"` |

Ce sont deux objets `NatsQueueClient` différents, connectés au même broker NATS, mais pas au même sujet — d'où une source fréquente de confusion ("j'ai publié mais mon serveur queue ne reçoit rien" → normal, ce n'est pas la même connexion/sujet).

---

## 2. Qui implémente quoi — le contrat `IQueueClient`

```mermaid
classDiagram
    class IQueueClient {
        <<interface>>
        +connect(host, port, topic)
        +disconnect()
        +isConnected() boolean
        +publish(message)
        +publish(message, replyTopic)
        +publish(destination, QueueMessage)
        +subscribe(MessageListener)
        +subscribe(destination, AckMessageListener)
        +subscribe(List~destination~, AckMessageListener) : default, fan-out
    }

    class NatsQueueClient {
        -Connection connection
        -String subject
        +connect(host, port, topic)
        +publish(message)
        +publish(destination, QueueMessage) JetStream
        +subscribe(destination, AckMessageListener) JetStream durable
    }

    class IQueueClientFactory {
        <<interface>>
        +type() String
        +create() IQueueClient
    }

    class NatsQueueClientFactory {
        +type() "nats"
        +create() new NatsQueueClient()
    }

    class QueueClients {
        <<static>>
        +create(selector) IQueueClient
        -loadFactories() : ServiceLoader~IQueueClientFactory~
    }

    IQueueClient <|.. NatsQueueClient : implements
    IQueueClientFactory <|.. NatsQueueClientFactory : implements
    NatsQueueClientFactory ..> NatsQueueClient : create()
    QueueClients ..> IQueueClientFactory : ServiceLoader.load(...)
```

- Interface : `payos/src/main/java/ma/s2m/payos/queue/IQueueClient.java`
- Implémentation NATS : `queue-service-nats/src/main/java/ma/s2m/payos/queue/nats/NatsQueueClient.java:38` — encapsule directement le client Java `io.nats:jnats` (`Nats.connect(...)` ligne 74).
- `NatsQueueClient` expose **deux API** dans la même classe :
  - API "legacy" simple : `publish(String)` / `publish(String, replyTopic)` / `subscribe(MessageListener)` — NATS core, un seul `subject` fixé à la connexion.
  - API "broker-agnostic" plus récente, basée sur **JetStream** : `publish(destination, QueueMessage)` / `subscribe(destination, AckMessageListener)` — crée un flux JetStream par destination, avec un abonnement "durable" nommé `destination + "-durable"`.
- Aucune autre implémentation "de production" de `IQueueClient` n'existe dans les repos actuels (pas de Kafka, pas d'in-memory prod). Seuls des doubles de test existent (`FakeQueueClient` dans `payos-service-notification` et `payos-notification-connector`), ce qui confirme que `IQueueClient` est conçu comme une stratégie interchangeable — NATS est aujourd'hui la seule brique réelle branchée derrière.

---

## 3. Qui instancie qui — mécanisme `ServiceLoader` (SPI), pas de Spring/CDI

Personne n'écrit `new NatsQueueClient()` directement dans le code métier. La création passe par le **Java `ServiceLoader`** :

```mermaid
sequenceDiagram
    participant Boot as BootServer.main()
    participant QSI as QueueServiceInitializer
    participant QC as QueueClients (static)
    participant SL as ServiceLoader<IQueueClientFactory>
    participant Fac as NatsQueueClientFactory
    participant Nat as new NatsQueueClient()
    participant Cfg as PayOSConfig

    Boot->>QSI: initialize(queue-service.configuration)
    QSI->>QC: create("nats")
    QC->>SL: load(IQueueClientFactory.class)
    SL-->>QC: [NatsQueueClientFactory] (via META-INF/services)
    QC->>Fac: factory.type() == "nats" ?
    Fac->>Nat: create()
    Nat-->>QSI: instance IQueueClient
    QSI->>Nat: connect(host, port, publisher-topic)
    QSI->>Cfg: PayOSConfig.setQueueClient(instance)
    Note over Cfg: exposé aux scripts JS comme $Queue
```

Et, en parallèle, un **second** chemin de création pour le transport "queue" lui-même :

```mermaid
sequenceDiagram
    participant Boot as BootServer.main()
    participant Servers as Servers.start("queue",...)
    participant Prov as QueueServerProvider
    participant QC as QueueClients.create(type)
    participant QS as new QueueServer(client, topics)

    Boot->>Servers: start(host, port, "queue", serverConf)
    Servers->>Prov: ServiceLoader<ServerProvider> → protocol()=="queue"
    Prov->>QC: resolveQueueClient(serverConf)
    QC-->>Prov: nouvelle instance NatsQueueClient
    Prov->>QS: new QueueServer(queueClient, consumerTopics)
    Servers->>QS: server.start(host, port)
    QS->>QS: queueClient.connect(...) puis subscribe(consumerTopics, listener)
```

**Fichiers clés :**
- `payos/src/main/java/ma/s2m/payos/queue/QueueClients.java` — résolution par `ServiceLoader`, fallback sur `"nats"` (`IConfigSpec.DEFAULT_QUEUE_CLIENT_TYPE`).
- `queue-service-nats/src/main/resources/META-INF/services/ma.s2m.payos.queue.IQueueClientFactory` → contient `ma.s2m.payos.queue.nats.NatsQueueClientFactory`.
- `payos/src/main/java/ma/s2m/payos/queue/QueueServiceInitializer.java:70-73` — chemin "publisher" (`$Queue`).
- `payos-server-queue/src/main/java/ma/s2m/payos/servers/providers/QueueServerProvider.java:22-31` — chemin "consumer" (`QueueServer`).
- `payos/src/main/java/ma/s2m/payos/servers/Servers.java` — résout aussi les `ServerProvider` (`http`, `https`, `tcp`, `queue`) via `ServiceLoader`, appelé depuis `BootServer.startServers(...)`.
- `payos-runtime/pom.xml` — assemble tous ces modules (`payos-kernel`, `payos-server-http`, `payos-server-tcp`, `payos-server-queue`, `queue-service-nats`, ...) en un seul jar shaded dont le `mainClass` est `ma.s2m.payos.BootServer`. **`payos-runtime` ne contient aucun code Java propre** — c'est un module d'assemblage Maven.

---

## 4. D'où viennent les noms de topics (sujets NATS) ?

**Il n'existe aucune classe `TopicBuilder`/`TopicNameResolver` dans le code.** Les noms de topics sont des **chaînes de configuration statiques**, pas construites dynamiquement (pas de logique par tenant/type de message).

```mermaid
graph LR
    subgraph Config["payos.json (config applicative)"]
        A["queue-service.configuration
        { host, port, type,
          publisher-topic,
          consumer-topic }"]
        B["servers[] entrée type=queue
        { protocol: 'queue', host, port,
          type, consumer-topic }"]
    end

    A -->|"publisher-topic"| QSI["QueueServiceInitializer
    .initialize()"]
    QSI -->|"client.connect(host, port, publisherTopic)"| QC1["NatsQueueClient #1
    .subject = publisher-topic"]

    B -->|"consumer-topic (liste)"| QSP["QueueServerProvider
    .create()"]
    QSP -->|"new QueueServer(client, consumerTopics)"| QS["QueueServer"]
    QS -->|"subscribe(consumerTopics, listener)"| QC2["NatsQueueClient #2"]

    style A fill:#1e3a5f,color:#fff
    style B fill:#1e3a5f,color:#fff
```

- Clés de config : `payos/src/main/java/ma/s2m/payos/config/IConfigSpec.java` (bloc `QueueService.Configuration` : `NAME`, `TYPE`, `PUBLISHER_TOPIC`, `CONSUMER_TOPIC`, `HOST`, `PORT`).
- Exemple concret (`payos-docs/configuration/queue-service.md`) :
  ```json
  { "queue-service": { "configuration": {
      "enabled": true, "name": "primary", "type": "nats",
      "host": "localhost", "port": 4222,
      "publisher-topic": "payos.events", "consumer-topic": "payos.requests"
  } } }
  ```
- Pour l'API JetStream "broker-agnostic" (`publish(destination, ...)` / `subscribe(destination, ...)`), la `destination` est une **chaîne fournie littéralement par l'appelant** (script métier ou config), simplement validée par une regex `[A-Za-z0-9_-]+` (pas de points, pas de wildcards). Il n'y a pas de préfixe/convention de nommage imposé par le framework.
- **Par défaut**, si aucun `consumer-topic` n'est configuré : `QueueServer.DEFAULT_TOPIC = "default-topic"`.

---

## 5. Flux de publication — trace complète (cas réel : "insight" auto-publié après chaque script API)

```mermaid
sequenceDiagram
    participant Client as Appelant HTTP/TCP
    participant ARH as ApiResourceHandler
    participant Engine as PolyglotScriptingEngine
    participant Script as Script JS (execute/emitInsight)
    participant Cfg as PayOSConfig
    participant NQC as NatsQueueClient (#1, publisher)
    participant NATS as Broker NATS

    Client->>ARH: requête API
    ARH->>Engine: executeScript(apiScript, request)
    Engine->>Script: execute(request) puis emitInsight(request, response, payload)
    Script-->>Engine: insight (objet JSON métier)
    Engine->>Engine: publishInsightToQueue(insight, request)
    Engine->>Cfg: PayOSConfig.getQueueClient()
    Cfg-->>Engine: instance NatsQueueClient (#1)
    Engine->>NQC: publish(insightJson)
    NQC->>NATS: connection.publish(subject=publisher-topic, bytes)
    Note over Engine,NQC: best-effort — erreur loguée, jamais propagée au client HTTP
```

- `PolyglotScriptingEngine.executeScript(...)` exige que le script API définisse trois fonctions JS : `execute`, `loadControlData`, `emitInsight`.
- `publishInsightToQueue(...)` récupère `PayOSConfig.getQueueClient()` puis appelle `queueClient.publish(insightJson)` → `NatsQueueClient.publish(String)` → publie sur le `subject` fixé à la connexion (= `publisher-topic`).
- Un script métier peut aussi publier explicitement via le binding `$Queue.publish(JSON.stringify(...))` (même chemin de code, un seul argument = le message).

---

## 6. Flux de souscription — trace complète (cas réel : `QueueServer` au démarrage)

```mermaid
sequenceDiagram
    participant Boot as BootServer.main()
    participant Servers as Servers.start("queue")
    participant Prov as QueueServerProvider
    participant QS as QueueServer
    participant NQC as NatsQueueClient (#2, consumer)
    participant NATS as Broker NATS
    participant App as Server.processRequest (pipeline commun HTTP/TCP/Queue)

    Boot->>Servers: start(host, port, "queue", serverConf)
    Servers->>Prov: create(serverConf)
    Prov->>QS: new QueueServer(client, consumerTopics)
    Servers->>QS: start(host, port)
    QS->>NQC: connect(host, port, consumerTopics[0])
    QS->>NQC: subscribe(consumerTopics, listener)
    NQC->>NATS: JetStream durable push-subscribe sur chaque topic
    NATS-->>NQC: message reçu
    NQC->>QS: listener.onMessage(message, acknowledgement)
    QS->>QS: parseRequest(message) → Request
    QS->>App: processRequest(appId, request)
    App-->>QS: response
    alt replyTo présent dans les métadonnées du message
        QS->>NQC: publish(replyTo, QueueMessage(response))
        NQC->>NATS: publish sur le sujet replyTo
    end
```

- Le "listener" est une lambda définie dans `QueueServer.start(...)` — c'est le callback `AckMessageListener` appelé par `NatsQueueClient` à chaque message reçu.
- Le message reçu est parsé en un `Request` canonique et traité par **le même pipeline** que les requêtes HTTP/TCP (`Server.processRequest` → `ResourceHandler`) — la queue est juste un transport de plus pour arriver au même moteur applicatif.
- Le "topic de réponse" (`replyTo`) n'est pas calculé : c'est une métadonnée portée par le message entrant, définie par l'émetteur d'origine (convention "reply-to" classique en messagerie).

---

## 7. Le broker : NATS, hébergé en dehors de ce dépôt

- Dépendance : `queue-service-nats/pom.xml` → `io.nats:jnats`.
- Aucune configuration d'infrastructure NATS (docker-compose, cluster) n'existe dans ces dépôts — le broker est un service externe, dont l'URL est injectée par variable d'environnement, ex. :
  ```
  NATS_URL=nats://nats.staging.internal:4222
  NATS_URL=nats://nats.prod.internal:4222
  ```
- Port par défaut : `4222` (port client standard NATS).
- `payosv2-packer` n'a **aucun rapport** avec la queue : c'est un utilitaire de chiffrement/déchiffrement de bundles de runtime, séparé du chemin de traitement des requêtes.

---

## Divergences doc/code à connaître

Deux endroits de `payos-docs` ne correspondent pas exactement au code réel — à corriger si vous éditez cette doc :

1. **`$Queue.publish(topic, message)` à deux arguments** (vu dans `payos-docs/developer/queue-messaging.md` et `scripting-bindings.md`) laisse penser que le premier argument est un nom de topic. **Ce n'est pas le cas** : la signature réelle est `publish(String message, String replyTopic)` — le premier argument est le **message**, le second un **topic de réponse optionnel** (confirmé par le test `NatsQueueClientLegacyPublishTest`). Seule la forme à un seul argument `$Queue.publish(json)` (documentée dans `javascript-api-endpoint-guide.md`) correspond au comportement réel.
2. **Clés de config `url`/`subject`/`credentials`** dans `payos-docs/configuration/example-multi-env.md` ne correspondent pas aux clés réellement lues par le code (`host`/`port`/`publisher-topic`/`consumer-topic`/`type` dans `IConfigSpec.QueueService.Configuration`). Cet exemple est probablement aspirationnel/obsolète.
