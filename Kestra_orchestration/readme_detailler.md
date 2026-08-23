Voici la fiche de révision pour le **Module 2 — Workflow Orchestration avec Kestra**, au format Markdown prêt pour GitHub.

> ⚠️ **Note** : dans les versions récentes du Zoomcamp (2024+), le Module 2 utilise **Kestra** (les anciennes versions utilisaient Airflow). Cette fiche couvre Kestra + la section **dlt** (data load tool). Vérifiez la syntaxe avec la [doc officielle Kestra](https://kestra.io/docs) et le repo du cours.

---


# Module 2 — Workflow Orchestration avec Kestra

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision du Module 2 : Orchestration de pipelines avec Kestra (+ dlt)

---

## 📑 Table des matières

1. [Introduction à l'orchestration de workflows](#1-introduction-à-lorchestration-de-workflows)
2. [Présentation de Kestra](#2-présentation-de-kestra)
3. [Installation avec Docker](#3-installation-avec-docker)
4. [Concepts fondamentaux](#4-concepts-fondamentaux)
5. [Anatomie d'un flow YAML](#5-anatomie-dun-flow-yaml)
6. [Les Tasks](#6-les-tasks)
7. [Inputs, Outputs et Variables](#7-inputs-outputs-et-variables)
8. [Expressions et templating (Pebble)](#8-expressions-et-templating-pebble)
9. [Triggers et planification](#9-triggers-et-planification)
10. [Backfill (rejeu historique)](#10-backfill-rejeu-historique)
11. [Orchestrer plusieurs flows](#11-orchestrer-plusieurs-flows)
12. [Cas pratique : pipeline NYC Taxi vers Postgres et GCS/BigQuery](#12-cas-pratique--pipeline-nyc-taxi)
13. [dlt — data load tool](#13-dlt--data-load-tool)
14. [Kestra vs Airflow](#14-kestra-vs-airflow)
15. [Bonnes pratiques](#15-bonnes-pratiques)
16. [Aide-mémoire](#16-aide-mémoire)

---

## 1. Introduction à l'orchestration de workflows

### 1.1 Pourquoi orchestrer ?

Un pipeline de données = **plusieurs étapes dépendantes** :

```
Extraction ─► Validation ─► Chargement ─► Transformation ─► Notification
```

Sans orchestrateur : scripts cron éparpillés, pas de visibilité, gestion d'erreurs manuelle.

### 1.2 Rôle d'un orchestrateur

| Fonctionnalité | Description |
|----------------|-------------|
| **Planification** | Exécution selon un schedule (cron) |
| **Dépendances** | Les tâches s'enchaînent dans le bon ordre (DAG) |
| **Retry** | Ré-exécution automatique en cas d'échec |
| **Monitoring** | UI, logs, métriques, alertes |
| **Paramétrage** | Dates, environnements, inputs dynamiques |
| **Backfill** | Rejouer l'historique sur une période passée |

### 1.3 Le concept de DAG

- **Directed Acyclic Graph** : graphe de tâches **orienté** et **sans cycle**
- Chaque nœud = une tâche ; chaque arête = une dépendance
- Un cycle rendrait l'exécution impossible (A attend B qui attend A...)

---

## 2. Présentation de Kestra

### 2.1 Qu'est-ce que Kestra ?

- Orchestrateur **open-source**, **déclaratif** et **event-driven**
- Les pipelines (flows) sont définis en **YAML** (pas en Python comme Airflow)
- **Language-agnostic** : exécute Python, SQL, Bash, Docker, dbt...
- UI web intégrée : édition, exécution, monitoring, logs en temps réel
- Plus de **800 plugins** : GCP, AWS, Snowflake, dbt, Spark, Slack...

### 2.2 Points forts de Kestra

- ✅ Flows en YAML versionnables (Git)
- ✅ UI avec **éditeur de code intégré** (autocomplétion, documentation)
- ✅ Exécution **événementielle** (déclenchement par fichier, message, API...)
- ✅ Backfill natif
- ✅ Pas besoin d'être développeur Python pour écrire un flow

### 2.3 Architecture

```
┌─────────────┐   ┌──────────────┐   ┌─────────────┐
│  Webserver  │   │   Executor   │   │   Worker(s) │
│  (UI + API) │──►│ (ordonnance) │──►│ (exécutent  │
└─────────────┘   └──────────────┘   │  les tasks) │
                                     └─────────────┘
              Base de données (Postgres en interne)
```

En mode **standalone local** (utilisé dans le cours) : tout tourne dans un seul conteneur Docker.

---

## 3. Installation avec Docker

### 3.1 Lancement rapide

```bash
docker run --pull=always --rm -it \
  -p 8080:8080 \
  --user=root \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp:/tmp \
  kestra/kestra:latest server local
```

- UI accessible sur **http://localhost:8080**
- Le montage du **socket Docker** permet à Kestra de lancer des conteneurs (tasks Docker)

### 3.2 Avec docker-compose (version du cours)

```yaml
# docker-compose.yml (simplifié)
volumes:
  kestra-data:

services:
  kestra:
    image: kestra/kestra:latest
    command: server local
    user: root
    ports:
      - "8080:8080"
    volumes:
      - kestra-data:/app/storage
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp:/tmp
```

```bash
docker compose up -d
```

> 💡 Dans le cours, Kestra tourne souvent **à côté** d'un conteneur Postgres (destination des données taxi) sur le même réseau Docker.

---

## 4. Concepts fondamentaux

### 4.1 Hiérarchie des objets

```
Flow (pipeline)
 └── Tasks (étapes)
      └── chaque task peut produire des outputs
Triggers (déclencheurs : schedule, webhook, flow...)
Inputs (paramètres du flow)
```

| Concept | Description |
|---------|-------------|
| **Flow** | Un pipeline complet, défini en YAML |
| **Namespace** | Espace de nommage / dossier logique (ex. `zoomcamp`) |
| **Task** | Une unité de travail (script, requête, transfert...) |
| **Execution** | Une instance d'exécution d'un flow |
| **Trigger** | Condition de déclenchement (schedule, événement) |
| **Input** | Paramètre passé au flow (typé : STRING, DATE, INT...) |
| **Output** | Valeur produite par une task, réutilisable ensuite |
| **Label** | Métadonnées clé-valeur (organisation, filtrage) |

### 4.2 Types de tasks

- **`io.kestra.plugin.core.flow.*`** : logique de flow (Sequential, Parallel, If, Switch, Subflow, ForEach)
- **Tasks de scripts** : Python, Bash, Node, SQL...
- **Tasks de plugins** : GCS, BigQuery, S3, dbt, Slack, HTTP...

---

## 5. Anatomie d'un flow YAML

```yaml
id: 01_getting_started        # identifiant unique du flow
namespace: zoomcamp           # espace de nommage
description: Mon premier flow

labels:
  env: dev
  course: zoomcamp

inputs:
  - id: user
    type: STRING
    defaults: world

tasks:
  - id: say_hello
    type: io.kestra.plugin.scripts.python.Commands
    commands:
      - echo "Hello {{ inputs.user }} !"   # expression Pebble

  - id: log_execution_date
    type: io.kestra.plugin.core.log.Log
    message: "Exécuté le {{ execution.startDate }}"
```

**Champs obligatoires** : `id`, `namespace`, `tasks`.

> 💡 Les tasks s'exécutent **séquentiellement par défaut** dans l'ordre de déclaration.

---

## 6. Les Tasks

### 6.1 Exécuter du Python

```yaml
tasks:
  - id: transform
    type: io.kestra.plugin.scripts.python.Script
    containerImage: python:3.11-slim    # image Docker d'exécution
    beforeCommands:
      - pip install pandas requests
    script: |
      import pandas as pd
      df = pd.read_csv("https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz")
      print(df.shape)
```

### 6.2 Exécuter du Bash

```yaml
  - id: download
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - curl -o data.csv.gz "{{ inputs.file_url }}"
      - gunzip data.csv.gz
```

### 6.3 Logique de flow

```yaml
tasks:
  # Parallélisme
  - id: parallel_tasks
    type: io.kestra.plugin.core.flow.Parallel
    tasks:
      - id: task_a
        type: io.kestra.plugin.core.log.Log
        message: "A"
      - id: task_b
        type: io.kestra.plugin.core.log.Log
        message: "B"

  # Condition
  - id: check
    type: io.kestra.plugin.core.flow.If
    condition: "{{ inputs.env == 'prod' }}"
    then:
      - id: prod_task
        type: io.kestra.plugin.core.log.Log
        message: "Production !"
    else:
      - id: dev_task
        type: io.kestra.plugin.core.log.Log
        message: "Dev"

  # Boucle
  - id: each_month
    type: io.kestra.plugin.core.flow.ForEach
    values: ["01", "02", "03"]
    tasks:
      - id: process_month
        type: io.kestra.plugin.core.log.Log
        message: "Mois {{ taskrun.value }}"
```

### 6.4 Retry et gestion d'erreurs

```yaml
  - id: fragile_task
    type: io.kestra.plugin.scripts.shell.Commands
    retry:
      type: constant          # constant | exponential | random
      interval: PT1M          # toutes les minutes
      maxAttempt: 5
      maxDuration: PT30M
    timeout: PT10M            # durée max d'exécution
    commands:
      - python script_qui_peut_echouer.py

  # Task exécutée en cas d'échec global
errors:
  - id: alert_on_failure
    type: io.kestra.plugin.core.log.Log
    message: "❌ Le flow a échoué !"
```

---

## 7. Inputs, Outputs et Variables

### 7.1 Déclarer des inputs

```yaml
inputs:
  - id: taxi_type
    type: SELECT
    defaults: yellow
    values: [yellow, green]

  - id: year
    type: STRING
    defaults: "2021"

  - id: month
    type: STRING
    defaults: "01"

  - id: run_date
    type: DATE
    required: false
```

Accès dans les tasks : `{{ inputs.taxi_type }}`, `{{ inputs.year }}`.

### 7.2 Produire et consommer des outputs

```yaml
tasks:
  - id: extract
    type: io.kestra.plugin.scripts.python.Script
    containerImage: python:3.11-slim
    outputFiles:
      - "clean_data.csv"
    script: |
      import pandas as pd
      df = pd.read_csv("raw.csv")
      df.dropna().to_csv("clean_data.csv", index=False)

  - id: load
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - wc -l {{ outputs.extract.outputFiles['clean_data.csv'] }}
```

### 7.3 Variables internes utiles

| Expression | Contenu |
|-----------|---------|
| `{{ execution.id }}` | ID de l'exécution |
| `{{ execution.startDate }}` | Date de démarrage |
| `{{ flow.id }}` / `{{ flow.namespace }}` | Métadonnées du flow |
| `{{ taskrun.value }}` | Valeur courante dans un ForEach |
| `{{ outputs.task_id.champ }}` | Output d'une task précédente |
| `{{ trigger.date }}` | Date du déclencheur (schedule) |

---

## 8. Expressions et templating (Pebble)

Kestra utilise le moteur de templates **Pebble** (syntaxe proche de Jinja2) :

```yaml
message: "{{ inputs.year }}-{{ inputs.month }}"          # concaténation
message: "{{ execution.startDate | date('yyyy-MM-dd') }}" # filtre date
message: "{{ 'yellow' if inputs.taxi == 'y' else 'green' }}" # conditionnelle
```

### Filtres utiles

| Filtre | Exemple | Résultat |
|--------|---------|----------|
| `date(format)` | `{{ now() | date('yyyy-MM-dd') }}` | `2026-08-23` |
| `dateAdd` | `{{ trigger.date | dateAdd(-1, 'MONTHS') }}` | mois précédent |
| `upper` / `lower` | `{{ inputs.x | upper }}` | majuscules |
| `default` | `{{ inputs.y | default('2021') }}` | valeur par défaut |
| `first` / `last` | sur une liste | premier/dernier élément |

---

## 9. Triggers et planification

### 9.1 Trigger Schedule (cron)

```yaml
triggers:
  - id: monthly
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 1 * *"        # le 1er de chaque mois à 9h
    # ou expressions lisibles :
    # cron: "@daily" / "@hourly" / "@weekly"
```

> 💡 **Syntaxe cron** : `minute heure jour-du-mois mois jour-de-semaine`

### 9.2 Trigger Flow (chaînage)

```yaml
triggers:
  - id: after_parent
    type: io.kestra.plugin.core.trigger.Flow
    preconditions:
      id: flows
      flows:
        - namespace: zoomcamp
          flowId: 02_gcp_taxi
          states: [SUCCESS]
```

### 9.3 Autres triggers

- **Webhook** : déclenchement via appel HTTP
- **File detection** : nouveau fichier dans GCS/S3
- **Schedule avec backfill** : rejouer des dates passées

---

## 10. Backfill (rejeu historique)

### 10.1 Principe

- Rejouer un flow planifié sur une **période passée**
- Cas typique du cours : ingérer les données taxi **mois par mois** sur plusieurs années

### 10.2 Dans l'UI

1. Ouvrir le flow → onglet **Triggers**
2. Cliquer sur **Backfill executions**
3. Définir `start` et `end`
4. Kestra crée une exécution par intervalle du schedule

### 10.3 Pattern du cours

```yaml
triggers:
  - id: monthly_backfill
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 0 1 * *"        # mensuel

# Dans les tasks, on utilise la date du trigger :
tasks:
  - id: set_labels
    type: io.kestra.plugin.core.execution.Labels
    labels:
      year: "{{ trigger.date | date('yyyy') }}"
      month: "{{ trigger.date | dateAdd(-1, 'MONTHS') | date('MM') }}"
```

> ⚠️ Le backfill nécessite souvent un trigger **disabled par défaut** pour éviter les doubles exécutions (`disabled: true` puis activation manuelle).

---

## 11. Orchestrer plusieurs flows

### 11.1 Subflow : appeler un flow depuis un autre

```yaml
  - id: run_child
    type: io.kestra.plugin.core.flow.Subflow
    namespace: zoomcamp
    flowId: 03_postgres_dbt
    wait: true               # attendre la fin
    transmitFailed: true     # propager l'échec
    inputs:
      taxi_type: yellow
```

### 11.2 Exécuter un flow par API

```bash
curl -X POST "http://localhost:8080/api/v1/executions/zoomcamp/01_flow" \
  -F "inputs.year=2021" \
  -F "inputs.month=01"
```

---

## 12. Cas pratique : pipeline NYC Taxi

### 12.1 Pipeline Postgres (flow local)

```yaml
id: 02_postgres_taxi
namespace: zoomcamp

inputs:
  - id: taxi
    type: SELECT
    values: [yellow, green]
    defaults: yellow
  - id: year
    type: STRING
    defaults: "2021"
  - id: month
    type: STRING
    defaults: "01"

variables:
  file: "{{ inputs.taxi }}_tripdata_{{ inputs.year }}-{{ inputs.month }}.csv"
  staging_table: "public.{{ inputs.taxi }}_tripdata_staging"
  data: "{{ outputs.extract.outputFiles[inputs.taxi ~ '_tripdata_' ~ inputs.year ~ '-' ~ inputs.month ~ '.csv'] }}"

tasks:
  - id: extract
    type: io.kestra.plugin.scripts.shell.Commands
    outputFiles:
      - "*.csv"
    commands:
      - curl -sSL "https://github.com/DataTalksClub/nyc-tlc-data/releases/download/{{ inputs.taxi }}/{{ vars.file }}.gz" | gunzip > {{ vars.file }}

  - id: create_table
    type: io.kestra.plugin.jdbc.postgresql.Queries
    # création de la table si inexistante (DDL)
    sql: CREATE TABLE IF NOT EXISTS ...

  - id: copy_in
    type: io.kestra.plugin.jdbc.postgresql.CopyIn
    format: CSV
    from: "{{ vars.data }}"
    table: "{{ vars.staging_table }}"
    header: true

  - id: merge_data
    type: io.kestra.plugin.jdbc.postgresql.Queries
    sql: INSERT INTO table_finale SELECT * FROM staging ON CONFLICT DO NOTHING;

  - id: purge_files
    type: io.kestra.plugin.core.storage.PurgeCurrentExecutionFiles
```

### 12.2 Pipeline GCP (GCS + BigQuery)

```yaml
id: 03_gcp_taxi
namespace: zoomcamp

tasks:
  - id: extract
    # ... téléchargement CSV (comme ci-dessus)

  - id: upload_to_gcs
    type: io.kestra.plugin.gcp.gcs.Upload
    from: "{{ vars.data }}"
    to: "gs://mon-bucket/{{ vars.file }}"
    serviceAccount: "{{ secret('GCP_SERVICE_ACCOUNT') }}"

  - id: bq_create_external_table
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: |
      CREATE OR REPLACE EXTERNAL TABLE projet.dataset.{{ inputs.taxi }}_external
      OPTIONS (format = 'CSV', uris = ['gs://mon-bucket/{{ vars.file }}']);

  - id: bq_create_table
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: |
      CREATE OR REPLACE TABLE projet.dataset.{{ inputs.taxi }}_tripdata
      PARTITION BY DATE(tpep_pickup_datetime)
      AS SELECT * FROM projet.dataset.{{ inputs.taxi }}_external;
```

> 🔗 Ce flow fait le lien avec le **Module 3** : table externe → table native partitionnée dans BigQuery !

### 12.3 Secrets

```yaml
serviceAccount: "{{ secret('GCP_SERVICE_ACCOUNT') }}"
```

Les secrets se définissent dans l'UI (**Namespaces → Secrets**) ou via variables d'environnement (`SECRET_GCP_SERVICE_ACCOUNT` encodé en base64).

---

## 13. dlt — data load tool

### 13.1 Qu'est-ce que dlt ?

- Bibliothèque **Python open-source** d'ingestion (le "L" de ELT)
- Extrait des données d'**APIs, fichiers, bases** → charge dans un **warehouse**
- Gère automatiquement : schéma, normalisation, incrémental, state

```bash
pip install dlt[bigquery]
```

### 13.2 Exemple minimal

```python
import dlt

@dlt.resource(write_disposition="replace")
def taxi_data():
    # générateur qui yield les enregistrements
    yield from fetch_api_data()

pipeline = dlt.pipeline(
    pipeline_name="taxi_pipeline",
    destination="bigquery",
    dataset_name="nytaxi"
)

load_info = pipeline.run(taxi_data())
print(load_info)
```

### 13.3 Concepts clés

| Concept | Description |
|---------|-------------|
| **Source** | Ensemble de resources (ex. une API complète) |
| **Resource** | Un endpoint / une table de données |
| **write_disposition** | `replace`, `append`, `merge` (incrémental) |
| **Incremental** | `dlt.sources.incremental("updated_at")` |
| **State** | Mémorisation du dernier point chargé |

### 13.4 dlt + Kestra

dlt s'intègre dans Kestra via une task de **script Python** — parfait pour la partie ingestion d'un pipeline orchestré.

---

## 14. Kestra vs Airflow

| Critère | Kestra | Airflow |
|---------|--------|---------|
| **Définition des pipelines** | YAML déclaratif | Python impératif |
| **Courbe d'apprentissage** | Douce | Plus raide |
| **UI** | Moderne, édition intégrée | Historique, lecture surtout |
| **Event-driven** | ✅ Natif | ⚠️ Possible (sensors, deferrable) |
| **Langages des tasks** | Tous (via scripts/conteneurs) | Python principalement |
| **Backfill** | Natif, via UI | CLI / API |
| **Maturité / écosystème** | Jeune mais croissance rapide | Standard de l'industrie |

---

## 15. Bonnes pratiques

- ✅ **Versionner les flows** dans Git (YAML = facile à reviewer)
- ✅ Utiliser des **inputs typés** plutôt que des valeurs en dur
- ✅ Nommer les tasks avec des `id` explicites (`extract_yellow_2021`)
- ✅ Prévoir les **retries** sur les tasks réseau (téléchargements, API)
- ✅ **Purger les fichiers temporaires** en fin de flow
- ✅ Utiliser les **secrets** pour les credentials (jamais en clair !)
- ✅ Labels pour tracer les exécutions (env, taxi_type, année...)
- ✅ Tables **staging** + merge plutôt qu'insert direct
- ✅ Un flow par responsabilité + orchestration via subflows/triggers

---

## 16. Aide-mémoire

### Commandes Docker

| Commande | Description |
|----------|-------------|
| `docker compose up -d` | Lancer Kestra |
| `docker compose logs -f kestra` | Suivre les logs |
| `docker compose down` | Arrêter |

### Types d'inputs courants

`STRING` · `INT` · `FLOAT` · `BOOL` · `DATE` · `DATETIME` · `DURATION` · `FILE` · `JSON` · `SELECT` · `MULTISELECT` · `SECRET`

### Plugins JDBC utiles

`io.kestra.plugin.jdbc.postgresql.Query` / `.Queries` / `.CopyIn`
`io.kestra.plugin.gcp.gcs.Upload` · `io.kestra.plugin.gcp.bigquery.Query`

### Pattern d'expressions courantes

```yaml
"{{ inputs.year }}-{{ inputs.month }}"                       # paramètres
"{{ trigger.date | dateAdd(-1, 'MONTHS') | date('yyyy-MM') }}" # backfill mensuel
"{{ outputs.task_id.outputFiles['fichier.csv'] }}"           # fichier d'une task
"{{ secret('MON_SECRET') }}"                                  # secret
```

---

## ❓ Questions de révision

1. **Quelle est la différence entre un flow, une task et une exécution ?**
   → Flow = définition YAML ; task = étape du flow ; exécution = instance concrète d'un run avec ses logs et outputs.

2. **Comment passer un paramètre à un flow Kestra ?**
   → Via les `inputs` (typés), accessibles avec `{{ inputs.nom }}` dans les tasks.

3. **Comment rejouer un pipeline mensuel sur 2 ans d'historique ?**
   → Trigger `Schedule` mensuel + **backfill** (UI : Triggers → Backfill executions) avec `start`/`end`, en utilisant `{{ trigger.date }}` dans les tasks.

4. **Pourquoi Kestra monte-t-il le socket Docker (`/var/run/docker.sock`) ?**
   → Pour permettre aux tasks de lancer des **conteneurs Docker** (scripts Python/Bash isolés avec leurs dépendances).

5. **Comment sécuriser une clé de service GCP dans un flow ?**
   → La stocker comme **secret** Kestra et la référencer avec `{{ secret('GCP_SERVICE_ACCOUNT') }}`.

6. **Différence entre `write_disposition: append` et `merge` dans dlt ?**
   → `append` ajoute toutes les lignes (doublons possibles) ; `merge` fait un upsert basé sur une clé (idempotent, adapté à l'incrémental).

---

> 📚 **Ressources** :
> - [Repo GitHub du Zoomcamp — Module 2](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/02-workflow-orchestration)
> - [Documentation Kestra](https://kestra.io/docs)
> - [Kestra — Plugin Registry](https://kestra.io/plugins)
> - [Documentation dlt](https://dlthub.com/docs)

---

## ✅ Récapitulatif

Cette fiche couvre l'intégralité du **Module 2 (Workflow Orchestration)** :

- **Fondamentaux** : pourquoi orchestrer, DAG, scheduling
- **Kestra** : installation Docker, flows YAML, tasks, inputs/outputs, expressions Pebble
- **Triggers & backfill** : le point clé du homework (rejeu mensuel des données taxi)
- **Cas pratiques** : pipelines NYC Taxi vers **Postgres** et **GCP (GCS + BigQuery)** — faisant le lien avec le Module 3
- **dlt** : l'outil d'ingestion Python couvert dans le module
- **Comparaison Airflow** + **bonnes pratiques** + **questions de révision**

> 💡 Copiez ce contenu dans `module2-workflow-orchestration-kestra.md`. Conseil pratique : testez le backfill mensuel sur les données taxi 2019-2021, c'est l'exercice central du module !

Votre collection grandit : **Modules 2, 3, 4, 5, 6 + Bruin** ✅. Il ne manque plus que le **Module 1 (Docker, Terraform, GCP)** — voulez-vous que je le rédige pour compléter la série ? 🚀
