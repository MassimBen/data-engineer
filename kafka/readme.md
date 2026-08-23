Voici la fiche de révision pour le **Module 6 — Streaming avec Apache Kafka**, au format Markdown prêt pour GitHub.

---

# Module 6 — Stream Processing avec Apache Kafka

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision du Module 6 : Traitement de flux (Streaming) avec Apache Kafka

---

## 📑 Table des matières

1. [Introduction au Stream Processing](#1-introduction-au-stream-processing)
2. [Introduction à Apache Kafka](#2-introduction-à-apache-kafka)
3. [Architecture de Kafka](#3-architecture-de-kafka)
4. [Installation avec Docker](#4-installation-avec-docker)
5. [Producers et Consumers](#5-producers-et-consumers)
6. [Kafka en Python (kafka-python / confluent-kafka)](#6-kafka-en-python)
7. [Topics, Partitions et Offsets](#7-topics-partitions-et-offsets)
8. [Consumer Groups](#8-consumer-groups)
9. [Sérialisation : JSON et Avro](#9-sérialisation--json-et-avro)
10. [Schema Registry](#10-schema-registry)
11. [Kafka Streams et Faust](#11-kafka-streams-et-faust)
12. [KSQL / ksqlDB](#12-ksql--ksqldb)
13. [Cas pratique : NYC Taxi en streaming](#13-cas-pratique--nyc-taxi-en-streaming)
14. [Kafka Connect](#14-kafka-connect)
15. [Aide-mémoire](#15-aide-mémoire-commandes-et-api)

---

## 1. Introduction au Stream Processing

### 1.1 Batch vs Streaming (rappel)

| Critère | Batch (Module 5) | Streaming (Module 6) |
|---------|------------------|----------------------|
| **Données** | Finies, bornées | Infinies, continues |
| **Latence** | Minutes / heures | Millisecondes / secondes |
| **Déclenchement** | Planifié (cron) | Événement par événement |
| **Outils** | Spark batch, dbt | Kafka, Flink, Spark Streaming |
| **Exemples** | Rapport quotidien | Détection de fraude, monitoring temps réel |

### 1.2 Cas d'usage du streaming

- **Détection de fraude** bancaire en temps réel
- **Monitoring** : logs, métriques, alertes
- **IoT** : capteurs, véhicules connectés
- **Clickstream** : analyse du comportement utilisateur en direct
- **Notifications** et recommandations instantanées

### 1.3 Le défi du streaming

- Les données arrivent **en continu**, sans fin
- Il faut traiter **événement par événement** (ou par micro-batch)
- Gérer les **événements en retard**, l'**ordre**, la **tolérance aux pannes**

---

## 2. Introduction à Apache Kafka

### 2.1 Qu'est-ce que Kafka ?

- Plateforme **open-source** de **streaming d'événements distribuée** (Apache, créée par LinkedIn)
- Agit comme un **journal de logs distribué** (append-only log)
- Permet de **publier, stocker et consommer** des flux d'événements de manière **durable** et **scalable**
- Devient le **système nerveux central** des architectures data modernes

### 2.2 Concepts clés

| Concept | Définition |
|---------|------------|
| **Event / Message** | Un enregistrement (clé, valeur, timestamp) |
| **Topic** | Catégorie/flux nommé où les messages sont publiés (≈ une table) |
| **Partition** | Subdivision d'un topic pour le parallélisme |
| **Offset** | Position d'un message dans une partition (immuable, croissant) |
| **Producer** | Application qui **publie** des messages dans un topic |
| **Consumer** | Application qui **lit** des messages depuis un topic |
| **Broker** | Serveur Kafka qui stocke et sert les données |
| **Cluster** | Ensemble de brokers travaillant ensemble |

### 2.3 Analogie simple

> Kafka = un **système postal distribué** :
> - Le **topic** = la boîte aux lettres thématique
> - Le **producer** = l'expéditeur
> - Le **consumer** = le destinataire
> - Le **broker** = le centre de tri
> - La **rétention** = le courrier reste disponible X jours même après lecture

---

## 3. Architecture de Kafka

### 3.1 Vue d'ensemble

```
┌───────────┐        ┌─────────────────────────────────┐        ┌───────────┐
│ Producer  │───────►│         KAFKA CLUSTER           │───────►│ Consumer  │
│  (app 1)  │        │  ┌─────────┐ ┌─────────┐        │        │  (app 2)  │
└───────────┘        │  │Broker 1 │ │Broker 2 │        │        └───────────┘
┌───────────┐        │  │ Topic A │ │ Topic A │        │        ┌───────────┐
│ Producer  │───────►│  │ Part. 0 │ │ Part. 1 │        │───────►│ Consumer  │
│  (app 2)  │        │  └─────────┘ └─────────┘        │        │  (app 3)  │
└───────────┘        └─────────────────────────────────┘        └───────────┘
```

### 3.2 Le log distribué

Un topic partitionné = plusieurs **logs ordonnés** :

```
Partition 0:  [msg0][msg1][msg2][msg3][msg4]  ──►  offsets 0,1,2,3,4
Partition 1:  [msg0][msg1][msg2]              ──►  offsets 0,1,2
Partition 2:  [msg0][msg1][msg2][msg3]        ──►  offsets 0,1,2,3
```

- Les messages sont **immuables** et **ordonnés dans une partition**
- **Pas d'ordre global** entre partitions !
- La **clé** du message détermine la partition (hash de la clé)

### 3.3 Réplication et tolérance aux pannes

- Chaque partition a un **leader** et des **réplicas** (followers)
- Facteur de réplication typique : **3**
- Si le leader tombe → un follower devient leader (**failover automatique**)
- `acks=all` : le producer attend la confirmation de tous les réplicas

### 3.4 Rétention des données

- Les messages sont **conservés** même après consommation !
- Rétention configurable : par **temps** (ex. 7 jours) ou par **taille**
- Différence majeure avec les files de messages classiques (RabbitMQ) : on peut **relire l'historique**

### 3.5 ZooKeeper vs KRaft

| | ZooKeeper (historique) | KRaft (moderne) |
|---|---|---|
| Rôle | Service externe de coordination | Métadonnées gérées par Kafka lui-même |
| Depuis | Origines de Kafka | Kafka 2.8+ (prod-ready 3.3+) |
| Avantage | — | Architecture simplifiée, plus scalable |

> 💡 Les versions récentes de Kafka utilisent **KRaft** : plus besoin de ZooKeeper !

---

## 4. Installation avec Docker

### 4.1 Docker Compose (KRaft, sans ZooKeeper)

```yaml
# docker-compose.yml
version: '3'
services:
  broker:
    image: confluentinc/cp-kafka:7.5.0
    hostname: broker
    container_name: broker
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@broker:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://broker:29092,CONTROLLER://broker:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
```

```bash
# Démarrer Kafka
docker compose up -d

# Vérifier
docker compose ps
docker logs broker
```

### 4.2 Commandes CLI de base

```bash
# Créer un topic
docker exec broker kafka-topics --create \
  --topic taxi-rides \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Lister les topics
docker exec broker kafka-topics --list --bootstrap-server localhost:9092

# Décrire un topic
docker exec broker kafka-topics --describe \
  --topic taxi-rides --bootstrap-server localhost:9092

# Producer en console (envoyer des messages)
docker exec -it broker kafka-console-producer \
  --topic taxi-rides --bootstrap-server localhost:9092

# Consumer en console (lire depuis le début)
docker exec -it broker kafka-console-consumer \
  --topic taxi-rides --bootstrap-server localhost:9092 \
  --from-beginning
```

---

## 5. Producers et Consumers

### 5.1 Le Producer

```
Application ──► Serializer ──► Partitioner ──► Broker (partition leader)
```

**Responsabilités :**
- **Sérialiser** la clé et la valeur (bytes)
- **Choisir la partition** : via la clé (hash) ou round-robin si pas de clé
- **Batcher** les messages pour l'efficacité
- Gérer les **retries** et les **acks**

**Configurations importantes :**

| Paramètre | Description | Valeurs |
|-----------|-------------|---------|
| `bootstrap.servers` | Adresses des brokers | `localhost:9092` |
| `acks` | Niveau de confirmation | `0`, `1`, `all` |
| `retries` | Nombre de tentatives | `3` (défaut) |
| `batch.size` | Taille du batch en bytes | `16384` |
| `linger.ms` | Attente avant envoi du batch | `0` |
| `key.serializer` | Sérialiseur de clé | `StringSerializer`... |
| `value.serializer` | Sérialiseur de valeur | `JsonSerializer`, `AvroSerializer`... |

**Garanties `acks` :**

| Valeur | Comportement | Garantie |
|--------|--------------|----------|
| `acks=0` | Pas d'attente | Perte possible (fire & forget) |
| `acks=1` | Confirmation du leader | Perte si le leader tombe juste après |
| `acks=all` | Confirmation de tous les réplicas | Durabilité maximale |

### 5.2 Le Consumer

**Responsabilités :**
- **S'abonner** à un ou plusieurs topics
- **Poller** les messages en boucle
- **Commiter les offsets** (position de lecture)
- **Désérialiser** les messages

**Configurations importantes :**

| Paramètre | Description |
|-----------|-------------|
| `group.id` | Identifiant du consumer group |
| `auto.offset.reset` | Où commencer si pas d'offset : `earliest` / `latest` |
| `enable.auto.commit` | Commit automatique des offsets (`true`/`false`) |
| `max.poll.records` | Nombre max de messages par poll |

### 5.3 Sémantiques de livraison

| Sémantique | Description | Comment |
|------------|-------------|---------|
| **At-most-once** | 0 ou 1 fois (perte possible) | Commit avant traitement |
| **At-least-once** | 1 fois minimum (doublons possibles) | Commit après traitement ✅ défaut courant |
| **Exactly-once** | Exactement 1 fois | Transactions Kafka + idempotence |

> 💡 En pratique : **at-least-once** + traitement **idempotent** (clés uniques, upserts) est le pattern le plus courant.

---

## 6. Kafka en Python

### 6.1 Les librairies

| Librairie | Description |
|-----------|-------------|
| `confluent-kafka-python` | Wrapper officiel Confluent (librdkafka, performant) ✅ recommandé |
| `kafka-python` | Pure Python, API simple |
| `faust-streaming` | Stream processing façon Kafka Streams en Python |

```bash
pip install confluent-kafka
```

### 6.2 Producer en Python

```python
from confluent_kafka import Producer
import json

config = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'taxi-producer'
}

producer = Producer(config)

def delivery_callback(err, msg):
    if err:
        print(f'❌ Erreur: {err}')
    else:
        print(f'✅ Message livré → {msg.topic()} [partition {msg.partition()}] offset {msg.offset()}')

# Envoi de messages
for ride in rides:
    producer.produce(
        topic='taxi-rides',
        key=str(ride['vendor_id']),          # la clé détermine la partition
        value=json.dumps(ride),
        callback=delivery_callback
    )

producer.flush()  # Attendre l'envoi de tous les messages
```

### 6.3 Consumer en Python

```python
from confluent_kafka import Consumer, KafkaError
import json

config = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'taxi-consumer-group',
    'auto.offset.reset': 'earliest',   # lire depuis le début
    'enable.auto.commit': True
}

consumer = Consumer(config)
consumer.subscribe(['taxi-rides'])

try:
    while True:
        msg = consumer.poll(timeout=1.0)   # attendre 1s max
        
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            print(f'Erreur: {msg.error()}')
            break
        
        ride = json.loads(msg.value().decode('utf-8'))
        print(f"Course reçue: {ride['ride_id']} — {ride['total_amount']}$")

except KeyboardInterrupt:
    pass
finally:
    consumer.close()   # commit final + quitte le group proprement
```

### 6.4 Pattern complet : lecture CSV → Kafka

```python
import csv, json
from confluent_kafka import Producer

producer = Producer({'bootstrap.servers': 'localhost:9092'})

with open('yellow_tripdata.csv') as f:
    reader = csv.DictReader(f)
    for row in reader:
        producer.produce(
            topic='taxi-rides',
            key=row['VendorID'],
            value=json.dumps(row)
        )
        producer.poll(0)   # traiter les callbacks en non-bloquant

producer.flush()
```

---

## 7. Topics, Partitions et Offsets

### 7.1 Les partitions en détail

```
Topic "taxi-rides" (3 partitions)

Producer ──key="A"──► Partition 0: [A1][A2][A3]...
         ──key="B"──► Partition 1: [B1][B2]...
         ──key="C"──► Partition 2: [C1][C2][C3][C4]...
```

- **Même clé → même partition** → ordre garanti par clé
- Sans clé → répartition **round-robin**
- Les partitions permettent le **parallélisme** (plusieurs consumers en même temps)

### 7.2 Combien de partitions ?

| Facteur | Considération |
|---------|---------------|
| Throughput | Plus de partitions = plus de parallélisme |
| Consumers | Max de consumers actifs par group = nb de partitions |
| Ordre | Ordre garanti uniquement **dans** une partition |
| Overhead | Trop de partitions = surcharge de métadonnées |

> ⚠️ On peut **augmenter** le nombre de partitions, mais **jamais le réduire** !

### 7.3 Les offsets

- Chaque message a un **offset** unique dans sa partition
- Le consumer **tracke sa position** via les offsets commités
- Offset stocké dans un topic interne : `__consumer_offsets`

```
Partition 0:  [0][1][2][3][4][5][6]
                       ▲
                  offset commité = 3
                  → prochain message lu = 4
```

- **`auto.offset.reset=earliest`** : relire tout l'historique
- **`auto.offset.reset=latest`** : lire seulement les nouveaux messages

---

## 8. Consumer Groups

### 8.1 Principe

- Un **consumer group** = ensemble de consumers qui se **partagent** les partitions d'un topic
- Chaque partition est lue par **un seul consumer** du group à la fois
- Permet la **scalabilité horizontale** de la consommation

```
Topic (4 partitions)          Consumer Group "analytics"
┌────────────┐                ┌──────────────┐
│ Partition 0│───────────────►│ Consumer 1   │
│ Partition 1│───────────────►│ Consumer 1   │
│ Partition 2│───────────────►│ Consumer 2   │
│ Partition 3│───────────────►│ Consumer 2   │
└────────────┘                └──────────────┘
```

### 8.2 Règles importantes

| Situation | Résultat |
|-----------|----------|
| 2 consumers, 4 partitions | Chacun lit 2 partitions |
| 4 consumers, 4 partitions | Chacun lit 1 partition (optimal) |
| 6 consumers, 4 partitions | 2 consumers **inactifs** (idle) |
| 2 groups différents | Chaque group reçoit **tous** les messages (pub/sub) |

### 8.3 Rebalancing

- Quand un consumer **rejoint/quitte** le group → **rebalance** des partitions
- Pendant le rebalance, la consommation est **en pause**
- Éviter les traitements trop longs (`max.poll.interval.ms`)

### 8.4 Deux patterns de consommation

```
PATTERN 1 — Load balancing (même group)      PATTERN 2 — Pub/Sub (groups différents)
                                             
Producer ──► Topic ──► Group A (C1, C2)      Producer ──► Topic ──► Group A (C1, C2)
                       → messages partagés                         ──► Group B (C3)
                                                                    → chaque group reçoit TOUT
```

---

## 9. Sérialisation : JSON et Avro

### 9.1 Pourquoi sérialiser ?

Kafka ne transporte que des **bytes**. Il faut convertir objets ↔ bytes.

| Format | Avantages | Inconvénients |
|--------|-----------|---------------|
| **JSON** | Lisible, simple, universel | Verbeux, pas de schéma strict, pas d'évolution contrôlée |
| **Avro** | Compact (binaire), schéma embarqué, évolution du schéma | Moins lisible, nécessite Schema Registry |
| **Protobuf** | Compact, typé, multi-langage | Setup plus complexe |

### 9.2 JSON (simple)

```python
import json

# Producer
producer.produce(topic='rides', value=json.dumps({'ride_id': 1, 'fare': 12.5}))

# Consumer
data = json.loads(msg.value().decode('utf-8'))
```

### 9.3 Avro (avec schéma)

```json
{
  "type": "record",
  "name": "Ride",
  "namespace": "com.zoomcamp.taxi",
  "fields": [
    {"name": "ride_id", "type": "string"},
    {"name": "vendor_id", "type": "int"},
    {"name": "pickup_datetime", "type": "long"},
    {"name": "total_amount", "type": "double"}
  ]
}
```

**Évolution de schéma** : Avro gère l'ajout/suppression de champs avec des **valeurs par défaut** et des règles de **compatibilité** (backward, forward, full).

---

## 10. Schema Registry

### 10.1 Principe

- Service **centralisé** qui stocke les **schémas** (Avro/Protobuf/JSON Schema)
- Le producer **enregistre** le schéma et n'envoie que l'**ID du schéma** + les données binaires
- Le consumer **récupère** le schéma via l'ID pour désérialiser
- Garantit la **compatibilité** entre versions de schémas

```
Producer ──(1) enregistre schéma──► Schema Registry
Producer ──(2) envoie [schema_id + données binaires]──► Kafka
Consumer ──(3) récupère schéma par ID──► Schema Registry
Consumer ──(4) lit message──► Kafka ──► désérialise
```

### 10.2 Docker Compose avec Schema Registry

```yaml
schema-registry:
  image: confluentinc/cp-schema-registry:7.5.0
  hostname: schema-registry
  container_name: schema-registry
  depends_on:
    - broker
  ports:
    - "8081:8081"
  environment:
    SCHEMA_REGISTRY_HOST_NAME: schema-registry
    SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker:29092'
    SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
```

### 10.3 Producer Avro en Python

```python
from confluent_kafka import Producer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

schema_registry_client = SchemaRegistryClient({'url': 'http://localhost:8081'})

with open('ride.avsc') as f:
    avro_schema = f.read()

avro_serializer = AvroSerializer(
    schema_registry_client,
    avro_schema
)

producer = Producer({'bootstrap.servers': 'localhost:9092'})

producer.produce(
    topic='taxi-rides-avro',
    key=str(ride['ride_id']),
    value=avro_serializer(ride, SerializationContext('taxi-rides-avro', MessageField.VALUE))
)
producer.flush()
```

### 10.4 Consumer Avro

```python
from confluent_kafka.schema_registry.avro import AvroDeserializer

avro_deserializer = AvroDeserializer(schema_registry_client, avro_schema)

ride = avro_deserializer(msg.value(), SerializationContext(msg.topic(), MessageField.VALUE))
```

---

## 11. Kafka Streams et Faust

### 11.1 Kafka Streams (Java/Scala)

- Librairie **client-side** pour le stream processing (pas de cluster séparé !)
- Opérations : `map`, `filter`, `groupByKey`, `window`, `join`, `aggregate`
- **Stateful processing** avec state stores locaux (RocksDB)

### 11.2 Faust (équivalent Python)

```python
import faust

app = faust.App('taxi-streams', broker='kafka://localhost:9092')

class Ride(faust.Record):
    ride_id: str
    total_amount: float
    pu_location_id: int

rides_topic = app.topic('taxi-rides', value_type=Ride)
high_fares_topic = app.topic('high-fares', value_type=Ride)

# Table agrégée (stateful)
revenue_by_zone = app.Table('revenue_by_zone', default=float)

@app.agent(rides_topic)
async def process(rides):
    async for ride in rides:
        # Filtrage
        if ride.total_amount > 50:
            await high_fares_topic.send(value=ride)
        # Agrégation
        revenue_by_zone[ride.pu_location_id] += ride.total_amount
```

### 11.3 Opérations de stream processing

| Opération | Type | Description |
|-----------|------|-------------|
| `filter` | Stateless | Garder certains événements |
| `map` | Stateless | Transformer chaque événement |
| `branch` | Stateless | Router vers plusieurs topics |
| `groupBy` / `aggregate` | **Stateful** | Comptages, sommes par clé |
| `window` | **Stateful** | Agrégations temporelles |
| `join` | **Stateful** | Joindre stream-stream ou stream-table |

### 11.4 Fenêtrage (windowing)

```
Tumbling window (5 min)     Hopping window           Session window
┌────┐┌────┐┌────┐          ┌──────┐                 ┌────┐  ┌──┐
│    ││    ││    │            ┌──────┐               │    │  │  │
└────┘└────┘└────┘              ┌──────┐             └────┘  └──┘
Non chevauchantes              Chevauchantes          Basées sur l'activité
```

---

## 12. KSQL / ksqlDB

### 12.1 Principe

- Moteur **SQL** pour le stream processing sur Kafka
- Pas de code Java/Python : du **SQL déclaratif** sur des streams !

### 12.2 Concepts

| Concept | Description |
|---------|-------------|
| **STREAM** | Flux d'événements immuables (append-only) |
| **TABLE** | Vue matérialisée de l'état courant par clé (changelog) |

### 12.3 Exemples

```sql
-- Créer un stream depuis un topic
CREATE STREAM taxi_rides (
    ride_id VARCHAR,
    vendor_id INT,
    total_amount DOUBLE,
    pu_location_id INT
) WITH (
    KAFKA_TOPIC='taxi-rides',
    VALUE_FORMAT='JSON'
);

-- Requête de filtrage (push query, continue)
SELECT * FROM taxi_rides
WHERE total_amount > 50
EMIT CHANGES;

-- Agrégation fenêtrée → nouvelle table
CREATE TABLE revenue_per_zone AS
SELECT
    pu_location_id,
    COUNT(*) AS trip_count,
    SUM(total_amount) AS total_revenue
FROM taxi_rides
WINDOW TUMBLING (SIZE 1 HOUR)
GROUP BY pu_location_id
EMIT CHANGES;
```

---

## 13. Cas pratique : NYC Taxi en streaming

### 13.1 Architecture du pipeline

```
┌──────────┐    ┌───────────────┐    ┌──────────────┐    ┌─────────────┐
│ CSV file │───►│   Producer    │───►│ Kafka topic  │───►│  Consumer / │
│ (rides)  │    │   (Python)    │    │ taxi-rides   │    │  Stream job │
└──────────┘    └───────────────┘    └──────────────┘    └──────┬──────┘
                                                                │
                                          ┌─────────────────────▼──────┐
                                          │  Agrégation par zone       │
                                          │  → topic "zone-revenue"    │
                                          └────────────────────────────┘
```

### 13.2 Producer : envoyer les courses

```python
import csv, json, time
from confluent_kafka import Producer

producer = Producer({'bootstrap.servers': 'localhost:9092'})

with open('green_tripdata.csv') as f:
    for row in csv.DictReader(f):
        producer.produce(
            topic='taxi-rides',
            key=row['PULocationID'],
            value=json.dumps(row)
        )
        producer.poll(0)
        time.sleep(0.01)   # simuler un flux temps réel

producer.flush()
```

### 13.3 Consumer : agrégation simple

```python
from collections import defaultdict
from confluent_kafka import Consumer
import json

consumer = Consumer({
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'revenue-calculator',
    'auto.offset.reset': 'earliest'
})
consumer.subscribe(['taxi-rides'])

revenue_by_zone = defaultdict(float)
count = 0

while True:
    msg = consumer.poll(1.0)
    if msg is None or msg.error():
        continue
    
    ride = json.loads(msg.value().decode('utf-8'))
    zone = ride['PULocationID']
    revenue_by_zone[zone] += float(ride['total_amount'])
    
    count += 1
    if count % 1000 == 0:
        top = sorted(revenue_by_zone.items(), key=lambda x: -x[1])[:5]
        print(f"Top 5 zones: {top}")
```

---

## 14. Kafka Connect

### 14.1 Principe

- Framework pour **connecter Kafka** à des systèmes externes **sans code**
- **Source connectors** : BDD, fichiers → Kafka
- **Sink connectors** : Kafka → BDD, S3, Elasticsearch, BigQuery...

```
Sources ──► [Source Connectors] ──► KAFKA ──► [Sink Connectors] ──► Destinations
(JDBC, Debezium,                    (JDBC, S3, GCS,
 files, MQTT...)                     Elasticsearch, BigQuery...)
```

### 14.2 Exemple : sink vers PostgreSQL (config JSON)

```json
{
  "name": "postgres-sink-taxi",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/taxi_db",
    "connection.user": "postgres",
    "connection.password": "secret",
    "topics": "taxi-rides",
    "auto.create": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_key"
  }
}
```

---

## 15. Aide-mémoire : commandes et API

### CLI Kafka

| Commande | Description |
|----------|-------------|
| `kafka-topics --create --topic T --partitions 3 --bootstrap-server localhost:9092` | Créer un topic |
| `kafka-topics --list --bootstrap-server localhost:9092` | Lister les topics |
| `kafka-topics --describe --topic T --bootstrap-server ...` | Détail d'un topic |
| `kafka-console-producer --topic T --bootstrap-server ...` | Producer console |
| `kafka-console-consumer --topic T --from-beginning --bootstrap-server ...` | Consumer console |
| `kafka-consumer-groups --list --bootstrap-server ...` | Lister les groups |
| `kafka-consumer-groups --describe --group G --bootstrap-server ...` | État d'un group (lag) |

### API Python (confluent-kafka)

| Code | Description |
|------|-------------|
| `Producer({'bootstrap.servers': ...})` | Créer un producer |
| `producer.produce(topic, key, value, callback)` | Envoyer un message |
| `producer.flush()` | Attendre l'envoi complet |
| `Consumer({'group.id': ..., 'auto.offset.reset': 'earliest'})` | Créer un consumer |
| `consumer.subscribe(['topic'])` | S'abonner |
| `msg = consumer.poll(1.0)` | Lire (timeout 1s) |
| `consumer.close()` | Fermer proprement |

### Concepts clés à retenir

| Terme | Définition express |
|-------|--------------------|
| **Topic** | Flux nommé de messages |
| **Partition** | Unité de parallélisme, ordre garanti à l'intérieur |
| **Offset** | Position immuable dans une partition |
| **Consumer group** | Partage des partitions entre consumers |
| **Lag** | Retard = dernier offset produit − offset consommé |
| **Retention** | Durée de conservation des messages |
| **acks=all** | Durabilité maximale côté producer |
| **At-least-once** | Doublons possibles → traitement idempotent |

---

## ❓ Questions de révision

1. **Pourquoi l'ordre des messages n'est-il garanti que dans une partition ?**
   → Les partitions sont indépendantes et traitées en parallèle ; il n'existe pas de séquencement global entre elles.

2. **Que se passe-t-il si un consumer group a plus de consumers que de partitions ?**
   → Les consumers excédentaires restent inactifs (standby) — chaque partition n'est lue que par un seul consumer du group.

3. **Différence entre `earliest` et `latest` pour `auto.offset.reset` ?**
   → `earliest` : relire tout l'historique disponible ; `latest` : ne lire que les nouveaux messages.

4. **Pourquoi utiliser Avro + Schema Registry plutôt que JSON ?**
   → Messages plus compacts, schéma contractuel, évolution contrôlée avec règles de compatibilité.

5. **Qu'est-ce que le consumer lag et pourquoi le surveiller ?**
   → Écart entre production et consommation ; un lag croissant signale un consumer trop lent ou en panne.

6. **Quelle sémantique choisir pour un pipeline financier et comment l'implémenter ?**
   → At-least-once (ou exactly-once avec transactions) + traitement idempotent pour gérer les doublons.

7. **Différence entre STREAM et TABLE dans ksqlDB ?**
   → STREAM = flux d'événements append-only ; TABLE = état courant par clé (changelog).

---

> 📚 **Ressources** :
> - [Repo GitHub du Zoomcamp — Module 6](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/06-streaming)
> - [Documentation Apache Kafka](https://kafka.apache.org/documentation/)
> - [Documentation Confluent (Python client)](https://docs.confluent.io/kafka-clients/python/current/overview.html)
> - [Kafka: The Definitive Guide (O'Reilly)](https://www.confluent.io/resources/kafka-the-definitive-guide/)

---

## ✅ Récapitulatif

Cette fiche couvre l'intégralité du **Module 6 (Streaming)** :

- **Fondamentaux** : batch vs streaming, architecture Kafka (brokers, topics, partitions, offsets)
- **Pratique** : installation Docker (KRaft), CLI, producer/consumer en Python
- **Concepts avancés** : consumer groups, sémantiques de livraison, réplication
- **Écosystème** : Avro + Schema Registry, Kafka Streams/Faust, ksqlDB, Kafka Connect
- **Cas pratique** : pipeline de streaming NYC Taxi complet
- **Aide-mémoire** + **questions de révision**

> 💡 Copiez ce contenu dans `module6-streaming-kafka.md` et poussez-le sur GitHub. Pour pratiquer : lancez le `docker-compose` du cours, envoyez les données taxi dans un topic, et écrivez votre premier consumer !


