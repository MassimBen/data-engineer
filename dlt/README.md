# Cours Complet : dlt (Data Load Tool)

> Basé sur le Workshop DataTalksClub — Data Engineering Zoomcamp 2026

---

## 📑 Table des matières

1. [Qu'est-ce que dlt ?](#1-quest-ce-que-dlt-)
2. [Installation & Setup](#2-installation--setup)
3. [Concepts Fondamentaux](#3-concepts-fondamentaux)
4. [Ingestion depuis une API REST](#4-ingestion-depuis-une-api-rest)
5. [Chargement Incrémental](#5-chargement-incrémental)
6. [Normalisation Automatique](#6-normalisation-automatique-auto-shredding)
7. [Dashboard & Validation](#7-dashboard--validation)
8. [Approche Assistée par IA (LLM-Native)](#8-approche-assistée-par-ia-llm-native)
9. [Changer de Destination](#9-changer-de-destination-local--cloud)
10. [Exemple Complet : Pipeline vers BigQuery](#10-exemple-complet--pipeline-vers-bigquery)
11. [Exploration avec Marimo & Ibis](#11-exploration-avec-marimo--ibis)
12. [Récapitulatif & Best Practices](#12-récapitulatif--best-practices)
13. [Ressources Complémentaires](#13-ressources-complémentaires)

---

## 1. Qu'est-ce que dlt ?

**dlt** (Data Load Tool) est une bibliothèque Python open-source qui automatise l'extraction, la normalisation et le chargement de données depuis diverses sources vers des destinations structurées.

Contrairement aux approches traditionnelles où vous écrivez manuellement des parsers et gérez les schémas, dlt s'occupe automatiquement de :

- **L'inférence de schéma** : détection et création automatique des tables
- **La normalisation** : décomposition automatique du JSON imbriqué en tables relationnelles
- **Le typage des données** : conversion automatique des types
- **Le chargement incrémental** : chargement uniquement des nouvelles données
- **La gestion des erreurs et retries** : fiabilité intégrée

---

## 2. Installation & Setup

### Installation de base

```bash
pip install dlt
```

### Avec DuckDB (pour le développement local)

```bash
pip install dlt[duckdb]
```

> **Pourquoi DuckDB ?** DuckDB est une base de données analytique rapide, sans dépendances externes, idéale pour prototyper localement.

---

## 3. Concepts Fondamentaux

### Pipeline dlt : Extract → Normalize → Load

Le fonctionnement d'un pipeline dlt suit trois étapes :

1. **Extract** : Récupération des données depuis la source (API, fichier, base de données)
2. **Normalize** : Transformation automatique des données (JSON imbriqué → tables relationnelles)
3. **Load** : Chargement dans la destination

### Structure de base d'un pipeline

```python
import dlt

# Définir le pipeline
pipeline = dlt.pipeline(
    pipeline_name="mon_pipeline",
    destination="duckdb",
    dataset_name="mon_dataset"
)

# Données source (exemple avec JSON imbriqué)
data = [
    {
        "vendor_name": "VTS",
        "record_hash": "b00361a396177a9cb410ff61f20015ad",
        "time": {
            "pickup": "2009-06-14 23:23:00",
            "dropoff": "2009-06-14 23:48:00"
        },
        "coordinates": {
            "start": {"lon": -73.787442, "lat": 40.641525},
            "end": {"lon": -73.980072, "lat": 40.742963}
        },
        "Payment": {
            "type": "Credit",
            "amt": 20.5,
            "tip": 9
        },
        "passengers": [
            {"name": "John", "rating": 4.9},
            {"name": "Jack", "rating": 3.9}
        ]
    }
]

# Exécuter le pipeline
info = pipeline.run(data, table_name="rides", write_disposition="replace")
print(info)
```

**Ce que dlt fait automatiquement :**

- Crée la table `rides` avec les colonnes appropriées
- Décompose les objets imbriqués (`time`, `coordinates`, `Payment`) en sous-colonnes
- Crée des tables enfants (`rides__passengers`) pour les listes
- Gère les types de données

---

## 4. Ingestion depuis une API REST

### Utilisation du RESTClient

dlt fournit un `RESTClient` intégré pour interagir facilement avec les APIs :

```python
import dlt
from dlt.sources.helpers.rest_client import RESTClient
from dlt.sources.helpers.rest_client.paginators import PageNumberPaginator

@dlt.resource(name="rides")
def ny_taxi():
    client = RESTClient(
        base_url="https://us-central1-dlthub-analytics.cloudfunctions.net",
        paginator=PageNumberPaginator(
            base_page=1,
            total_path=None
        )
    )

    for page in client.paginate("data_engineering_zoomcamp_api"):
        yield page  # Gestion mémoire via générateur

# Créer et exécuter le pipeline
pipeline = dlt.pipeline(destination="duckdb")
load_info = pipeline.run(ny_taxi, write_disposition="replace")
print(load_info)

# Explorer les données chargées
df = pipeline.dataset(dataset_type="default").rides.df()
```

### Gestion de la mémoire avec les générateurs

Pour les gros volumes de données, utilisez des **générateurs Python** (`yield`) au lieu de charger tout en mémoire :

```python
import json
import requests

def stream_download_jsonl(url):
    response = requests.get(url, stream=True)
    response.raise_for_status()
    for line in response.iter_lines():
        if line:
            yield json.loads(line)
```

---

## 5. Chargement Incrémental

Le chargement incrémental permet de ne charger **que les nouvelles données** à chaque exécution, ce qui rend les pipelines plus rapides et économiques.

### Deux méthodes supportées

| Méthode | Description | Cas d'usage |
|---------|-------------|-------------|
| **Append** | Ajoute uniquement les nouveaux enregistrements | Données immuables (événements, logs) |
| **Merge** | Met à jour les enregistrements existants | Données modifiables (statuts, paiements) |

### Exemple de chargement incrémental avec Merge

```python
@dlt.resource(primary_key="id", write_disposition="merge")
def events(updated=dlt.sources.incremental("updated_at")):
    yield from fetch_events(since=updated.last_value)
```

> **Note :** dlt stocke l'état (state) dans une table séparée à la destination pour suivre ce qui a déjà été traité.

---

## 6. Normalisation Automatique (Auto-Shredding)

L'un des points forts de dlt est sa capacité à transformer automatiquement le JSON imbriqué en **tables relationnelles propres**.

### Exemple de normalisation

**Données source (JSON) :**

```json
{
    "vendor_name": "VTS",
    "Payment": {"type": "Credit", "amt": 20.5},
    "passengers": [
        {"name": "John", "rating": 4.9},
        {"name": "Jack", "rating": 3.9}
    ]
}
```

**Tables créées par dlt :**

| Table | Contenu |
|-------|---------|
| `rides` | Table principale avec `vendor_name`, `payment__type`, `payment__amt` |
| `rides__passengers` | Table enfant avec `name`, `rating`, `_dlt_parent_id` |

Cette normalisation est appelée **"auto-shredding"** dans l'écosystème dlt.

---

## 7. Dashboard & Validation

### Visualiser le pipeline

dlt intègre un dashboard Streamlit pour inspecter vos pipelines :

```bash
pip install streamlit
dlt pipeline <nom_du_pipeline> show
```

Ce dashboard permet de visualiser :

- Les schémas de chaque table
- Les tables enfants créées depuis le JSON imbriqué
- Les traces d'exécution (extraction → normalisation → chargement)
- Les métadonnées (timestamps, nombre de lignes, succès/échecs)
- Les aperçus SQL des données

---

## 8. Approche Assistée par IA (LLM-Native)

Le workshop DataTalksClub 2026 met l'accent sur une approche moderne : **l'ingestion assistée par IA**.

### Le workflow recommandé

1. **Scaffold** : Initialiser depuis un template

   ```bash
   uvx dlt init github duckdb
   ```

2. **Générer** : Utiliser un agent IA (Cursor, Continue) pour générer la configuration

3. **Exécuter** → **Erreur** → **Corriger** : Boucle rapide d'itération

4. **Valider** : Utiliser le dashboard + MCP pour vérifier

5. **Explorer** : Analyser dans un notebook Marimo + Ibis

### MCP Server (Model Context Protocol)

dlt fournit un **MCP Server** qui connecte votre agent IA directement aux métadonnées de votre pipeline. Cela permet :

- Des requêtes en langage naturel sur vos données
- Une génération de code sans hallucination
- Un debugging en temps réel

```bash
# Exemple de conversation avec le MCP
User: "What tables are available?"
MCP: "Commits (4,500 rows), Contributors (131 rows), Commits__parents (nested child table)"
```

---

## 9. Changer de Destination (Local → Cloud)

L'un des grands avantages de dlt est la **portabilité du code**. Un pipeline développé avec DuckDB localement peut être déployé en production sur BigQuery, Snowflake ou MotherDuck **sans réécrire le code**.

```python
# Local (développement)
pipeline = dlt.pipeline(
    pipeline_name='taxi_data',
    destination='duckdb',
    dataset_name='taxi_rides',
)

# Production (cloud)
pipeline = dlt.pipeline(
    pipeline_name='taxi_data',
    destination='bigquery',  # ou 'snowflake'
    dataset_name='taxi_rides',
)
```

---

## 10. Exemple Complet : Pipeline vers BigQuery avec Datasets Reels

Cette section presente un **pipeline complet et fonctionnel** qui ingere deux datasets publics reels — les courses de taxis a New York — et les charge dans **Google BigQuery**.

### Les datasets utilises

| Dataset | Source | Format | Periode |
|---------|--------|--------|---------|
| **NYC Yellow Taxi** | [NYC TLC](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Parquet | Mensuel |
| **NYC Green Taxi** | [NYC TLC](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Parquet | Mensuel |

Les fichiers sont disponibles publiquement sur :
- `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_YYYY-MM.parquet`
- `https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_YYYY-MM.parquet`

### Architecture du pipeline

```
+---------------+     +-------------------+     +------------------+
|  NYC TLC S3   | --> |  dlt (Python)     | --> |  BigQuery        |
|  (Parquet)    |     |  - Download       |     |  - yellow_taxi   |
|               |     |  - Normalisation  |     |  - green_taxi    |
+---------------+     |  - Incremental    |     |  - _dlt_loads    |
                      +-------------------+     +------------------+
```

### Prerequis

#### 1. Creer un projet GCP et activer BigQuery

- Creez un projet sur [Google Cloud Console](https://console.cloud.google.com/)
- Activez l'API BigQuery
- Creez un dataset BigQuery (ex: `nyc_taxi_data`)

#### 2. Authentification GCP

**Option A — Compte de service (recommande pour la production)**

```bash
# Telechargez la cle JSON depuis GCP Console -> IAM -> Comptes de service
gcloud auth activate-service-account --key-file=/chemin/vers/service-account.json

# Ou definissez la variable d'environnement
export GOOGLE_APPLICATION_CREDENTIALS=/chemin/vers/service-account.json
```

**Option B — Authentification utilisateur (developpement)**

```bash
gcloud auth application-default login
```

#### 3. Permissions requises

Le compte de service doit avoir les roles suivants :
- `BigQuery Data Editor`
- `BigQuery Job User`

### Installation des dependances

```bash
pip install dlt[gcp] pandas pyarrow requests
```

### Code complet du pipeline

#### `pipeline_nyc_taxi.py` — Pipeline d'ingestion

```python
"""
Pipeline complet dlt -> BigQuery
Ingestion des datasets NYC Yellow & Green Taxi depuis des fichiers Parquet publics
"""

import dlt
import requests
import pandas as pd
from io import BytesIO
from datetime import datetime, timedelta


# ---------------------------------------------------------------------------
# CONFIGURATION
# ---------------------------------------------------------------------------

BASE_URL = "https://d37ci6vzurychx.cloudfront.net/trip-data"

# Periode a charger (ajustez selon vos besoins)
START_DATE = "2024-01"
END_DATE = "2024-03"


# ---------------------------------------------------------------------------
# SOURCE : Telechargement et lecture des fichiers Parquet
# ---------------------------------------------------------------------------

def download_parquet(taxi_type: str, year_month: str) -> pd.DataFrame:
    """
    Telecharge un fichier Parquet depuis l'URL publique NYC TLC.

    Args:
        taxi_type: 'yellow' ou 'green'
        year_month: Format 'YYYY-MM'

    Returns:
        DataFrame pandas avec les donnees du fichier
    """
    url = f"{BASE_URL}/{taxi_type}_tripdata_{year_month}.parquet"

    print(f"Telechargement : {url}")
    response = requests.get(url, timeout=120)
    response.raise_for_status()

    df = pd.read_parquet(BytesIO(response.content))

    # Ajout de metadonnees pour le tracking
    df["_dlt_taxi_type"] = taxi_type
    df["_dlt_file_month"] = year_month
    df["_dlt_loaded_at"] = datetime.utcnow().isoformat()

    print(f"  -> {len(df)} lignes chargees")
    return df


def get_months_range(start: str, end: str) -> list:
    """Genere une liste de mois au format YYYY-MM entre start et end."""
    months = []
    current = datetime.strptime(start, "%Y-%m")
    end_dt = datetime.strptime(end, "%Y-%m")

    while current <= end_dt:
        months.append(current.strftime("%Y-%m"))
        current += timedelta(days=32)
        current = current.replace(day=1)

    return months


# ---------------------------------------------------------------------------
# RESSOURCES dlt
# ---------------------------------------------------------------------------

@dlt.resource(
    name="yellow_taxi",
    write_disposition="replace",  # Remplacement complet pour la demo
    columns={
        "VendorID": {"data_type": "bigint"},
        "tpep_pickup_datetime": {"data_type": "timestamp"},
        "tpep_dropoff_datetime": {"data_type": "timestamp"},
        "passenger_count": {"data_type": "double"},
        "trip_distance": {"data_type": "double"},
        "PULocationID": {"data_type": "bigint"},
        "DOLocationID": {"data_type": "bigint"},
        "RatecodeID": {"data_type": "double"},
        "store_and_fwd_flag": {"data_type": "text"},
        "payment_type": {"data_type": "bigint"},
        "fare_amount": {"data_type": "double"},
        "extra": {"data_type": "double"},
        "mta_tax": {"data_type": "double"},
        "tip_amount": {"data_type": "double"},
        "tolls_amount": {"data_type": "double"},
        "improvement_surcharge": {"data_type": "double"},
        "total_amount": {"data_type": "double"},
        "congestion_surcharge": {"data_type": "double"},
        "Airport_fee": {"data_type": "double"},
    }
)
def yellow_taxi_trips():
    """
    Ressource dlt pour les courses de taxis jaunes (Yellow Taxi).
    Telecharge les fichiers Parquet mois par mois et yield les lignes.
    """
    months = get_months_range(START_DATE, END_DATE)

    for month in months:
        try:
            df = download_parquet("yellow", month)

            # Conversion en dictionnaires pour dlt
            records = df.to_dict(orient="records")
            yield records

        except Exception as e:
            print(f"Erreur lors du telechargement de yellow_{month}: {e}")
            continue


@dlt.resource(
    name="green_taxi",
    write_disposition="replace",
    columns={
        "VendorID": {"data_type": "bigint"},
        "lpep_pickup_datetime": {"data_type": "timestamp"},
        "lpep_dropoff_datetime": {"data_type": "timestamp"},
        "passenger_count": {"data_type": "double"},
        "trip_distance": {"data_type": "double"},
        "PULocationID": {"data_type": "bigint"},
        "DOLocationID": {"data_type": "bigint"},
        "RatecodeID": {"data_type": "double"},
        "store_and_fwd_flag": {"data_type": "text"},
        "payment_type": {"data_type": "bigint"},
        "fare_amount": {"data_type": "double"},
        "extra": {"data_type": "double"},
        "mta_tax": {"data_type": "double"},
        "tip_amount": {"data_type": "double"},
        "tolls_amount": {"data_type": "double"},
        "improvement_surcharge": {"data_type": "double"},
        "total_amount": {"data_type": "double"},
        "congestion_surcharge": {"data_type": "double"},
        "trip_type": {"data_type": "double"},
    }
)
def green_taxi_trips():
    """
    Ressource dlt pour les courses de taxis verts (Green Taxi).
    Telecharge les fichiers Parquet mois par mois et yield les lignes.
    """
    months = get_months_range(START_DATE, END_DATE)

    for month in months:
        try:
            df = download_parquet("green", month)
            records = df.to_dict(orient="records")
            yield records

        except Exception as e:
            print(f"Erreur lors du telechargement de green_{month}: {e}")
            continue


# ---------------------------------------------------------------------------
# PIPELINE : Configuration et execution
# ---------------------------------------------------------------------------

def run_pipeline():
    """Configure et execute le pipeline vers BigQuery."""

    # Configuration du pipeline BigQuery
    pipeline = dlt.pipeline(
        pipeline_name="nyc_taxi_pipeline",
        destination="bigquery",
        dataset_name="nyc_taxi_data",      # Nom du dataset BigQuery
        dev_mode=False,                    # False = tables persistantes
    )

    print("=" * 60)
    print("Demarrage du pipeline NYC Taxi -> BigQuery")
    print("=" * 60)

    # Execution
    load_info = pipeline.run([
        yellow_taxi_trips(),
        green_taxi_trips()
    ])

    print("\n" + "=" * 60)
    print("Pipeline execute avec succes !")
    print("=" * 60)
    print(load_info)
    print("\n")

    # Resume des tables creees
    print("Tables creees dans BigQuery :")
    with pipeline.sql_client() as client:
        try:
            tables = client.execute_sql("""
                SELECT table_name, row_count
                FROM `nyc_taxi_data.__TABLES__`
                ORDER BY table_name
            """)
            for table in tables:
                print(f"   • {table[0]} : {table[1]} lignes")
        except Exception as e:
            print(f"   (Impossible de recuperer le resume: {e})")

    return load_info


if __name__ == "__main__":
    run_pipeline()
```

#### `.dlt/config.toml` — Configuration avancee

```toml
# Fichier de configuration dlt (cree automatiquement avec `dlt init`)

[pipeline]
loader_file_format = "jsonl"          # Format de chargement : jsonl ou parquet

[normalize]
workers = 4                           # Parallelisation de la normalisation

[load]
workers = 4                           # Parallelisation du chargement

[destination.bigquery]
location = "EU"                       # Region BigQuery (US, EU, asia-northeast1...)
dataset_name_layout = "{dataset_name}"  # Nommage du dataset

# Options de performance
[destination.bigquery.http_timeout]
connect = 60
read = 300
```

#### `.dlt/secrets.toml` — Credentials (NE PAS VERSIONNER !)

```toml
# Ajoutez ce fichier a .gitignore !

[destination.bigquery.credentials]
project_id = "mon-projet-gcp"
private_key = "-----BEGIN PRIVATE KEY-----\n..."
client_email = "service-account@mon-projet-gcp.iam.gserviceaccount.com"
```

> **Alternative avec variables d'environnement :**
> ```bash
> export DESTINATION__BIGQUERY__CREDENTIALS__PROJECT_ID="mon-projet-gcp"
> export DESTINATION__BIGQUERY__CREDENTIALS__CLIENT_EMAIL="..."
> export DESTINATION__BIGQUERY__CREDENTIALS__PRIVATE_KEY="..."
> ```

### Execution du pipeline

```bash
# 1. Initialiser le projet dlt (cree les dossiers .dlt/)
dlt init nyc_taxi bigquery

# 2. Configurer les secrets dans .dlt/secrets.toml

# 3. Lancer le pipeline
python pipeline_nyc_taxi.py
```

### Resultat dans BigQuery

Apres execution, dlt cree automatiquement les tables suivantes dans votre dataset `nyc_taxi_data` :

| Table | Description |
|-------|-------------|
| `yellow_taxi` | Courses de taxis jaunes avec toutes les colonnes |
| `green_taxi` | Courses de taxis verts avec toutes les colonnes |
| `_dlt_loads` | Metadonnees des executions du pipeline |
| `_dlt_pipeline_state` | Etat du pipeline (pour l'incremental) |
| `_dlt_version` | Version du schema |

### Verification dans BigQuery

```sql
-- Nombre total de courses par type de taxi et par mois
SELECT
  _dlt_taxi_type,
  _dlt_file_month,
  COUNT(*) AS trip_count,
  ROUND(AVG(trip_distance), 2) AS avg_distance,
  ROUND(AVG(total_amount), 2) AS avg_total_amount
FROM `mon-projet-gcp.nyc_taxi_data.yellow_taxi`
GROUP BY 1, 2
ORDER BY 2;

-- Top 10 zones de depart les plus populaires (Yellow Taxi)
SELECT
  PULocationID,
  COUNT(*) AS pickup_count
FROM `mon-projet-gcp.nyc_taxi_data.yellow_taxi`
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;

-- Comparaison Yellow vs Green par mois
SELECT
  _dlt_file_month,
  SUM(CASE WHEN _dlt_taxi_type = 'yellow' THEN 1 ELSE 0 END) AS yellow_count,
  SUM(CASE WHEN _dlt_taxi_type = 'green' THEN 1 ELSE 0 END) AS green_count
FROM (
  SELECT _dlt_file_month, _dlt_taxi_type FROM `mon-projet-gcp.nyc_taxi_data.yellow_taxi`
  UNION ALL
  SELECT _dlt_file_month, _dlt_taxi_type FROM `mon-projet-gcp.nyc_taxi_data.green_taxi`
)
GROUP BY 1
ORDER BY 1;

-- Verifier la derniere execution du pipeline
SELECT *
FROM `mon-projet-gcp.nyc_taxi_data._dlt_loads`
ORDER BY load_id DESC
LIMIT 5;
```

### Version Incrementale (Production)

Pour un pipeline en production, il est preferable d'utiliser le **chargement incremental** afin de ne charger que les nouveaux mois :

```python
@dlt.resource(
    name="yellow_taxi",
    write_disposition="merge",
    primary_key="_dlt_file_month",  # Cle de merge
    columns={...}
)
def yellow_taxi_trips_incremental(
    updated=dlt.sources.incremental("_dlt_file_month")
):
    """
    Version incrementale : ne telecharge que les mois non encore charges.
    """
    months = get_months_range(START_DATE, END_DATE)

    # Filtrer les mois deja charges
    if updated.last_value:
        months = [m for m in months if m > updated.last_value]

    for month in months:
        try:
            df = download_parquet("yellow", month)
            records = df.to_dict(orient="records")
            yield records
        except Exception as e:
            print(f"Erreur: {e}")
            continue
```

### Transition DuckDB -> BigQuery

Le meme code fonctionne avec les deux destinations. Voici comment basculer :

```python
import dlt
import os

# Detection automatique de l'environnement
IS_PROD = os.getenv("ENV", "dev") == "prod"
destination = "bigquery" if IS_PROD else "duckdb"

pipeline = dlt.pipeline(
    pipeline_name="nyc_taxi_pipeline",
    destination=destination,
    dataset_name="nyc_taxi_data",
)

# Le reste du code est IDENTIQUE
load_info = pipeline.run([yellow_taxi_trips(), green_taxi_trips()])
```

### Bonnes pratiques BigQuery avec dlt

| Pratique | Description |
|----------|-------------|
| **Partitionnement** | Configurez `partition` sur `_dlt_file_month` ou `tpep_pickup_datetime` pour les grosses tables |
| **Clustering** | Utilisez `cluster` sur `PULocationID` ou `DOLocationID` pour optimiser les requetes geographiques |
| **Format Parquet** | Pour les tres gros volumes, utilisez `loader_file_format = "parquet"` |
| **Staging GCS** | Activez le staging GCS pour des chargements plus rapides |
| **Monitoring** | Surveillez la table `_dlt_loads` pour detecter les echecs |
| **Couts** | Utilisez `dev_mode=True` + DuckDB en local pour eviter les couts BigQuery en developpement |

### Exemple avec partitionnement et clustering

```python
@dlt.resource(
    name="yellow_taxi",
    write_disposition="merge",
    primary_key="_dlt_file_month",
    columns={
        "tpep_pickup_datetime": {
            "data_type": "timestamp",
            "partition": True,        # Partitionnement par date de course
        },
        "PULocationID": {
            "data_type": "bigint",
            "cluster": True,          # Clustering par zone de depart
        },
        "DOLocationID": {
            "data_type": "bigint",
            "cluster": True,          # Clustering par zone d'arrivee
        },
    }
)
def yellow_taxi_trips_partitioned(...):
    ...
```

---

## 11. Exploration avec Marimo & Ibis

Après avoir ingéré vos données avec dlt, l'étape suivante consiste à les **explorer, analyser et visualiser**. Le workshop DataTalksClub 2026 recommande la combinaison **Marimo + Ibis** pour une expérience interactive et portable.

### Qu'est-ce que Marimo ?

**Marimo** est un notebook Python réactif qui remplace Jupyter. Contrairement aux notebooks traditionnels, Marimo garantit :

- **Réactivité** : les cellules se réexécutent automatiquement quand leurs dépendances changent
- **Reproductibilité** : pas d'état caché, l'ordre d'exécution est toujours correct
- **Export** : possibilité d'exporter en script Python pur ou en application web
- **Widgets interactifs** : sliders, boutons, tableaux filtrables intégrés nativement

### Qu'est-ce que Ibis ?

**Ibis** est une bibliothèque d'analyse de données qui offre une API DataFrame unifiée sur **plusieurs backends** (DuckDB, PostgreSQL, BigQuery, Snowflake, etc.). Elle permet d'écrire du code Python qui se compile en SQL optimisé.

- **Aucune donnée ne transite en mémoire** : les calculs restent dans la base de données
- **Portabilité** : même code pour DuckDB local ou BigQuery en production
- **Performance** : profite de l'optimisation du moteur SQL sous-jacent

### Installation

```bash
pip install marimo ibis-framework[duckdb]
```

### Workflow complet : dlt → Ibis → Marimo

#### Étape 1 : Ingestion avec dlt

```python
import dlt

pipeline = dlt.pipeline(
    pipeline_name="github_pipeline",
    destination="duckdb",
    dataset_name="github_data"
)

# Charger les données (exemple avec l'API GitHub)
load_info = pipeline.run(github_source(), write_disposition="replace")
print(load_info)
```

#### Étape 2 : Connexion avec Ibis

```python
import ibis

# Se connecter à la base DuckDB créée par dlt
con = ibis.duckdb.connect("github_pipeline.duckdb")

# Lister les tables disponibles
print(con.list_tables())

# Charger une table comme expression Ibis (lazy)
commits = con.table("commits")

# Inspection du schéma
print(commits.schema())
```

#### Étape 3 : Analyse avec Ibis

```python
# Filtrer et agréger sans charger les données en mémoire
result = (
    commits
    .filter(commits["committer_date"] > "2024-01-01")
    .group_by("author_name")
    .agg(
        commit_count=commits["id"].count(),
        first_commit=commits["committer_date"].min(),
        last_commit=commits["committer_date"].max()
    )
    .order_by(ibis.desc("commit_count"))
    .limit(10)
)

# Exécuter et récupérer un DataFrame pandas
df = result.to_pandas()
print(df)
```

> **Astuce :** Utilisez `.sql()` pour écrire du SQL brut si besoin :
> ```python
> result = con.sql("SELECT * FROM commits WHERE committer_date > '2024-01-01'")
> ```

#### Étape 4 : Visualisation dans Marimo

Créez un notebook Marimo et explorez vos données interactivement :

```python
# Cellule 1 : Imports
import marimo as mo
import ibis
import altair as alt

# Cellule 2 : Connexion
con = ibis.duckdb.connect("github_pipeline.duckdb")
commits = con.table("commits")

# Cellule 3 : Widget interactif
author_filter = mo.ui.text(label="Filtrer par auteur", value="")

# Cellule 4 : Requête réactive (se réexécute automatiquement)
filtered = (
    commits
    .filter(commits["author_name"].contains(author_filter.value))
    .group_by(commits["committer_date"].truncate("M").name("month"))
    .agg(count=commits["id"].count())
    .order_by("month")
)

# Cellule 5 : Affichage
mo.hstack([author_filter, filtered.to_pandas()])

# Cellule 6 : Graphique avec Altair
chart = alt.Chart(filtered.to_pandas()).mark_line().encode(
    x="month:T",
    y="count:Q"
)
mo.md(f"## Évolution des commits\n{chart}")
```

### Pourquoi cette stack ?

| Outil | Rôle | Avantage |
|-------|------|----------|
| **dlt** | Ingestion | Automatise l'extraction et la normalisation |
| **Ibis** | Transformation | Code Python portable qui compile en SQL |
| **Marimo** | Exploration | Notebook réactif, reproductible, exportable |

### Exemple complet : Pipeline d'analyse de contributions GitHub

```python
# --- pipeline.py (dlt) ---
import dlt
from dlt.sources.helpers.rest_client import RESTClient

@dlt.resource(name="commits")
def github_commits(owner="DataTalksClub", repo="data-engineering-zoomcamp"):
    client = RESTClient(base_url="https://api.github.com")
    yield from client.paginate(f"/repos/{owner}/{repo}/commits")

pipeline = dlt.pipeline(destination="duckdb", dataset_name="github")
pipeline.run(github_commits())

# --- explore.py (Marimo notebook) ---
# %% Cellule 1
import marimo as mo
import ibis
import altair as alt

# %% Cellule 2
con = ibis.duckdb.connect("github.duckdb")
commits = con.table("commits")

# %% Cellule 3
# Top contributeurs
top_authors = (
    commits
    .group_by("commit__author__name")
    .agg(count=commits["sha"].count())
    .order_by(ibis.desc("count"))
    .limit(20)
)

# %% Cellule 4
mo.md("## Top 20 contributeurs")
mo.ui.table(top_authors.to_pandas(), selection=None)

# %% Cellule 5
# Timeline des commits
timeline = (
    commits
    .mutate(
        date=commits["commit__committer__date"].cast("date")
    )
    .group_by("date")
    .agg(commits_count=commits["sha"].count())
    .order_by("date")
)

chart = alt.Chart(timeline.to_pandas()).mark_bar().encode(
    x="date:T",
    y="commits_count:Q",
    tooltip=["date", "commits_count"]
).interactive()

mo.md(f"## Timeline des commits\n{chart}")
```

### Lancer le notebook Marimo

```bash
# Mode édition
marimo edit explore.py

# Mode exécution (application web)
marimo run explore.py

# Exporter en script Python
marimo export script explore.py -o explore_script.py

# Exporter en HTML statique
marimo export html explore.py -o explore.html
```

### Bonnes pratiques Marimo + Ibis + dlt

1. **Séparez ingestion et analyse** : gardez `pipeline.py` et `explore.py` dans des fichiers distincts
2. **Utilisez Ibis pour tout le traitement** : évitez de charger des DataFrames pandas complets en mémoire
3. **Profitez de la réactivité Marimo** : les widgets filtrent et mettent à jour les graphiques automatiquement
4. **Versionnez vos notebooks** : Marimo produit du Python pur, facilement versionnable avec Git
5. **Passez à l'échelle** : remplacez `ibis.duckdb` par `ibis.bigquery` sans changer le code d'analyse

---

## 12. Récapitulatif & Best Practices

| Concept | Bonne pratique |
|---------|---------------|
| **Mémoire** | Utilisez toujours `yield` pour les gros volumes |
| **Pagination** | Utilisez les paginateurs intégrés (`PageNumberPaginator`, etc.) |
| **Schéma** | Laissez dlt inférer automatiquement, validez avec le dashboard |
| **Incrémental** | Utilisez `merge` pour les données modifiables, `append` pour les événements |
| **IA** | Commencez par un scaffold, itérez avec des boucles courtes |
| **Validation** | Inspectez toujours avec `dlt pipeline <name> show` |
| **Analyse** | Préférez Ibis aux DataFrames pandas pour les gros volumes |
| **Exploration** | Utilisez Marimo pour des notebooks réactifs et reproductibles |

---

## 13. Ressources Complémentaires

- 📹 [Workshop DataTalksClub complet sur YouTube](https://www.youtube.com/watch?v=5eMytPBgmVs)
- 📓 [Notebook Google Colab officiel](https://github.com/DataTalksClub/data-engineering-zoomcamp)
- 📚 [Documentation dlt](https://dlthub.com/docs)
- 💬 [Communauté Slack dlt](https://dlthub.com/community)
- 📝 [Documentation Marimo](https://docs.marimo.io/)
- 🦆 [Documentation Ibis](https://ibis-project.org/)

---

> Ce cours couvre l'essentiel du workshop DataTalksClub sur dlt. L'approche moderne privilégie les **pipelines config-driven**, la **validation systématique** et l'**assistance IA** pour construire des systèmes d'ingestion fiables et scalables.
