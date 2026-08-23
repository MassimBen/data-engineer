# Module 2 (Ancienne version) — Workflow Orchestration avec Apache Airflow

> **Data Engineering Zoomcamp — DataTalksClub (éditions 2022–2023)**
> Fiche de révision : Orchestration de pipelines avec Apache Airflow
> ⚠️ Depuis 2024, le Module 2 utilise **Kestra**. Cette fiche couvre la version historique avec **Airflow**, toujours très demandée en entretien.

---

## 📑 Table des matières

1. [Introduction à l'orchestration et à Airflow](#1-introduction-à-lorchestration-et-à-airflow)
2. [Architecture d'Airflow](#2-architecture-dairflow)
3. [Installation avec Docker Compose](#3-installation-avec-docker-compose)
4. [Concepts fondamentaux : DAG, Tasks, Operators](#4-concepts-fondamentaux--dag-tasks-operators)
5. [Les Operators](#5-les-operators)
6. [Écrire son premier DAG](#6-écrire-son-premier-dag)
7. [Paramètres du DAG : schedule, catchup, dates](#7-paramètres-du-dag--schedule-catchup-dates)
8. [Templating Jinja et macros](#8-templating-jinja-et-macros)
9. [Cas pratique local : ingestion NYC Taxi vers Postgres](#9-cas-pratique-local--ingestion-nyc-taxi-vers-postgres)
10. [Cas pratique GCP : ingestion vers GCS et BigQuery](#10-cas-pratique-gcp--ingestion-vers-gcs-et-bigquery)
11. [Transfers GCS → BigQuery](#11-transfers-gcs--bigquery)
12. [XComs : partage de données entre tâches](#12-xcoms--partage-de-données-entre-tâches)
13. [Bonnes pratiques](#13-bonnes-pratiques)
14. [Aide-mémoire](#14-aide-mémoire)

---

## 1. Introduction à l'orchestration et à Airflow

### 1.1 Rappel : pourquoi orchestrer ?

Un pipeline de données = plusieurs étapes **dépendantes** qu'il faut :
- **Planifier** (scheduling type cron)
- **Enchaîner** dans le bon ordre (DAG)
- **Surveiller** (UI, logs, alertes)
- **Rejouer** en cas d'échec (retries) ou pour l'historique (**backfill**)

### 1.2 Apache Airflow

- Orchestrateur open source créé chez **Airbnb** (2014), projet Apache depuis 2016
- Les workflows sont définis **en Python** ("workflows as code")
- DAG = **Directed Acyclic Graph** : graphe de tâches orienté, **sans cycle**
- UI web riche : monitoring, Gantt, logs, retry manuel, backfill

> ⚠️ Airflow **orchestre**, il ne **transporte pas** la donnée : les tâches exécutent des scripts qui déplacent les données ; Airflow gère l'ordre, la planification et la supervision.

---

## 2. Architecture d'Airflow

### 2.1 Composants

| Composant | Rôle |
|-----------|------|
| **Webserver** | UI web (visualisation des DAGs, logs, triggers manuels) — port 8080 |
| **Scheduler** | Lit les DAGs, planifie et déclenche les tâches |
| **Executor** | Mécanisme d'exécution des tâches (Local, Celery, Kubernetes) |
| **Workers** | Exécutent réellement les tâches (selon l'executor) |
| **Metadata DB** | PostgreSQL/MySQL : état des DAGs, tâches, exécutions |
| **DAG folder** | Répertoire des fichiers Python définissant les DAGs (`./dags`) |

### 2.2 Executors

| Executor | Usage |
|----------|-------|
| `SequentialExecutor` | 1 tâche à la fois (démo uniquement, SQLite) |
| `LocalExecutor` | Parallélisme sur une machine (celui du cours) |
| `CeleryExecutor` | Workers distribués via une file (Redis/RabbitMQ) |
| `KubernetesExecutor` | 1 pod par tâche (production cloud-native) |

### 2.3 Schéma

```
                 ┌──────────────┐
                 │  Webserver   │ (UI :8080)
                 └──────┬───────┘
                        │
┌──────────┐    ┌───────┴────────┐    ┌──────────────┐
│  dags/   │───►│   Scheduler    │───►│  Executor /  │
│ (Python) │    │                │    │   Workers    │
└──────────┘    └───────┬────────┘    └──────────────┘
                        │
                 ┌──────┴───────┐
                 │ Metadata DB  │
                 │  (Postgres)  │
                 └──────────────┘
```

---

## 3. Installation avec Docker Compose

### 3.1 Docker Compose officiel

Le cours part du `docker-compose.yaml` officiel d'Airflow :

```bash
# Récupérer le docker-compose officiel
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.5.1/docker-compose.yaml'

# Créer les dossiers
mkdir -p ./dags ./logs ./plugins

# UID de l'utilisateur (permissions Linux)
echo -e "AIRFLOW_UID=$(id -u)" > .env

# Initialiser la metadata DB + créer l'utilisateur admin
docker compose up airflow-init

# Lancer Airflow
docker compose up -d
```

### 3.2 Services du docker-compose

| Service | Rôle |
|---------|------|
| `airflow-webserver` | UI → http://localhost:8080 (login: `airflow`/`airflow`) |
| `airflow-scheduler` | Planification des tâches |
| `airflow-worker` | Exécution (si CeleryExecutor) |
| `airflow-init` | Initialisation one-shot (DB + user) |
| `postgres` | Metadata DB |
| `redis` | Broker (si Celery) |

### 3.3 Personnalisations du cours

```yaml
# docker-compose.yaml (extraits adaptés au cours)
x-airflow-common:
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'   # ne pas charger les exemples
    GOOGLE_APPLICATION_CREDENTIALS: /.google/credentials/google_credentials.json
    GCP_PROJECT_ID: 'mon-projet-gcp'
    GCP_GCS_BUCKET: 'mon-bucket'
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ~/.google/credentials/:/.google/credentials:ro   # clé GCP
```

> 🔑 Monter la clé de service GCP en volume + définir `GOOGLE_APPLICATION_CREDENTIALS` permet aux tâches d'appeler GCS/BigQuery.

---

## 4. Concepts fondamentaux : DAG, Tasks, Operators

### 4.1 Vocabulaire

| Terme | Définition |
|-------|-----------|
| **DAG** | Workflow : ensemble de tâches + leurs dépendances + un schedule |
| **Task** | Unité de travail, instance d'un **Operator** |
| **Operator** | Modèle de tâche (Bash, Python, Docker...) |
| **Task Instance** | Exécution d'une tâche pour une date donnée |
| **DagRun** | Exécution complète d'un DAG pour une date donnée |
| **execution_date** | Date **logique** du run (début de l'intervalle de données) |

### 4.2 Cycle de vie d'une tâche

```
none → scheduled → queued → running → success
                                  └→ failed → up_for_retry → ...
```

### 4.3 Dépendances entre tâches

```python
# Notation bitshift
task_1 >> task_2 >> task_3          # séquentiel
task_1 >> [task_2, task_3] >> task_4  # parallèle puis jointure

# Équivalent explicite
task_2.set_upstream(task_1)
```

---

## 5. Les Operators

### 5.1 Principaux operators

| Operator | Usage |
|----------|-------|
| `BashOperator` | Exécute une commande bash |
| `PythonOperator` | Appelle une fonction Python |
| `DockerOperator` | Lance un conteneur Docker |
| `PostgresOperator` | Exécute du SQL sur Postgres |
| `GCSToBigQueryOperator`* | Charge GCS → BigQuery |
| `LocalFilesystemToGCSOperator`* | Upload local → GCS |
| `BigQueryInsertJobOperator`* | Requête BigQuery |
| `DummyOperator` / `EmptyOperator` | Tâche vide (structure du graphe) |

\* Providers : `apache-airflow-providers-google`

### 5.2 Exemples

```python
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

# Bash
download_task = BashOperator(
    task_id='download_dataset',
    bash_command='curl -sSL {{ params.url }} > {{ params.local_file }}'
)

# Python
def _format_file(**context):
    # logique pandas : parquet -> csv, etc.
    pass

format_task = PythonOperator(
    task_id='format_to_parquet',
    python_callable=_format_file,
)
```

---

## 6. Écrire son premier DAG

### 6.1 Structure type

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'retries': 1,
}

with DAG(
    dag_id='mon_premier_dag',
    default_args=default_args,
    description='DAG de démonstration',
    schedule_interval='@daily',     # ou cron '0 5 * * *'
    start_date=datetime(2021, 1, 1),
    catchup=False,
    tags=['zoomcamp', 'demo'],
) as dag:

    t1 = BashOperator(task_id='tache_1', bash_command='echo "start"')
    t2 = BashOperator(task_id='tache_2', bash_command='sleep 5')
    t3 = BashOperator(task_id='tache_3', bash_command='echo "done"')

    t1 >> t2 >> t3
```

### 6.2 `default_args`

| Argument | Rôle |
|----------|------|
| `owner` | Propriétaire du DAG |
| `depends_on_past` | La tâche attend le succès du run précédent (utile pour l'incrémental) |
| `start_date` | Date de début de planification |
| `retries` | Nombre de tentatives en cas d'échec |
| `retry_delay` | Délai entre tentatives (`timedelta(minutes=5)`) |
| `email_on_failure` | Notification email |

---

## 7. Paramètres du DAG : schedule, catchup, dates

### 7.1 `schedule_interval`

```python
schedule_interval='@daily'        # presets : @hourly @daily @weekly @monthly
schedule_interval='0 2 * * *'     # cron : tous les jours à 2h
schedule_interval=None            # déclenchement manuel uniquement
```

> 🔑 Depuis Airflow 2.4 : `schedule=` remplace `schedule_interval=`.

### 7.2 `start_date`, `execution_date` et intervalles

Concept crucial (et contre-intuitif) : Airflow exécute le run d'une période **à la fin de cette période**.

```
schedule @monthly, start_date = 2021-01-01

execution_date = 2021-01-01  →  run déclenché le 2021-02-01
                               (traite les données de janvier)
```

- `execution_date` = **début** de l'intervalle de données
- Le run se déclenche à `execution_date + schedule_interval`

### 7.3 `catchup`

- `catchup=True` : Airflow **rattrape** tous les intervalles manqués entre `start_date` et aujourd'hui → c'est le mécanisme de **backfill**
- `catchup=False` : seule la dernière période est planifiée

```bash
# Backfill manuel en CLI
docker compose run airflow-cli airflow dags backfill \
  --start-date 2019-01-01 --end-date 2021-01-01 taxi_data_dag
```

---

## 8. Templating Jinja et macros

Airflow utilise **Jinja2** : les champs marqués `template_fields` des operators sont rendus dynamiquement.

### 8.1 Variables essentielles

| Expression Jinja | Signification | Exemple |
|------------------|---------------|---------|
| `{{ ds }}` | execution_date (YYYY-MM-DD) | `2021-01-01` |
| `{{ ds_nodash }}` | Sans tirets | `20210101` |
| `{{ execution_date }}` | Objet datetime | — |
| `{{ next_execution_date }}` | Fin de l'intervalle | — |
| `{{ macros.ds_format(...) }}` | Formatage de date | — |
| `{{ params.xxx }}` | Paramètre custom du DAG | — |
| `{{ var.value.xxx }}` | Variable Airflow (UI/CLI) | — |
| `{{ ti.xcom_pull(...) }}` | XCom d'une autre tâche | — |

### 8.2 Pattern du cours : fichier mensuel dynamique

```python
# URL du fichier du mois correspondant à execution_date
URL_PREFIX = 'https://d37ci6vzurychx.cloudfront.net/trip-data'
URL_TEMPLATE = URL_PREFIX + '/yellow_tripdata_{{ execution_date.strftime("%Y-%m") }}.parquet'
OUTPUT_TEMPLATE = 'output_{{ execution_date.strftime("%Y-%m") }}.parquet'

with DAG(..., schedule_interval='@monthly', ...) as dag:
    download_task = BashOperator(
        task_id='download',
        bash_command=f'curl -sSL {URL_TEMPLATE} > {OUTPUT_TEMPLATE}'
    )
```

> 🔑 Chaque run mensuel télécharge automatiquement le fichier du **bon mois** grâce au templating → c'est le cœur de l'exercice du cours.

---

## 9. Cas pratique local : ingestion NYC Taxi vers Postgres

DAG `data_ingestion_local` (version locale du cours) :

```python
import os
from datetime import datetime
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from ingest_script import ingest_callable   # fonction d'ingestion (Module 1)

AIRFLOW_HOME = os.environ.get('AIRFLOW_HOME', '/opt/airflow/')

PG_HOST = os.getenv('PG_HOST')
PG_USER = os.getenv('PG_USER')
PG_PASSWORD = os.getenv('PG_PASSWORD')
PG_PORT = os.getenv('PG_PORT')
PG_DATABASE = os.getenv('PG_DATABASE')

URL_PREFIX = 'https://d37ci6vzurychx.cloudfront.net/trip-data'
URL_TEMPLATE = URL_PREFIX + '/yellow_tripdata_{{ execution_date.strftime("%Y-%m") }}.parquet'
OUTPUT_FILE_TEMPLATE = AIRFLOW_HOME + '/output_{{ execution_date.strftime("%Y-%m") }}.parquet'
TABLE_NAME_TEMPLATE = 'yellow_taxi_{{ execution_date.strftime("%Y_%m") }}'

with DAG(
    dag_id='local_ingestion_dag',
    schedule_interval='0 6 2 * *',     # le 2 de chaque mois à 6h
    start_date=datetime(2021, 1, 1),
    default_args={'retries': 1},
    catchup=True,
    max_active_runs=3,
) as dag:

    download_dataset_task = BashOperator(
        task_id='download_dataset',
        bash_command=f'curl -sSL {URL_TEMPLATE} > {OUTPUT_FILE_TEMPLATE}'
    )

    ingest_task = PythonOperator(
        task_id='ingest_to_postgres',
        python_callable=ingest_callable,
        op_kwargs=dict(
            user=PG_USER, password=PG_PASSWORD, host=PG_HOST,
            port=PG_PORT, db=PG_DATABASE,
            table_name=TABLE_NAME_TEMPLATE,
            parquet_file=OUTPUT_FILE_TEMPLATE,
        ),
    )

    download_dataset_task >> ingest_task
```

**Pipeline** : `télécharger le parquet du mois → ingestion dans Postgres (table par mois)`

> ⚠️ Le réseau Docker : les conteneurs Airflow doivent être sur le **même réseau** que Postgres (`pg-network`) — joindre Postgres par son nom de conteneur, pas `localhost`.

---

## 10. Cas pratique GCP : ingestion vers GCS et BigQuery

DAG `data_ingestion_gcs_dag` :

```
download ──► format_to_parquet ──► local_to_gcs ──► gcs_to_bq
```

```python
import os
from datetime import datetime
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.providers.google.cloud.operators.gcs import GCSToBigQueryOperator  # selon version
from airflow.providers.google.cloud.transfers.local_to_gcs import LocalFilesystemToGCSOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryCreateExternalTableOperator

PROJECT_ID = os.environ.get('GCP_PROJECT_ID')
BUCKET = os.environ.get('GCP_GCS_BUCKET')
BIGQUERY_DATASET = os.environ.get('BIGQUERY_DATASET', 'trips_data_all')

AIRFLOW_HOME = os.environ.get('AIRFLOW_HOME', '/opt/airflow/')
URL_PREFIX = 'https://d37ci6vzurychx.cloudfront.net/trip-data'

URL_TEMPLATE = URL_PREFIX + '/yellow_tripdata_{{ execution_date.strftime("%Y-%m") }}.parquet'
PARQUET_FILE = AIRFLOW_HOME + '/yellow_{{ execution_date.strftime("%Y-%m") }}.parquet'
PATH_TO_GCS = 'raw/yellow_{{ execution_date.strftime("%Y-%m") }}.parquet'

with DAG(
    dag_id='gcs_ingestion_dag',
    schedule_interval='@monthly',
    start_date=datetime(2019, 1, 1),
    catchup=True,
    max_active_runs=3,
    tags=['zoomcamp', 'gcp'],
) as dag:

    download_dataset_task = BashOperator(
        task_id='download_dataset',
        bash_command=f'curl -sSL {URL_TEMPLATE} > {PARQUET_FILE}'
    )

    local_to_gcs_task = LocalFilesystemToGCSOperator(
        task_id='local_to_gcs',
        src=PARQUET_FILE,
        dst=PATH_TO_GCS,
        bucket=BUCKET,
    )

    gcs_to_bq_task = BigQueryCreateExternalTableOperator(
        task_id='gcs_to_bq',
        table_resource={
            'tableReference': {
                'projectId': PROJECT_ID,
                'datasetId': BIGQUERY_DATASET,
                'tableId': 'yellow_taxi_external',
            },
            'externalDataConfiguration': {
                'sourceFormat': 'PARQUET',
                'sourceUris': [f'gs://{BUCKET}/raw/yellow_*.parquet'],
            },
        },
    )

    download_dataset_task >> local_to_gcs_task >> gcs_to_bq_task
```

> 🔑 Le wildcard `gs://bucket/raw/yellow_*.parquet` dans la **table externe** BigQuery agrège tous les mois uploadés — fait le lien direct avec le **Module 3 (tables externes vs natives)**.

---

## 11. Transfers GCS → BigQuery

Pattern EL complet du cours :

```
CSV/Parquet (web) ──► local ──► GCS (data lake) ──► BigQuery (data warehouse)
```

| Operator | Sens |
|----------|------|
| `LocalFilesystemToGCSOperator` | local → GCS |
| `GCSToBigQueryOperator` | GCS → table native BigQuery |
| `BigQueryCreateExternalTableOperator` | GCS → table **externe** BigQuery |
| `BigQueryInsertJobOperator` | requête SQL dans BigQuery (transformation, CTAS) |

> 💡 Rappel Module 3 : table **externe** = données restent sur GCS ; table **native** = données copiées dans le stockage géré BigQuery (meilleures performances, partitionnement/clustering possibles).

---

## 12. XComs : partage de données entre tâches

- **XCom (cross-communication)** : petit stockage clé-valeur pour passer des infos entre tâches
- Stocké dans la **metadata DB** → ⚠️ réservé aux **petites données** (chemins de fichiers, compteurs), jamais les datasets !

```python
# Push (implicite via return ou explicite)
def _extract(**context):
    context['ti'].xcom_push(key='file_path', value='/tmp/data.parquet')

# Pull
def _load(**context):
    path = context['ti'].xcom_pull(task_ids='extract_task', key='file_path')

# Dans un BashOperator (templating)
bash_command='echo "{{ ti.xcom_pull(task_ids=\'extract_task\') }}"'
```

> 🔑 Pour les gros volumes : passer un **chemin** (GCS, fichier) via XCom, jamais les données elles-mêmes.

---

## 13. Bonnes pratiques

### ✅ À faire

- **Idempotence** : un run rejoué doit produire le même résultat (supprimer/écraser avant d'insérer)
- **Tâches atomiques** : une tâche = une responsabilité
- Paramétrer avec des **variables d'environnement** / Variables Airflow, jamais de secrets en dur
- `catchup` et `depends_on_past` choisis **consciemment**
- `max_active_runs` limité pour ne pas saturer le worker lors des backfills
- Templating Jinja pour tout ce qui dépend de la date (URLs, noms de fichiers, tables)
- Tags et descriptions sur les DAGs

### ❌ À éviter

- Traiter de gros volumes **dans** les tâches Airflow (XCom, PythonOperator lourds) → déléguer (GCS, BigQuery, Spark)
- `datetime.now()` dans le code du DAG (utiliser `execution_date` / `{{ ds }}`)
- Logique lourde au **top-level** du fichier DAG (ré-exécutée à chaque parse par le scheduler)
- Credentials en clair dans le code
- `catchup=True` + `start_date` ancien sans prévoir la charge du backfill

---

## 14. Aide-mémoire

### 14.1 Commandes CLI (dans le conteneur)

```bash
# Initialisation et lancement
docker compose up airflow-init
docker compose up -d

# Lister / déclencher des DAGs
docker compose run airflow-cli airflow dags list
docker compose run airflow-cli airflow dags trigger <dag_id>
docker compose run airflow-cli airflow tasks list <dag_id>

# État des tâches / logs
docker compose run airflow-cli airflow tasks state <dag_id> <task_id> <execution_date>

# Backfill
docker compose run airflow-cli airflow dags backfill \
  --start-date 2019-01-01 --end-date 2021-01-01 <dag_id>

# Tester une tâche isolément
docker compose run airflow-cli airflow tasks test <dag_id> <task_id> 2021-01-01

# Variables
docker compose run airflow-cli airflow variables set MA_CLE ma_valeur
```

### 14.2 Cron rapide

```
┌ minute (0-59)
│ ┌ heure (0-23)
│ │ ┌ jour du mois (1-31)
│ │ │ ┌ mois (1-12)
│ │ │ │ ┌ jour de semaine (0-6, dim=0)
* * * * *

'0 6 2 * *'   → le 2 de chaque mois à 6h00
'0 0 * * 1'   → chaque lundi à minuit
```

### 14.3 Jinja essentiel

```
{{ ds }}                                    → 2021-01-01
{{ execution_date.strftime("%Y-%m") }}      → 2021-01
{{ params.x }} / {{ var.value.x }}          → paramètre / variable
{{ ti.xcom_pull(task_ids='t1') }}           → XCom
```

### 14.4 UI Airflow — vues utiles

| Vue | Usage |
|-----|-------|
| **Grid/Tree** | État des runs et tâches par date |
| **Graph** | Visualisation du DAG |
| **Calendar** | Historique par jour |
| **Gantt** | Durées des tâches, goulets |
| **Code** | Source du DAG parsé |
| **Logs** | Par task instance (debug) |

---

## ❓ Questions de révision

1. **Quelle est la différence entre `execution_date` et la date réelle d'exécution ?**
   → `execution_date` est le **début de l'intervalle de données** traité ; le run se déclenche à la **fin** de cet intervalle (ex. schedule mensuel : le run de janvier part le 1er février).

2. **À quoi sert `catchup=True` ?**
   → À rattraper automatiquement tous les intervalles passés depuis `start_date` → mécanisme de **backfill** intégré.

3. **Comment un run mensuel sait-il quel fichier télécharger ?**
   → Via le **templating Jinja** : `{{ execution_date.strftime("%Y-%m") }}` dans l'URL, le nom de fichier et le nom de table.

4. **Pourquoi ne pas passer un DataFrame via XCom ?**
   → Les XComs sont stockés dans la metadata DB : réservés aux petites valeurs. Pour la donnée volumineuse, passer un **chemin** (fichier/GCS).

5. **Comment rejouer 2 ans d'historique ?**
   → `airflow dags backfill --start-date ... --end-date ... <dag_id>`, ou `catchup=True` avec une `start_date` ancienne.

6. **Quels composants Airflow sont indispensables au minimum ?**
   → Scheduler, Webserver (optionnel mais standard), Metadata DB, et un executor (LocalExecutor en mono-machine).

---

> 📚 **Ressources** :
> - [Repo du cours (branches historiques 2022/2023) — Module 2 Airflow](https://github.com/DataTalksClub/data-engineering-zoomcamp)
> - [Documentation Apache Airflow](https://airflow.apache.org/docs/)
> - [Airflow Docker Compose officiel](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)
> - [Macros Jinja Airflow](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html)
> - 🆕 Version actuelle du module : voir la fiche **Module 2 — Kestra**
