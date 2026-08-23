Voici la fiche de révision détaillée pour le Module 5 — **Batch Processing avec Apache Spark** — au format Markdown, prête pour GitHub.




# Module 5 — Batch Processing avec Apache Spark

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision du Module 5 : Traitement par lots (Batch Processing) avec Apache Spark



## 📑 Table des matières

1. [Introduction au Batch Processing](#1-introduction-au-batch-processing)
2. [Introduction à Apache Spark](#2-introduction-à-apache-spark)
3. [Architecture de Spark](#3-architecture-de-spark)
4. [Installation et premiers pas](#4-installation-et-premiers-pas)
5. [Les RDD (Resilient Distributed Datasets)](#5-les-rdd-resilient-distributed-datasets)
6. [DataFrames et Spark SQL](#6-dataframes-et-spark-sql)
7. [Lecture et écriture de données](#7-lecture-et-écriture-de-données)
8. [Les transformations et actions](#8-les-transformations-et-actions)
9. [Les partitions et le partitioning](#9-les-partitions-et-le-partitioning)
10. [Les joins dans Spark](#10-les-joins-dans-spark)
11. [Spark SQL avancé](#11-spark-sql-avancé)
12. [Cas pratique : NYC Taxi Data](#12-cas-pratique--nyc-taxi-data)
13. [Spark dans le cloud (Dataproc / EMR)](#13-spark-dans-le-cloud-dataproc--emr)
14. [Optimisation et bonnes pratiques](#14-optimisation-et-bonnes-pratiques)
15. [Aide-mémoire](#15-aide-mémoire-commandes-et-api)

---

## 1. Introduction au Batch Processing

### 1.1 Batch vs Streaming

| Critère | Batch Processing | Stream Processing |
|---------|------------------|-------------------|
| **Données** | Finies, bornées | Infinies, non bornées |
| **Latence** | Minutes / heures | Millisecondes / secondes |
| **Exemples** | Rapports quotidiens, ETL nocturnes | Alertes temps réel, fraude |
| **Outils** | Spark, SQL, dbt, Hive | Kafka Streams, Flink, Spark Streaming |

### 1.2 Quand utiliser le batch ?

- Traitement de gros volumes de données historiques
- Pipelines ETL planifiés (quotidiens, hebdomadaires)
- Pas besoin de résultats en temps réel
- Préparation des données pour l'analytique et le ML

---

## 2. Introduction à Apache Spark

### 2.1 Qu'est-ce que Spark ?

- Moteur de **traitement distribué** open-source (Apache)
- Conçu pour traiter de **gros volumes de données** en parallèle sur un **cluster de machines**
- API disponibles en **Python (PySpark)**, Scala, Java, R et SQL
- Jusqu'à **100x plus rapide** que Hadoop MapReduce (traitement en mémoire)

### 2.2 L'écosystème Spark

```
┌─────────────────────────────────────────────────────┐
│                  Apache Spark                       │
├──────────────┬──────────────┬───────────┬───────────┤
│  Spark SQL   │ Spark        │  MLlib    │ GraphX    │
│ (DataFrames) │ Streaming    │  (ML)     │ (Graphes) │
├──────────────┴──────────────┴───────────┴───────────┤
│                  Spark Core (RDDs)                  │
├─────────────────────────────────────────────────────┤
│   Standalone │ YARN │ Mesos │ Kubernetes            │
└─────────────────────────────────────────────────────┘
```

### 2.3 Quand utiliser Spark ?

✅ **Utiliser Spark quand :**
- Les données ne tiennent pas sur une seule machine
- Traitement complexe nécessitant plusieurs étapes
- Besoin de parallélisme sur des dizaines/centaines de nœuds

❌ **Éviter Spark quand :**
- Les données tiennent en mémoire d'une seule machine → **pandas** suffit
- Requêtes simples sur un data warehouse → **SQL** (BigQuery, etc.) suffit

---

## 3. Architecture de Spark

### 3.1 Composants principaux

```
                ┌──────────────┐
                │    Driver    │  ← Programme principal (SparkSession)
                │  (Master)    │
                └──────┬───────┘
                       │
              ┌────────┴────────┐
              │ Cluster Manager │  ← Alloue les ressources (YARN, K8s, Standalone)
              └────────┬────────┘
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌───────────┐ ┌───────────┐ ┌───────────┐
   │  Worker   │ │  Worker   │ │  Worker   │
   │ Executor  │ │ Executor  │ │ Executor  │
   │  Tasks    │ │  Tasks    │ │  Tasks    │
   └───────────┘ └───────────┘ └───────────┘
```

- **Driver** : exécute le programme principal, crée la `SparkSession`, planifie les tâches
- **Cluster Manager** : alloue les ressources entre les applications
- **Executors** : processus sur les workers qui exécutent les tâches et stockent les données en mémoire
- **Task** : unité de travail la plus petite, traitée par un executor

### 3.2 Modes d'exécution

| Mode | Description |
|------|-------------|
| `local` | Une seule machine (développement/test) |
| `local[*]` | Une machine, tous les cœurs |
| `standalone` | Cluster Spark natif |
| `yarn` | Cluster Hadoop (YARN) |
| `kubernetes` | Cluster Kubernetes |

---

## 4. Installation et premiers pas

### 4.1 Installation locale (PySpark)

```bash
# Via pip (nécessite Java 8/11/17 installé)
pip install pyspark

# Vérifier Java
java -version
```

### 4.2 Installation complète (avec Spark UI)

```bash
# Télécharger Spark
wget https://archive.apache.org/dist/spark/spark-3.5.0/spark-3.5.0-bin-hadoop3.tgz
tar xzfv spark-3.5.0-bin-hadoop3.tgz

# Variables d'environnement
export SPARK_HOME="${HOME}/spark/spark-3.5.0-bin-hadoop3"
export PATH="${SPARK_HOME}/bin:${PATH}"
export PYTHONPATH="${SPARK_HOME}/python:${SPARK_HOME}/python/lib/py4j-*.zip:${PYTHONPATH}"
```

### 4.3 Création d'une SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("my-app") \
    .getOrCreate()

# Interface web : http://localhost:4040
```

### 4.4 Spark shell et notebooks

```bash
# Shell PySpark interactif
pyspark

# Avec Jupyter
pip install jupyter
jupyter notebook
```

---

## 5. Les RDD (Resilient Distributed Datasets)

### 5.1 Définition

- Structure de données **fondamentale** de Spark (bas niveau)
- Collection **distribuée**, **immuable** et **tolérante aux pannes**
- Les DataFrames sont construits **par-dessus** les RDD

### 5.2 Création d'un RDD

```python
# Depuis une liste Python
rdd = spark.sparkContext.parallelize([1, 2, 3, 4, 5])

# Depuis un fichier texte
rdd = spark.sparkContext.textFile("data/fichier.txt")
```

### 5.3 Exemple classique : map / filter / reduce

```python
# Préparation des données
rows = [
    ("green", "2019-01-01", 100),
    ("yellow", "2019-01-01", 200),
    ("green", "2019-01-02", 150),
]

rdd = spark.sparkContext.parallelize(rows)

# Transformations
result = rdd \
    .filter(lambda row: row[0] == "green") \
    .map(lambda row: (row[1], row[2])) \
    .reduceByKey(lambda a, b: a + b) \
    .collect()

# [('2019-01-01', 100), ('2019-01-02', 150)]
```

### 5.4 RDD vs DataFrame

| RDD | DataFrame |
|-----|-----------|
| Bas niveau | Haut niveau (comme pandas) |
| Pas de schéma | Schéma défini (colonnes typées) |
| Optimisation manuelle | Optimisé par Catalyst (moteur de requêtes) |
| Verbeux | Concis |

> 💡 **En pratique : on utilise presque toujours les DataFrames.** Les RDD servent pour des cas très spécifiques.

---

## 6. DataFrames et Spark SQL

### 6.1 Création d'un DataFrame

```python
# Depuis un fichier Parquet
df = spark.read.parquet("data/trips.parquet")

# Depuis un CSV
df = spark.read \
    .option("header", "true") \
    .csv("data/zones.csv")

# Depuis une liste avec schéma
from pyspark.sql import types

schema = types.StructType([
    types.StructField("id", types.IntegerType(), True),
    types.StructField("name", types.StringType(), True),
])

df = spark.createDataFrame([(1, "Alice"), (2, "Bob")], schema=schema)
```

### 6.2 Inspection des données

```python
df.show(5)              # Affiche 5 lignes
df.printSchema()        # Affiche le schéma
df.count()              # Nombre de lignes
df.columns              # Liste des colonnes
df.describe().show()    # Statistiques descriptives
df.head(3)              # 3 premières lignes (objets Row)
```

### 6.3 Schémas et types de données

```python
from pyspark.sql import types

# Types courants
types.IntegerType()     # Entier
types.LongType()        # Entier long
types.DoubleType()      # Décimal
types.StringType()      # Chaîne de caractères
types.TimestampType()   # Date-heure
types.DateType()        # Date
types.BooleanType()     # Booléen

# Lire le schéma d'un DataFrame
df.schema
```

---

## 7. Lecture et écriture de données

### 7.1 Formats de fichiers

| Format | Usage | Particularités |
|--------|-------|----------------|
| **Parquet** | ✅ Recommandé pour Spark | Colonnaire, compressé, schéma embarqué |
| **CSV** | Échange de données | Pas de types, verbeux, header optionnel |
| **JSON** | Données semi-structurées | Une ligne JSON par ligne |
| **Avro / ORC** | Alternatives colonniaires | Moins courants avec Spark |

### 7.2 Lecture

```python
# Parquet (inférence automatique du schéma)
df = spark.read.parquet("data/yellow_tripdata_*.parquet")

# CSV avec schéma explicite (recommandé)
df = spark.read \
    .option("header", "true") \
    .schema(schema) \
    .csv("data/zones.csv")

# Lire plusieurs fichiers avec un wildcard
df = spark.read.parquet("data/yellow_tripdata_2025-*.parquet")
```

### 7.3 Écriture

```python
# Écrire en Parquet (une partition = un dossier)
df.write.parquet("output/trips/")

# Écraser les données existantes
df.write.mode("overwrite").parquet("output/trips/")

# Partitionner les données en sortie
df.write.partitionBy("pickup_date").parquet("output/trips/")

# Écrire en CSV
df.write.option("header", "true").csv("output/zones/")
```

> ⚠️ Spark écrit **plusieurs fichiers part-\*** (un par partition), pas un seul fichier. C'est normal et voulu !

---

## 8. Les transformations et actions

### 8.1 Concept clé : l'évaluation paresseuse (Lazy Evaluation)

- Les **transformations** ne sont **pas exécutées immédiatement** : Spark construit un plan d'exécution (DAG)
- L'exécution est déclenchée uniquement par une **action**
- Cela permet à Spark d'**optimiser** le plan complet (Catalyst optimizer)

```
Transformation ──► Transformation ──► ACTION ──► Exécution !
   (lazy)              (lazy)         (déclencheur)
```

### 8.2 Transformations principales

```python
from pyspark.sql import functions as F

# Sélectionner des colonnes
df.select("PULocationID", "fare_amount")

# Filtrer
df.filter(df.fare_amount > 10)
df.where("fare_amount > 10")

# Ajouter / modifier une colonne
df = df.withColumn("total", df.fare_amount + df.tip_amount)

# Renommer une colonne
df = df.withColumnRenamed("old_name", "new_name")

# Supprimer des doublons
df.dropDuplicates(["col1", "col2"])

# Trier
df.orderBy(df.fare_amount.desc())
df.sort("fare_amount", ascending=False)

# Fonctions de colonnes
df.withColumn("pickup_date", F.to_date(df.tpep_pickup_datetime))
df.withColumn("revenue_zone", F.concat(F.lit("Zone_"), df.PULocationID))
```

### 8.3 Actions principales

```python
df.show()           # Afficher
df.count()          # Compter
df.collect()        # ⚠️ Ramène TOUT dans le driver (mémoire !)
df.take(5)          # Prendre 5 lignes
df.head()           # Première ligne
df.write.parquet()  # Écrire
df.reduce(...)      # Réduire (RDD)
```

> ⚠️ **Ne jamais utiliser `collect()`** sur de gros DataFrames : cela ramène toutes les données sur le driver → risque de crash mémoire (OOM).

### 8.4 Transformations narrow vs wide

| Type | Description | Exemples |
|------|-------------|----------|
| **Narrow** | Chaque partition d'entrée contribue à **une seule** partition de sortie (pas de shuffle) | `select`, `filter`, `withColumn` |
| **Wide** | Nécessite un **shuffle** (redistribution des données sur le réseau) | `groupBy`, `join`, `orderBy`, `distinct` |

> 💡 Les **shuffles** sont coûteux (réseau + disque). Les minimiser est la clé de la performance.

---

## 9. Les partitions et le partitioning

### 9.1 Qu'est-ce qu'une partition ?

- Unité de **distribution** des données entre les nœuds
- Chaque partition est traitée par **une tâche** sur **un executor**
- Par défaut : `spark.sql.shuffle.partitions = 200`

### 9.2 Contrôler les partitions

```python
# Nombre de partitions d'un DataFrame
df.rdd.getNumPartitions()

# Repartitionner (déclenche un shuffle)
df = df.repartition(24)

# Repartitionner par colonne (regroupe les mêmes clés)
df = df.repartition("PULocationID")

# Réduire les partitions SANS shuffle complet
df = df.coalesce(4)

# Modifier le nombre de partitions par défaut pour les shuffles
spark.conf.set("spark.sql.shuffle.partitions", "50")
```

### 9.3 Bonnes pratiques

- Viser des partitions de **100 Mo à 1 Go** chacune
- Trop de partitions → overhead de gestion des tâches
- Trop peu de partitions → sous-utilisation du cluster, risque OOM
- `repartition()` avant une écriture pour contrôler le nombre de fichiers de sortie

---

## 10. Les joins dans Spark

### 10.1 Types de joins

```python
# Join simple
df_result = df_trips.join(df_zones, df_trips.PULocationID == df_zones.LocationID)

# Types de join
df_trips.join(df_zones, on="LocationID", how="inner")       # inner (défaut)
df_trips.join(df_zones, on="LocationID", how="left")        # left
df_trips.join(df_zones, on="LocationID", how="right")       # right
df_trips.join(df_zones, on="LocationID", how="outer")       # full outer
df_trips.join(df_zones, on="LocationID", how="left_semi")   # semi
df_trips.join(df_zones, on="LocationID", how="left_anti")   # anti
```

### 10.2 Stratégies de join

| Stratégie | Quand | Comment ça marche |
|-----------|-------|-------------------|
| **Broadcast Join** | Une table est **petite** (< 10 Mo par défaut) | La petite table est **copiée sur tous les executors** → pas de shuffle ! |
| **Sort-Merge Join** | Deux **grosses** tables | Shuffle des deux tables puis fusion (défaut) |

```python
# Forcer un broadcast join
from pyspark.sql.functions import broadcast

df_result = df_trips.join(
    broadcast(df_zones),          # ← la petite table
    df_trips.PULocationID == df_zones.LocationID
)

# Configurer le seuil automatique (défaut : 10 Mo)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "50m")
```

> 💡 **Règle d'or** : toujours broadcaster la petite table (ex : `dim_zones` avec 265 lignes) quand on la joint à une grosse table de faits.

---

## 11. Spark SQL avancé

### 11.1 Enregistrer des tables temporaires

```python
df_trips.createOrReplaceTempView("trips")
df_zones.createOrReplaceTempView("zones")
```

### 11.2 Requêtes SQL

```python
df_result = spark.sql("""
    SELECT 
        z.Zone,
        COUNT(*) AS trip_count,
        SUM(t.total_amount) AS total_revenue
    FROM trips t
    JOIN zones z ON t.PULocationID = z.LocationID
    GROUP BY z.Zone
    ORDER BY total_revenue DESC
""")
df_result.show()
```

### 11.3 Group By et agrégations

```python
from pyspark.sql import functions as F

df.groupBy("PULocationID") \
  .agg(
      F.count("*").alias("trip_count"),
      F.sum("total_amount").alias("revenue"),
      F.avg("trip_distance").alias("avg_distance"),
      F.max("fare_amount").alias("max_fare"),
  ) \
  .orderBy(F.desc("revenue")) \
  .show()
```

### 11.4 Fonctions utiles

```python
from pyspark.sql import functions as F

# Dates
F.to_date(col)                    # Extraire la date
F.to_timestamp(col)               # Convertir en timestamp
F.date_trunc("month", col)        # Tronquer au mois
F.year(col), F.month(col)         # Extraire année/mois
F.datediff(col1, col2)            # Différence en jours

# Chaînes
F.concat(col1, col2)              # Concaténer
F.upper(col), F.lower(col)        # Casse
F.substring(col, 1, 3)            # Sous-chaîne

# Divers
F.lit("valeur")                   # Colonne constante
F.when(cond, val).otherwise(val)  # CASE WHEN
F.coalesce(col1, col2)            # Première valeur non nulle
F.isnull(col), F.isnotnull(col)   # Test de nullité
```

### 11.5 Fonctions définies par l'utilisateur (UDF)

```python
from pyspark.sql import functions as F
from pyspark.sql import types

def categorize_fare(fare):
    if fare < 10:
        return "cheap"
    elif fare < 30:
        return "medium"
    return "expensive"

# Enregistrer l'UDF
categorize_udf = F.udf(categorize_fare, types.StringType())

df = df.withColumn("fare_category", categorize_udf(df.fare_amount))
```

> ⚠️ Les UDF Python sont **lentes** (sérialisation JVM ↔ Python). Préférer les fonctions natives Spark (`F.when`, etc.) quand c'est possible.

---

## 12. Cas pratique : NYC Taxi Data

### 12.1 Pipeline typique du cours

```
CSV (brut) ──► Spark : nettoyage + typage ──► Parquet (Data Lake)
                                                    │
                                                    ▼
                                        Spark : agrégations + joins
                                                    │
                                                    ▼
                                    Résultats ──► BigQuery / rapport
```

### 12.2 Convertir CSV → Parquet

```python
from pyspark.sql import SparkSession, types

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("nyc-taxi") \
    .getOrCreate()

# Schéma explicite (évite l'inférence coûteuse et les erreurs de type)
schema = types.StructType([
    types.StructField("VendorID", types.IntegerType(), True),
    types.StructField("tpep_pickup_datetime", types.TimestampType(), True),
    types.StructField("tpep_dropoff_datetime", types.TimestampType(), True),
    types.StructField("passenger_count", types.IntegerType(), True),
    types.StructField("trip_distance", types.DoubleType(), True),
    types.StructField("PULocationID", types.IntegerType(), True),
    types.StructField("DOLocationID", types.IntegerType(), True),
    types.StructField("payment_type", types.IntegerType(), True),
    types.StructField("fare_amount", types.DoubleType(), True),
    types.StructField("tip_amount", types.DoubleType(), True),
    types.StructField("total_amount", types.DoubleType(), True),
])

df = spark.read \
    .option("header", "true") \
    .schema(schema) \
    .csv("yellow_tripdata_2025-01.csv")

df = df.repartition(24)
df.write.parquet("data/pq/yellow/2025/01/")
```

### 12.3 Analyse type du cours

```python
df = spark.read.parquet("data/pq/yellow/*/*/")
df_zones = spark.read.option("header", "true").csv("taxi_zone_lookup.csv")

df.registerTempTable("yellow")
df_zones.registerTempTable("zones")

# Zones les plus fréquentes en pickup
spark.sql("""
    SELECT z.Zone, COUNT(1) AS count
    FROM yellow y
    JOIN zones z ON y.PULocationID = z.LocationID
    GROUP BY z.Zone
    ORDER BY count DESC
    LIMIT 10
""").show()
```

---

## 13. Spark dans le cloud (Dataproc / EMR)

### 13.1 Google Cloud Dataproc

```bash
# Créer un cluster Dataproc
gcloud dataproc clusters create my-cluster \
    --region=europe-west1 \
    --master-machine-type=n1-standard-4 \
    --num-workers=2 \
    --worker-machine-type=n1-standard-4

# Soumettre un job Spark
gcloud dataproc jobs submit pyspark \
    --cluster=my-cluster \
    --region=europe-west1 \
    gs://my-bucket/scripts/job.py \
    -- --input=gs://my-bucket/data/ --output=gs://my-bucket/output/

# Supprimer le cluster (important pour les coûts !)
gcloud dataproc clusters delete my-cluster --region=europe-west1
```

### 13.2 Connecter Spark à GCS

```bash
# Installer le connecteur GCS et le configurer
spark = SparkSession.builder \
    .appName("gcs") \
    .config("spark.jars", "gcs-connector-hadoop3-latest.jar") \
    .getOrCreate()

# Lire directement depuis GCS
df = spark.read.parquet("gs://my-bucket/data/trips/")
```

### 13.3 Connecter Spark à BigQuery

```python
df = spark.read \
    .format("bigquery") \
    .option("table", "project.dataset.table") \
    .load()

df.write \
    .format("bigquery") \
    .option("table", "project.dataset.output_table") \
    .mode("overwrite") \
    .save()
```

### 13.4 Script de job soumissible

```python
# job.py — structure d'un job Spark production
import argparse
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

parser = argparse.ArgumentParser()
parser.add_argument("--input", required=True)
parser.add_argument("--output", required=True)
args = parser.parse_args()

spark = SparkSession.builder.appName("taxi-job").getOrCreate()

df = spark.read.parquet(args.input)

df_result = df.groupBy("PULocationID") \
    .agg(F.count("*").alias("trips"))

df_result.write.mode("overwrite").parquet(args.output)

spark.stop()
```

```bash
# Soumission locale
spark-submit \
    --master="local[*]" \
    job.py \
    --input=data/pq/yellow/ \
    --output=output/report/
```

---

## 14. Optimisation et bonnes pratiques

### 14.1 Checklist d'optimisation

- ✅ Utiliser **Parquet** plutôt que CSV (colonnaire, prédicats pushdown)
- ✅ **Broadcaster** les petites tables dans les joins
- ✅ **Filtrer tôt** : réduire les données dès le début du pipeline
- ✅ Sélectionner uniquement les **colonnes nécessaires**
- ✅ Ajuster le **nombre de partitions** (ni trop, ni trop peu)
- ✅ Utiliser les **fonctions natives** plutôt que des UDF Python
- ✅ `persist()` / `cache()` un DataFrame réutilisé plusieurs fois

```python
# Mettre en cache un DataFrame réutilisé
df.cache()
df.count()  # Matérialise le cache

# Libérer la mémoire
df.unpersist()
```

### 14.2 Lire la Spark UI (localhost:4040)

| Onglet | Contenu |
|--------|---------|
| **Jobs** | Liste des jobs déclenchés par les actions |
| **Stages** | Découpage en stages (séparés par les shuffles) |
| **Tasks** | Tâches individuelles et leur durée |
| **Storage** | DataFrames en cache |
| **SQL** | Plans d'exécution des requêtes (DAG visuel) |
| **Executors** | Mémoire, CPU et état des executors |

### 14.3 Problèmes courants

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Job très lent | Data skew (clés déséquilibrées) | Salting des clés, repartition |
| OOM sur le driver | `collect()` sur un gros DF | `take(n)`, écrire sur disque |
| OOM sur executors | Partitions trop grosses | Augmenter `repartition(n)` |
| Trop de petits fichiers | Trop de partitions en sortie | `coalesce(n)` avant `write` |
| Shuffle excessif | Joins/groupBy mal placés | Broadcast, filtrer avant |

---

## 15. Aide-mémoire : commandes et API

### SparkSession

```python
SparkSession.builder.master("local[*]").appName("app").getOrCreate()
spark.stop()
```

### Lecture / Écriture

| Opération | Code |
|-----------|------|
| Lire Parquet | `spark.read.parquet("path")` |
| Lire CSV | `spark.read.option("header","true").schema(s).csv("path")` |
| Écrire Parquet | `df.write.mode("overwrite").parquet("path")` |
| Partitionner | `df.write.partitionBy("col").parquet("path")` |

### DataFrame

| Opération | Code |
|-----------|------|
| Sélection | `df.select("c1", "c2")` |
| Filtrer | `df.filter(df.c1 > 10)` |
| Nouvelle colonne | `df.withColumn("c", F.expr)` |
| Grouper | `df.groupBy("c").agg(F.count("*"))` |
| Joindre | `df1.join(df2, df1.id == df2.id, "left")` |
| Broadcast | `df1.join(F.broadcast(df2), "id")` |
| Repartitionner | `df.repartition(24)` |
| Coalesce | `df.coalesce(4)` |
| SQL | `df.createOrReplaceTempView("t")` + `spark.sql("...")` |

### Ligne de commande

```bash
pyspark                                           # Shell interactif
spark-submit --master local[*] job.py             # Soumettre un job local
gcloud dataproc jobs submit pyspark --cluster=c job.py   # Dataproc
```

---

## ❓ Questions de révision

1. **Quelle est la différence entre transformation et action ?**
   → Les transformations sont *lazy* (plan logique), les actions déclenchent l'exécution.

2. **Qu'est-ce qu'un shuffle et pourquoi l'éviter ?**
   → Redistribution des données sur le réseau (wide transformations). Coûteux : réseau + disque.

3. **Quand utiliser un broadcast join ?**
   → Quand une des tables est petite (< seuil configurable, 10 Mo par défaut).

4. **Pourquoi Parquet est-il préféré à CSV avec Spark ?**
   → Format colonnaire, compressé, schéma embarqué, predicate pushdown.

5. **Différence entre `repartition()` et `coalesce()` ?**
   → `repartition` fait un shuffle complet (peut augmenter/réduire), `coalesce` fusionne sans shuffle complet (réduit seulement).

6. **Quel est le rôle du driver dans Spark ?**
   → Orchestre l'exécution : crée le plan, distribue les tâches aux executors, collecte les résultats.

---

> 📚 **Ressources** :
> - [Repo GitHub du Zoomcamp — Module 5](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/05-batch)
> - [Documentation officielle Spark](https://spark.apache.org/docs/latest/)
> - [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)
````



## ✅ Récapitulatif

La fiche couvre l'intégralité du **Module 5 (Batch Processing)** du Zoomcamp :

- **Concepts fondamentaux** : batch vs streaming, architecture Driver/Executors
- **RDD et DataFrames** : transformations, actions, lazy evaluation
- **Manipulation de données** : CSV → Parquet, schémas, partitions
- **Optimisation** : broadcast joins, shuffles, caching, Spark UI
- **Cas pratique** : pipeline NYC Taxi Data complet
- **Cloud** : Dataproc, GCS, BigQuery, `spark-submit`
- **Aide-mémoire** + **questions de révision** pour tester vos connaissances

> 💡 **Conseil** : copiez ce contenu dans un fichier `module5-batch-spark.md` et poussez-le sur GitHub. Pour aller plus loin, entraînez-vous avec le notebook officiel du cours et l'homework du Module 5 !

Souhaitez-vous que je fasse de même pour un autre module (ex. **Module 6 — Streaming avec Kafka**) ? 🚀
