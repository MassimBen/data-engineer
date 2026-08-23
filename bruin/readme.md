Voici la fiche de révision sur **Bruin** (outil de pipelines de données utilisé dans le Zoomcamp), au format Markdown prêt pour GitHub.

> ⚠️ **Transparence** : le module Bruin du Zoomcamp est relativement récent et mon contenu peut comporter des imprécisions sur certains détails syntaxiques exacts. Les concepts présentés ici sont exacts, mais je vous conseille de vérifier les syntaxes avec la [documentation officielle Bruin](https://bruin-data.github.io/bruin/) et le repo du Zoomcamp.

---

# Bruin — Data Pipelines & Ingestion

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision : Construire des pipelines de données avec Bruin (+ DuckDB, ingestion, qualité des données)

---

## 📑 Table des matières

1. [Introduction à Bruin](#1-introduction-à-bruin)
2. [Installation et premiers pas](#2-installation-et-premiers-pas)
3. [Concepts fondamentaux](#3-concepts-fondamentaux)
4. [Les assets Bruin](#4-les-assets-bruin)
5. [Les assets SQL](#5-les-assets-sql)
6. [Les assets Python](#6-les-assets-python)
7. [L'ingestion de données (ingestr)](#7-lingestion-de-données-ingestr)
8. [La qualité des données](#8-la-qualité-des-données)
9. [Matérialisation et stratégies](#9-matérialisation-et-stratégies)
10. [Pipeline complet : cas pratique](#10-pipeline-complet--cas-pratique)
11. [DuckDB comme entrepôt local](#11-duckdb-comme-entrepôt-local)
12. [Bruin vs dbt et autres outils](#12-bruin-vs-dbt-et-autres-outils)
13. [Aide-mémoire](#13-aide-mémoire-commandes-et-syntaxe)

---

## 1. Introduction à Bruin

### 1.1 Qu'est-ce que Bruin ?

- Outil **open-source** de **pipelines de données** (écrit en Go)
- Permet de construire des pipelines **end-to-end** : **ingestion → transformation → validation**
- Unifie ce qui nécessitait auparavant plusieurs outils :
  - **Ingestion** (via **ingestr**, intégré)
  - **Transformation SQL / Python**
  - **Qualité des données** (checks intégrés)
  - **Orchestration** des dépendances entre assets

### 1.2 Positionnement dans le modern data stack

```
Sources          Ingestion        Stockage         Transformation      BI
(API, CSV,  ──►   Bruin      ──►  DuckDB /     ──►  Bruin (SQL/Python) ──► Dashboards
 DB, SaaS)       (ingestr)        BigQuery /
                                  Postgres
```

### 1.3 Pourquoi Bruin ?

| Besoin | Avant (plusieurs outils) | Avec Bruin |
|--------|--------------------------|------------|
| Ingestion | Airbyte, Fivetran, scripts custom | `bruin run` (ingestr intégré) |
| Transformation | dbt | Assets SQL/Python natifs |
| Qualité | dbt tests, Great Expectations | Checks intégrés dans l'asset |
| Orchestration | Airflow, Dagster | Dépendances résolues automatiquement |

> 💡 Bruin est un outil **"batteries included"** : un seul binaire, pas de serveur lourd à installer, idéal pour les pipelines locaux et en production légère.

---

## 2. Installation et premiers pas

### 2.1 Installation

```bash
# Mac / Linux
curl -LsSf https://getbruin.com/install/cli | sh

# Vérifier l'installation
bruin --version

# Extension VS Code recommandée : "Bruin"
# (visualisation du lineage, exécution depuis l'éditeur)
```

### 2.2 Créer un projet

```bash
# Initialiser un nouveau projet Bruin
bruin init my-pipeline

# Structure générée
cd my-pipeline
```

```
my-pipeline/
├── .bruin.yml              # Configuration des connexions
├── pipeline.yml            # Métadonnées du pipeline
└── pipeline/
    ├── assets/             # Les assets (SQL, Python, ingestion)
    │   ├── my_asset.asset.yml
    │   └── my_asset.sql
    └── requirements.txt    # Dépendances Python (si besoin)
```

### 2.3 Commande de base

```bash
# Exécuter le pipeline complet
bruin run

# Exécuter un asset spécifique
bruin run pipeline/assets/my_asset.sql

# Valider le pipeline (syntaxe, dépendances)
bruin validate ./pipeline
```

---

## 3. Concepts fondamentaux

### 3.1 Le vocabulaire Bruin

| Concept | Définition |
|---------|------------|
| **Pipeline** | Ensemble d'assets liés par des dépendances |
| **Asset** | Unité de travail : une table, un fichier, un traitement |
| **Connection** | Connexion à une source/destination (DuckDB, Postgres, BigQuery...) |
| **Dependency** | Relation entre assets (upstream → downstream) |
| **Materialization** | Comment le résultat d'un asset est stocké (table, view, incremental...) |
| **Quality check** | Test de qualité exécuté après l'asset |

### 3.2 Le graphe de dépendances (DAG)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  ingestion    │────►│  staging     │────►│   marts      │
│  (raw data)   │     │  (nettoyage) │     │  (analytique)│
└──────────────┘     └──────────────┘     └──────────────┘
```

- Bruin **détecte automatiquement** les dépendances via `depends`
- Il exécute les assets **dans le bon ordre**
- L'extension VS Code permet de **visualiser le lineage**

### 3.3 Cycle de vie d'un asset

```
Définition (yml/sql/py) ──► Validation ──► Résolution des dépendances
       ──► Exécution ──► Matérialisation ──► Quality checks
```

---

## 4. Les assets Bruin

### 4.1 Types d'assets

| Type | Extension | Usage |
|------|-----------|-------|
| **Ingestion** | `.asset.yml` | Charger des données depuis une source externe |
| **SQL** | `.sql` | Transformer des données en SQL |
| **Python** | `.py` | Traitements personnalisés en Python |
| **Seed** | fichier CSV | Charger des données de référence |

### 4.2 Anatomie d'un asset SQL

Un asset SQL Bruin est un fichier `.sql` avec un **bloc de configuration en commentaires** en en-tête :

```sql
/* @bruin

name: staging.trips
type: bq.sql                     # ou duckdb.sql, postgres.sql...
description: Nettoyage des données de courses de taxi

materialization:
  type: table
  strategy: create+replace

depends:
  - raw.taxi_trips

columns:
  - name: trip_id
    type: integer
    checks:
      - name: unique
      - name: not_null
  - name: total_amount
    type: float
    checks:
      - name: positive

@bruin */

SELECT
    trip_id,
    pickup_datetime,
    dropoff_datetime,
    total_amount
FROM raw.taxi_trips
WHERE total_amount IS NOT NULL
```

### 4.3 Sections clés du bloc `@bruin`

| Clé | Rôle |
|-----|------|
| `name` | Nom de l'asset (format `schema.table`) |
| `type` | Type d'exécution (`duckdb.sql`, `bq.sql`, `python`...) |
| `depends` | Liste des assets upstream |
| `materialization` | Comment persister le résultat |
| `columns` | Schéma attendu + quality checks |
| `owner` / `description` | Documentation |

> 💡 **Analogie dbt** : `depends` ≈ `ref()`/`source()`, `materialization` ≈ `materialized`, `checks` ≈ `tests`.

---

## 5. Les assets SQL

### 5.1 Exemple avec DuckDB

```sql
/* @bruin
name: marts.payment_summary
type: duckdb.sql
materialization:
  type: table
depends:
  - staging.trips
@bruin */

SELECT
    payment_type,
    COUNT(*) AS trip_count,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_fare
FROM staging.trips
GROUP BY payment_type
ORDER BY total_revenue DESC
```

### 5.2 Variables et templating

Bruin supporte le templating (Jinja-like) pour les dates d'exécution :

```sql
/* @bruin
name: staging.daily_trips
type: duckdb.sql
@bruin */

SELECT *
FROM raw.trips
WHERE pickup_date BETWEEN '{{ start_date }}' AND '{{ end_date }}'
```

```bash
# Passer des variables à l'exécution
bruin run --start-date 2025-01-01 --end-date 2025-01-31
```

---

## 6. Les assets Python

### 6.1 Structure

Un asset Python a un bloc `@bruin` dans les commentaires, et la logique dans le code :

```python
"""@bruin
name: staging.enriched_trips
type: python
depends:
  - staging.trips
connection: duckdb-default
@bruin"""

import pandas as pd
import duckdb

def materialize():
    conn = duckdb.connect("duckdb.db")
    df = conn.execute("SELECT * FROM staging.trips").df()
    
    # Traitement Python
    df["fare_per_km"] = df["total_amount"] / df["trip_distance"]
    
    return df  # Bruin matérialise le DataFrame retourné
```

### 6.2 Quand utiliser Python plutôt que SQL ?

- Logique complexe difficile à exprimer en SQL
- Appels d'API, ML, parsing de formats exotiques
- Manipulation de données avec pandas

---

## 7. L'ingestion de données (ingestr)

### 7.1 Principe

- Bruin intègre **ingestr** pour charger des données depuis des dizaines de sources
- Un asset d'ingestion est défini en **YAML pur** (`.asset.yml`), pas de code !

### 7.2 Exemple : CSV vers DuckDB

```yaml
# pipeline/assets/taxi_trips.asset.yml
name: raw.taxi_trips
type: ingestr

parameters:
  source_connection: csv-source
  source_table: 'taxi_tripdata.csv'

  destination_connection: duckdb-default
  destination_table: raw.taxi_trips
```

### 7.3 Configuration des connexions

Dans `.bruin.yml` :

```yaml
environments:
  default:
    connections:
      duckdb:
        - name: duckdb-default
          path: duckdb.db
      postgres:
        - name: pg-default
          host: localhost
          port: 5432
          username: postgres
          password: secret
          database: mydb
```

### 7.4 Sources supportées (exemples)

- Bases de données : PostgreSQL, MySQL, SQL Server, BigQuery, Snowflake
- Fichiers : CSV, Parquet, JSON
- SaaS / API : Google Sheets, Notion, Stripe, Shopify, Chess...
- Stockage : S3, GCS

---

## 8. La qualité des données

### 8.1 Checks intégrés

Les checks sont définis **dans l'asset**, au niveau des colonnes :

```yaml
columns:
  - name: trip_id
    type: integer
    checks:
      - name: unique
      - name: not_null
  - name: fare_amount
    type: float
    checks:
      - name: positive
      - name: accepted_values
        value: [1, 2, 3]     # pour les valeurs catégorielles
```

### 8.2 Checks disponibles

| Check | Description |
|-------|-------------|
| `unique` | Pas de doublons |
| `not_null` | Pas de valeurs nulles |
| `positive` | Valeurs > 0 |
| `negative` / `non_negative` | Signe des valeurs |
| `accepted_values` | Valeurs dans une liste autorisée |
| `min` / `max` | Bornes numériques |
| `pattern` | Correspond à une regex |

### 8.3 Checks personnalisés en SQL

```yaml
custom_checks:
  - name: revenue_coherence
    query: |
      SELECT COUNT(*) FROM staging.trips
      WHERE total_amount < fare_amount
    value: 0
```

> 💡 Les checks s'exécutent **après** la matérialisation. Un échec **bloque le pipeline** (comportement configurable).

---

## 9. Matérialisation et stratégies

### 9.1 Types de matérialisation

| Type | Description |
|------|-------------|
| `table` | Crée une table physique |
| `view` | Crée une vue (requête stockée) |
| `none` | Pas de persistance (traitement intermédiaire) |

### 9.2 Stratégies

| Stratégie | Comportement | Cas d'usage |
|-----------|--------------|-------------|
| `create+replace` | Supprime et recrée la table | Données recalculées entièrement |
| `append` | Ajoute les lignes | Logs, nouvelles données |
| `merge` | Upsert sur une clé | Données mises à jour |
| `delete+insert` | Supprime une période puis insère | Reprocessing partitionné |
| `time_interval` | Insertion par intervalle de temps | Pipelines incrémentaux datés |

### 9.3 Exemple incrémental

```sql
/* @bruin
name: staging.trips_incremental
type: duckdb.sql
materialization:
  type: table
  strategy: time_interval
  incremental_key: pickup_date
depends:
  - raw.taxi_trips
@bruin */

SELECT *
FROM raw.taxi_trips
WHERE pickup_date BETWEEN '{{ start_date }}' AND '{{ end_date }}'
```

> 💡 **Analogie dbt** : `strategy: merge` ≈ `materialized='incremental'` avec `unique_key`.

---

## 10. Pipeline complet : cas pratique

### 10.1 Architecture type (données NYC Taxi)

```
taxi_tripdata.csv
       │
       ▼
┌─────────────────┐
│ raw.taxi_trips  │  ← Asset ingestr (YAML)
└────────┬────────┘
         ▼
┌─────────────────┐
│ staging.trips   │  ← Asset SQL (nettoyage, typage, filtres)
└────────┬────────┘
         ▼
┌─────────────────┐
│ marts.revenue   │  ← Asset SQL (agrégations pour le reporting)
└─────────────────┘
```

### 10.2 Les trois fichiers

**1. Ingestion** — `assets/taxi_trips.asset.yml`
```yaml
name: raw.taxi_trips
type: ingestr
parameters:
  source_connection: csv-source
  source_table: 'yellow_tripdata.csv'
  destination_connection: duckdb-default
  destination_table: raw.taxi_trips
```

**2. Staging** — `assets/staging_trips.sql`
```sql
/* @bruin
name: staging.trips
type: duckdb.sql
materialization:
  type: table
depends:
  - raw.taxi_trips
columns:
  - name: total_amount
    checks:
      - name: not_null
@bruin */

SELECT *
FROM raw.taxi_trips
WHERE passenger_count > 0
  AND trip_distance > 0
```

**3. Mart** — `assets/marts_revenue.sql`
```sql
/* @bruin
name: marts.monthly_revenue
type: duckdb.sql
depends:
  - staging.trips
@bruin */

SELECT
    DATE_TRUNC('month', tpep_pickup_datetime) AS month,
    COUNT(*) AS trips,
    SUM(total_amount) AS revenue
FROM staging.trips
GROUP BY 1
ORDER BY 1
```

### 10.3 Exécution

```bash
# Valider
bruin validate ./pipeline

# Exécuter tout le pipeline
bruin run

# Exécuter un asset + ses dépendances downstream
bruin run --downstream pipeline/assets/staging_trips.sql

# Exécuter seulement l'asset
bruin run --only pipeline/assets/staging_trips.sql
```

---

## 11. DuckDB comme entrepôt local

### 11.1 Pourquoi DuckDB ?

- Base analytique **embarquée** (comme SQLite, mais **colonnaire**)
- Aucun serveur à installer — un simple fichier `.db`
- **Très rapide** sur les requêtes analytiques (OLAP)
- Lit nativement **Parquet, CSV, JSON** et même **S3/GCS**

### 11.2 Inspection rapide

```bash
# CLI DuckDB
duckdb duckdb.db

# Requêtes
D SHOW TABLES;
D SELECT COUNT(*) FROM raw.taxi_trips;
D DESCRIBE staging.trips;
```

### 11.3 Requêter des fichiers directement

```sql
-- DuckDB peut interroger les fichiers sans import !
SELECT * FROM 'yellow_tripdata_2025-01.parquet' LIMIT 10;
SELECT * FROM read_csv_auto('zones.csv');
```

> 💡 Combo idéal pour le cours : **Bruin + DuckDB** = pipeline de données complet qui tourne **100% en local**, gratuit, sans infrastructure.

---

## 12. Bruin vs dbt et autres outils

| Critère | dbt | Bruin |
|---------|-----|-------|
| **Périmètre** | Transformation uniquement (le T de ELT) | Ingestion + transformation + qualité |
| **Langage principal** | SQL + Jinja + YAML | SQL / Python / YAML |
| **Ingestion** | ❌ (besoin d'Airbyte/Fivetran) | ✅ ingestr intégré |
| **Assets Python** | Limité (dbt >= 1.3, cloud DWH) | ✅ Natif et simple |
| **Orchestration** | Non (dbt Cloud scheduler, ou Airflow) | Ordonnancement des dépendances intégré |
| **Cible** | Data warehouses (BigQuery, Snowflake...) | Warehouses + DuckDB local |
| **Maturité** | Très mature, standard de l'industrie | Plus jeune, en croissance rapide |
| **Installation** | pip + adaptateurs | Un binaire unique (Go) |

> 💡 dbt reste le **standard industriel** pour la transformation sur warehouse ; Bruin est excellent pour des pipelines **end-to-end légers**, du prototypage local et des équipes qui veulent un seul outil.

---

## 13. Aide-mémoire : commandes et syntaxe

### CLI

| Commande | Description |
|----------|-------------|
| `bruin init <nom>` | Créer un nouveau projet |
| `bruin run` | Exécuter le pipeline complet |
| `bruin run <chemin/asset>` | Exécuter un asset spécifique |
| `bruin run --downstream <asset>` | Asset + tout ce qui en dépend |
| `bruin run --start-date <date>` | Exécution avec dates (incrémental) |
| `bruin validate ./pipeline` | Valider syntaxe et dépendances |
| `bruin connections list` | Lister les connexions configurées |
| `bruin render <asset>` | Voir le SQL compilé (templating résolu) |
| `bruin clean` | Nettoyer les artefacts temporaires |

### Bloc d'asset minimal

```sql
/* @bruin
name: schema.mon_asset
type: duckdb.sql
materialization:
  type: table
depends:
  - schema.asset_amont
@bruin */

SELECT ...
```

### Asset d'ingestion minimal

```yaml
name: raw.ma_table
type: ingestr
parameters:
  source_connection: <connexion-source>
  source_table: <table-ou-fichier>
  destination_connection: duckdb-default
  destination_table: raw.ma_table
```

---

## ❓ Questions de révision

1. **Qu'est-ce qui différencie fondamentalement Bruin de dbt ?**
   → dbt ne fait que la transformation ; Bruin couvre aussi l'ingestion (ingestr) et la qualité, dans un seul outil.

2. **Comment Bruin connaît-il l'ordre d'exécution des assets ?**
   → Via les clés `depends` déclarées dans chaque asset, qui forment un DAG.

3. **Quelle stratégie de matérialisation choisir pour retraiter une période précise ?**
   → `delete+insert` ou `time_interval` avec `--start-date` / `--end-date`.

4. **Pourquoi DuckDB est-il souvent utilisé avec Bruin dans le cours ?**
   → Base analytique embarquée, zéro serveur, très rapide, lit Parquet/CSV nativement → pipeline 100% local.

5. **Où définit-on les tests de qualité dans Bruin ?**
   → Directement dans l'asset, au niveau des `columns` (checks) ou en `custom_checks`.

6. **Quel est le rôle du fichier `.bruin.yml` ?**
   → Définir les environnements et les connexions aux sources/destinations.

---

> 📚 **Ressources** :
> - [Documentation officielle Bruin](https://bruin-data.github.io/bruin/)
> - [Repo GitHub Bruin](https://github.com/bruin-data/bruin)
> - [Repo GitHub du Zoomcamp](https://github.com/DataTalksClub/data-engineering-zoomcamp)
> - [Documentation ingestr](https://bruin-data.github.io/ingestr/)



## Récapitulatif

Cette fiche couvre :

- **Les concepts** : pipelines, assets, dépendances (DAG), matérialisation
- **Les 3 types d'assets** : ingestion (YAML), SQL, Python
- **La qualité des données** intégrée (checks sur colonnes + custom checks)
- **L'ingestion** avec ingestr et les connexions `.bruin.yml`
- **DuckDB** comme entrepôt local (combo gagnant du cours)
- **Comparaison avec dbt** (utile pour relier au Module 4 !)
- **Aide-mémoire** des commandes CLI et questions de révision

> ⚠️ **Rappel** : Bruin évolue vite — vérifiez la syntaxe exacte des assets (notamment `ingestr` et les `custom_checks`) avec la doc officielle et les vidéos du Zoomcamp avant de publier.


