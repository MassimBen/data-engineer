Voici la fiche de révision pour le **Module 3 — Data Warehouse et BigQuery**, au format Markdown prêt pour GitHub.

---
# Module 3 — Data Warehouse avec Google BigQuery

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision du Module 3 : Data Warehouses et Google BigQuery

---

## 📑 Table des matières

1. [Introduction aux Data Warehouses](#1-introduction-aux-data-warehouses)
2. [OLTP vs OLAP](#2-oltp-vs-olap)
3. [Introduction à BigQuery](#3-introduction-à-bigquery)
4. [Architecture de BigQuery](#4-architecture-de-bigquery)
5. [Stockage : format colonnaire et Capacitor](#5-stockage--format-colonnaire-et-capacitor)
6. [Chargement de données dans BigQuery](#6-chargement-de-données-dans-bigquery)
7. [Tables externes vs tables natives](#7-tables-externes-vs-tables-natives)
8. [Partitioning](#8-partitioning)
9. [Clustering](#9-clustering)
10. [Optimisation des requêtes et des coûts](#10-optimisation-des-requêtes-et-des-coûts)
11. [ML avec BigQuery (BQML)](#11-ml-avec-bigquery-bqml)
12. [Cas pratique : NYC Taxi Data](#12-cas-pratique--nyc-taxi-data)
13. [Bonnes pratiques](#13-bonnes-pratiques)
14. [Aide-mémoire](#14-aide-mémoire-commandes-et-sql)

---

## 1. Introduction aux Data Warehouses

### 1.1 Qu'est-ce qu'un Data Warehouse ?

- Système de stockage **centralisé** optimisé pour l'**analyse** (pas pour les transactions)
- Contient des données **historiques**, provenant de **sources multiples**
- Sert à la **BI**, aux **rapports**, à la **data science**
- Souvent au cœur du pattern **ELT** : Extract → Load → Transform (transformation DANS le warehouse, ex. avec dbt — cf. Module 4)

### 1.2 Data Warehouse vs Data Lake vs Lakehouse

| | Data Warehouse | Data Lake | Lakehouse |
|---|---|---|---|
| **Données** | Structurées | Tous formats (brutes) | Les deux |
| **Schéma** | Schema-on-write | Schema-on-read | Hybride |
| **Usage** | BI, rapports SQL | Data science, ML | Unifié |
| **Exemples** | BigQuery, Redshift, Snowflake | S3, GCS + Spark | Databricks, BigLake |

### 1.3 Pourquoi un Data Warehouse ?

- ✅ Requêtes analytiques **rapides** sur de gros volumes
- ✅ **Découplage** des systèmes transactionnels (ne pas ralentir la prod)
- ✅ **Historisation** des données
- ✅ Source unique de vérité pour l'**analyse**

---

## 2. OLTP vs OLAP

| Critère | OLTP (transactionnel) | OLAP (analytique) |
|---------|----------------------|-------------------|
| **But** | Opérations quotidiennes | Analyse, rapports |
| **Requêtes** | Petites, fréquentes (INSERT, UPDATE) | Grosses, complexes (agrégations) |
| **Volume par requête** | Quelques lignes | Millions de lignes |
| **Stockage** | Orienté **ligne** | Orienté **colonne** |
| **Exemples** | PostgreSQL, MySQL | BigQuery, Redshift, Snowflake |
| **Schéma** | Normalisé (3NF) | Dénormalisé (star schema) |

> 💡 **Analogie** : OLTP = la caisse du supermarché (rapide, une transaction à la fois). OLAP = le bilan comptable annuel (analyse de toutes les ventes).

### Le stockage orienté colonnes

```
Données :                          Stockage LIGNE :        Stockage COLONNE :
id | ville  | prix                 1,Paris,100             1,2,3
1  | Paris  | 100                  2,Lyon,200              Paris,Lyon,Paris
2  | Lyon   | 200                  3,Paris,150             100,200,150
3  | Paris  | 150
```

**Avantages du format colonne :**
- On ne lit que les **colonnes nécessaires** à la requête
- **Compression** très efficace (valeurs similaires côte à côte)
- **Agrégations** ultra-rapides (SUM, AVG sur une colonne)

---

## 3. Introduction à BigQuery

### 3.1 Qu'est-ce que BigQuery ?

- Data Warehouse **serverless** de Google Cloud
- **Pas de serveur à gérer** : pas de cluster à provisionner
- Scalabilité quasi-infinie : petaoctets de données
- Tarification à l'usage

### 3.2 Modèle de tarification 💰

| Modèle | Description | Prix indicatif |
|--------|-------------|----------------|
| **On-demand** | Payé par **quantité de données scannées** | ~5 $ / To scanné |
| **Flat-rate / Editions** | Capacité réservée (slots) | Fixe mensuel/horaire |
| **Stockage** | Données stockées | ~0,02 $ / Go / mois (actif) |

> ⚠️ **Règle d'or** : le coût dépend des **données scannées**, pas du résultat ! Un `SELECT *` sur une table de 1 To coûte ~5 $.

**Gratuit :**
- **1 To de requêtes / mois** (on-demand)
- **10 Go de stockage / mois**
- Chargement de données, export, métadonnées

### 3.3 Interface et outils

- **Console web** : console.cloud.google.com/bigquery
- **CLI** : `bq`
- **Client Python** : `google-cloud-bigquery`
- Intégrations : dbt, Airflow, Looker Studio, pandas...

```bash
# Installation du client Python
pip install google-cloud-bigquery pandas pyarrow
```

```python
from google.cloud import bigquery

client = bigquery.Client(project='mon-projet')

query = """
SELECT COUNT(*) AS total
FROM `nytaxi.yellow_tripdata`
"""
df = client.query(query).to_dataframe()
print(df)
```

---

## 4. Architecture de BigQuery

### 4.1 Architecture séparée : stockage ≠ compute

```
                 ┌─────────────────┐
   Requête SQL ─►│  Dremel (query) │
                 │  Moteur distribué│
                 └────────┬────────┘
                          │  Petabit network (Jupiter)
                 ┌────────▼────────┐
                 │ Colossus        │
                 │ Stockage        │
                 │ colonnaire      │
                 └─────────────────┘
```

- **Dremel** : moteur d'exécution des requêtes (compute)
- **Colossus** : système de fichiers distribué (stockage)
- **Jupiter** : réseau interne ultra-rapide entre les deux
- ➡️ **Stockage et compute scalent indépendamment** (contrairement à Redshift classique ou aux BDD traditionnelles)

### 4.2 Exécution d'une requête (arbre d'exécution)

```
        Root server (reçoit la requête SQL)
              │
     ┌────────┼────────┐
   Mixer   Mixer    Mixer        ← agrègent les résultats intermédiaires
     │       │        │
  Leaf    Leaf ...  Leaf         ← lisent le stockage en parallèle (slots)
```

- **Slot** = unité de calcul (CPU + RAM) — ~2000 slots par requête on-demand
- Shuffle en mémoire entre étapes
- Résultats en **cache** 24h (requête identique → gratuit, instantané !)

### 4.3 Organisation des ressources

```
Projet GCP
 └── Dataset (≈ schéma / base)
      ├── Table
      ├── Vue (view)
      └── Fonction (UDF)
```

Nom complet d'une table : `projet.dataset.table`

---

## 5. Stockage : format colonnaire et Capacitor

- BigQuery stocke les données en **Capacitor** : format **colonnaire** propriétaire
- Les données sont découpées en **blocs** compressés
- Seules les **colonnes demandées** sont lues → `SELECT col` coûte moins cher que `SELECT *`
- **Réplication automatique**, haute durabilité (géré par Colossus)

> 💡 Parquet et Avro (formats open-source) reposent sur les mêmes principes — BigQuery les lit nativement à l'import.

---

## 6. Chargement de données dans BigQuery

### 6.1 Méthodes de chargement

| Méthode | Description | Coût |
|---------|-------------|------|
| **Batch load** (CSV, JSON, Avro, Parquet) | Depuis GCS ou upload local | Gratuit |
| **Streaming insert** | Ligne par ligne via API | Payant (~0,01 $/200 Mo) |
| **Queries** (`CREATE TABLE AS SELECT`) | Depuis une autre table | Coût de la requête |
| **Data Transfer Service** | Connecteurs SaaS planifiés | Variable |

### 6.2 Formats recommandés

| Format | Orienté | Recommandé ? |
|--------|---------|--------------|
| **Parquet** | Colonne | ✅ Idéal (schéma, compression, rapide) |
| **Avro** | Ligne | ✅ Bon (schéma embarqué) |
| **CSV/JSON** | Ligne | ⚠️ OK mais plus lent, types inférés |

### 6.3 Chargement avec le CLI `bq`

```bash
# Créer un dataset
bq mk --dataset mon-projet:nytaxi

# Charger un fichier Parquet depuis GCS
bq load \
  --source_format=PARQUET \
  --autodetect \
  nytaxi.yellow_tripdata \
  gs://mon-bucket/yellow_tripdata_*.parquet
```

### 6.4 Chargement avec Python

```python
from google.cloud import bigquery

client = bigquery.Client()

job_config = bigquery.LoadJobConfig(
    source_format=bigquery.SourceFormat.PARQUET,
    autodetect=True,
    write_disposition="WRITE_TRUNCATE"   # ou WRITE_APPEND
)

uri = "gs://mon-bucket/yellow_tripdata.parquet"
table_id = "mon-projet.nytaxi.yellow_tripdata"

load_job = client.load_table_from_uri(uri, table_id, job_config=job_config)
load_job.result()   # attendre la fin

table = client.get_table(table_id)
print(f"{table.num_rows:,} lignes chargées")
```

### 6.5 Création de table par requête

```sql
-- CTAS : Create Table As Select
CREATE OR REPLACE TABLE nytaxi.yellow_clean AS
SELECT *
FROM nytaxi.yellow_tripdata
WHERE fare_amount > 0;
```

---

## 7. Tables externes vs tables natives

### 7.1 Comparaison

| Critère | Table externe | Table native (interne) |
|---------|---------------|------------------------|
| **Données** | Restent dans GCS | Copiées dans le stockage BigQuery |
| **Performance** | Plus lente | Plus rapide |
| **Partitioning/Clustering** | Limité (Hive-style) | ✅ Complet |
| **Coût stockage BQ** | 0 | Payé |
| **Cas d'usage** | Données temporaires, exploration | Analyse récurrente, production |

### 7.2 Créer une table externe

```sql
CREATE OR REPLACE EXTERNAL TABLE nytaxi.yellow_external
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://mon-bucket/yellow_tripdata_*.parquet']
);

-- On peut requêter directement les fichiers GCS !
SELECT * FROM nytaxi.yellow_external LIMIT 10;
```

### 7.3 Pattern recommandé du cours

```
GCS (fichiers Parquet)
   │
   ├─► Table externe (exploration, vérification)
   │
   └─► CREATE TABLE native AS SELECT ... FROM table_externe
        → matérialisation dans BigQuery pour les perfs
```

---

## 8. Partitioning

### 8.1 Principe

- Découper une table en **segments** selon une colonne (souvent une date)
- Les requêtes avec filtre sur cette colonne ne scannent que les **partitions pertinentes**
- ➡️ **Moins de données scannées = requêtes plus rapides ET moins chères**

### 8.2 Types de partitioning

| Type | Colonne | Granularité |
|------|---------|-------------|
| **Time-unit column** | `DATE`, `TIMESTAMP`, `DATETIME` | jour, mois, année |
| **Ingestion time** | `_PARTITIONTIME` (auto) | heure, jour, mois, année |
| **Integer-range** | `INT64` | plages définies (ex. 0-99, 100-199) |

### 8.3 Création d'une table partitionnée

```sql
-- Partitionnement par colonne de date
CREATE OR REPLACE TABLE nytaxi.yellow_partitioned
PARTITION BY DATE(tpep_pickup_datetime)
AS SELECT * FROM nytaxi.yellow_tripdata;

-- Partitionnement par plage d'entiers
CREATE OR REPLACE TABLE nytaxi.by_zone
PARTITION BY RANGE_BUCKET(PULocationID, GENERATE_ARRAY(0, 4000, 10))
AS SELECT * FROM nytaxi.yellow_tripdata;
```

### 8.4 Effet mesurable

```sql
-- ❌ Table non partitionnée : scanne TOUTE la table
SELECT COUNT(*) FROM nytaxi.yellow_tripdata
WHERE DATE(tpep_pickup_datetime) = '2024-01-15';
-- → ex. 3,5 Go scannés

-- ✅ Table partitionnée : scanne UNE seule partition
SELECT COUNT(*) FROM nytaxi.yellow_partitioned
WHERE DATE(tpep_pickup_datetime) = '2024-01-15';
-- → ex. 50 Mo scannés (98% de moins !)
```

### 8.5 Vérifier les partitions

```sql
-- Lister les partitions d'une table
SELECT table_name, partition_id, total_rows
FROM `nytaxi.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'yellow_partitioned';
```

> ⚠️ **Limite** : 4000 partitions max par table, et on ne peut partitionner que sur **une seule** colonne.

---

## 9. Clustering

### 9.1 Principe

- **Trier** les données au sein des partitions selon 1 à 4 colonnes
- BigQuery lit uniquement les **blocs** susceptibles de contenir les valeurs filtrées
- Complémentaire du partitioning !

### 9.2 Partitioning vs Clustering

| Critère | Partitioning | Clustering |
|---------|--------------|------------|
| **Base** | 1 colonne (date/int) | Jusqu'à 4 colonnes |
| **Granularité** | Segments grossiers | Tri fin dans les blocs |
| **Coût requête connu à l'avance** | ✅ Oui (pruning sec) | ❌ Non (pruning à l'exécution) |
| **Cardinalité de la colonne** | Faible/moyenne | ✅ Haute cardinalité OK |
| **Cas d'usage** | Filtres de dates fréquents | Filtres sur ID, tags multiples |

### 9.3 Quand choisir quoi ?

```
Le filtre est sur une DATE ?  ──► Partitioning
Colonne à HAUTE cardinalité (ex. ride_id) ? ──► Clustering
Table < 1 Go ? ──► Ni l'un ni l'autre (overhead inutile)
Filtres sur PLUSIEURS colonnes variées ? ──► Clustering
```

### 9.4 Création avec clustering

```sql
CREATE OR REPLACE TABLE nytaxi.yellow_optimized
PARTITION BY DATE(tpep_pickup_datetime)
CLUSTER BY VendorID
AS SELECT * FROM nytaxi.yellow_tripdata;

-- ✅ Requête qui bénéficie des deux optimisations
SELECT *
FROM nytaxi.yellow_optimized
WHERE DATE(tpep_pickup_datetime) BETWEEN '2024-01-01' AND '2024-01-31'
  AND VendorID = 2;
-- → pruning par partition PUIS par blocs de cluster
```

### 9.5 Propriétés du clustering

- Le clustering est **automatiquement maintenu** (reclustering en arrière-plan, gratuit)
- Ajout possible sur une table existante :
```sql
ALTER TABLE nytaxi.yellow_tripdata
SET OPTIONS (clustering_columns = (VendorID, PULocationID));
```

---

## 10. Optimisation des requêtes et des coûts

### 10.1 Les règles d'or

```sql
-- ❌ Éviter SELECT * : scanne toutes les colonnes
SELECT * FROM nytaxi.yellow_tripdata;

-- ✅ Sélectionner uniquement les colonnes nécessaires
SELECT VendorID, total_amount
FROM nytaxi.yellow_tripdata;
```

| ❌ À éviter | ✅ À faire |
|------------|-----------|
| `SELECT *` | Sélectionner les colonnes utiles |
| Pas de filtre sur la partition | Toujours filtrer sur la colonne de partition |
| `ORDER BY` géant sans LIMIT | Limiter les résultats |
| Fonctions sur la colonne de partition dans le WHERE | Filtres directs (`WHERE date_col = ...`) |
| Croiser de grosses tables | Filtrer AVANT les JOIN |
| Requêter sans estimer | **Dry run** avant exécution |

### 10.2 Vérifier le coût AVANT d'exécuter

```bash
# Dry run : estime les octets scannés sans exécuter
bq query --dry_run --use_legacy_sql=false \
'SELECT * FROM nytaxi.yellow_tripdata WHERE VendorID = 1'
```

Dans la console : le **validateur de requête** (en haut à droite) affiche les octets estimés.

### 10.3 Définir un quota de sécurité

```python
job_config = bigquery.QueryJobConfig(
    maximum_bytes_billed=1_000_000_000   # échoue si > 1 Go scanné
)
client.query(sql, job_config=job_config)
```

### 10.4 Autres optimisations

- **Cache** : requête identique < 24h → résultat gratuit depuis le cache
- **Aperçu (Preview)** dans la console : gratuit, ne lance pas de requête
- **Vues matérialisées** : pré-calcul automatiquement rafraîchi
- `LIMIT` **ne réduit PAS** le coût (scans identiques) !

---

## 11. ML avec BigQuery (BQML)

### 11.1 Principe

- Créer et utiliser des modèles de **machine learning en SQL**, directement dans BigQuery
- Pas d'export de données, pas d'infrastructure ML

### 11.2 Étapes d'un projet BQML

```sql
-- 1️⃣ CRÉER le modèle
CREATE OR REPLACE MODEL nytaxi.tip_model
OPTIONS(
  model_type = 'linear_reg',
  input_label_cols = ['tip_amount'],
  data_split_method = 'auto_split'
) AS
SELECT
  passenger_count,
  trip_distance,
  fare_amount,
  tip_amount
FROM nytaxi.yellow_tripdata
WHERE tip_amount IS NOT NULL;

-- 2️⃣ ÉVALUER le modèle
SELECT * FROM ML.EVALUATE(MODEL nytaxi.tip_model);
-- → mean_absolute_error, r2_score, etc.

-- 3️⃣ PRÉDIRE
SELECT * FROM ML.PREDICT(MODEL nytaxi.tip_model, (
  SELECT passenger_count, trip_distance, fare_amount
  FROM nytaxi.yellow_tripdata
  LIMIT 100
));
```

### 11.3 Types de modèles supportés

| `model_type` | Usage |
|--------------|-------|
| `linear_reg` | Régression (prédire une valeur) |
| `logistic_reg` | Classification |
| `kmeans` | Clustering (non supervisé) |
| `boosted_tree_classifier/regressor` | XGBoost |
| `dnn_classifier/regressor` | Deep learning |
| `ARIMA_PLUS` | Séries temporelles |
| `tensorflow` | Modèles TensorFlow importés |

### 11.4 Feature engineering avec BQML

```sql
OPTIONS(
  model_type = 'linear_reg',
  input_label_cols = ['tip_amount'],
  -- Transformation automatique des features
  TRANSFORM(
    passenger_count,
    trip_distance,
    ML.BUCKETIZE(trip_distance, [1, 5, 10]) AS distance_bucket
  )
)
```

---

## 12. Cas pratique : NYC Taxi Data

### 12.1 Pipeline complet du cours

```
CSV/Parquet (source web)
   │
   ▼  (upload)
GCS bucket (Data Lake)
   │
   ▼  bq load / table externe
Table externe BigQuery
   │
   ▼  CREATE TABLE ... AS SELECT
Table native
   │
   ▼  PARTITION BY + CLUSTER BY
Table optimisée
   │
   ▼
Analyses SQL / BQML / Looker Studio
```

### 12.2 Requêtes types du cours

```sql
-- Comparer le coût : table normale vs partitionnée+clusterisée

-- Table brute : ~scan complet
SELECT COUNT(DISTINCT PULocationID)
FROM nytaxi.yellow_tripdata
WHERE DATE(tpep_pickup_datetime) BETWEEN '2024-01-01' AND '2024-01-31';

-- Table optimisée : pruning partition + cluster
SELECT COUNT(DISTINCT PULocationID)
FROM nytaxi.yellow_optimized
WHERE DATE(tpep_pickup_datetime) BETWEEN '2024-01-01' AND '2024-01-31';
```

### 12.3 Inspection des métadonnées

```sql
-- Taille des tables d'un dataset
SELECT table_id, size_bytes / POW(10, 9) AS size_gb, row_count
FROM nytaxi.__TABLES__;
```

---

## 13. Bonnes pratiques

### Checklist production ✅

- [ ] Charger en **Parquet/Avro** plutôt qu'en CSV
- [ ] **Partitionner** les grosses tables par date
- [ ] **Clusterer** sur les colonnes de filtre fréquentes
- [ ] Toujours **filtrer sur la colonne de partition**
- [ ] Éviter `SELECT *` en production
- [ ] **Dry run** avant les grosses requêtes
- [ ] Définir `maximum_bytes_billed` / des quotas
- [ ] Utiliser les **vues matérialisées** pour les rapports récurrents
- [ ] Table **externe** pour l'exploration, **native** pour la production
- [ ] Monitorer les coûts via `INFORMATION_SCHEMA.JOBS_BY_*`

### Audit des requêtes

```sql
-- Historique des jobs et coûts
SELECT
  job_id,
  user_email,
  total_bytes_processed / POW(10, 12) AS tb_processed,
  total_bytes_processed / POW(10, 12) * 5 AS cost_usd
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY total_bytes_processed DESC;
```

---

## 14. Aide-mémoire : commandes et SQL

### CLI `bq`

| Commande | Description |
|----------|-------------|
| `bq mk --dataset projet:dataset` | Créer un dataset |
| `bq load --source_format=PARQUET dataset.table gs://bucket/*.parquet` | Charger des données |
| `bq query --use_legacy_sql=false 'SELECT ...'` | Exécuter une requête |
| `bq query --dry_run '...'` | Estimer le coût |
| `bq show dataset.table` | Métadonnées d'une table |
| `bq extract dataset.table gs://bucket/export.csv` | Exporter vers GCS |

### SQL BigQuery

| Syntaxe | Description |
|---------|-------------|
| `CREATE OR REPLACE TABLE t AS SELECT ...` | CTAS |
| `PARTITION BY DATE(col)` | Partitionnement par date |
| `CLUSTER BY col1, col2` | Clustering |
| `CREATE OR REPLACE EXTERNAL TABLE ... OPTIONS(...)` | Table externe sur GCS |
| `ML.EVALUATE(MODEL m)` | Évaluer un modèle BQML |
| `ML.PREDICT(MODEL m, (SELECT ...))` | Prédire |
| `dataset.INFORMATION_SCHEMA.PARTITIONS` | Infos sur les partitions |
| `dataset.__TABLES__` | Taille et nb de lignes des tables |

### Python (google-cloud-bigquery)

| Code | Description |
|------|-------------|
| `bigquery.Client(project=...)` | Créer le client |
| `client.query(sql).to_dataframe()` | Requête → DataFrame pandas |
| `client.load_table_from_uri(uri, table, job_config)` | Charger depuis GCS |
| `client.get_table(table_id).num_rows` | Nombre de lignes |

---

## ❓ Questions de révision

1. **Pourquoi BigQuery est-il si rapide sur les agrégations ?**
   → Stockage **colonnaire** + exécution **massivement parallèle** (Dremel, milliers de slots) + séparation stockage/compute.

2. **Quelle est la différence de coût entre `SELECT col` et `SELECT *` ?**
   → La tarification on-demand dépend des octets scannés ; `SELECT *` scanne toutes les colonnes de toutes les lignes → coût maximal.

3. **Partitioning vs Clustering : quand utiliser l'un ou l'autre ?**
   → Partitioning : colonne de date, faible cardinalité, coût prévisible. Clustering : haute cardinalité, filtres multi-colonnes, tri fin dans les partitions.

4. **Une table externe est-elle performante ?**
   → Moins qu'une table native : les données restent dans GCS, sans optimisation BigQuery complète. À réserver à l'exploration.

5. **`LIMIT 10` réduit-il le coût d'une requête ?**
   → **Non** ! Le scan est identique. Pour réduire le coût : filtrer les partitions et sélectionner moins de colonnes.

6. **Comment estimer le coût d'une requête sans l'exécuter ?**
   → `bq query --dry_run`, le validateur de la console, ou `dry_run=True` dans le client Python.

7. **Quel modèle BQML pour prédire le pourboire d'une course ?**
   → `linear_reg` (régression) : le pourboire est une valeur continue.

---

> 📚 **Ressources** :
> - [Repo GitHub du Zoomcamp — Module 3](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/03-data-warehouse)
> - [Documentation BigQuery](https://cloud.google.com/bigquery/docs)
> - [BigQuery best practices (Google Cloud)](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
> - [BQML documentation](https://cloud.google.com/bigquery/docs/bqml-introduction)

---

## ✅ Récapitulatif

Cette fiche couvre l'intégralité du **Module 3 (Data Warehouse & BigQuery)** :

- **Concepts** : OLTP vs OLAP, stockage colonne vs ligne, warehouse vs lake
- **Architecture BigQuery** : séparation stockage/compute (Colossus, Dremel, slots)
- **Pratique** : chargement (bq, Python), tables externes vs natives
- **Optimisation** : **partitioning** et **clustering** (le cœur du module !), gestion des coûts
- **BQML** : machine learning directement en SQL
- **Aide-mémoire** + **questions de révision**

> 💡 Copiez ce contenu dans `module3-bigquery-data-warehouse.md`. Point clé du homework : comparez toujours les **bytes scannés** entre table brute et table partitionnée/clusterisée — c'est LA démonstration attendue !

Vous avez désormais les fiches des **Modules 3, 4, 5, 6 + Bruin**. Souhaitez-vous que je complète avec le **Module 1 (Docker/Terraform)** ou le **Module 2 (Workflow Orchestration avec Kestra)** pour avoir la collection complète ? 🚀
