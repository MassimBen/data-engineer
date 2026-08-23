# Module 4 — Analytics Engineering

> **Fiche de révision en français** — Data Engineering Zoomcamp · DataTalksClub  
> Sujet central : transformer des données chargées dans un entrepôt en vues analytiques fiables avec **dbt**.

## Table des matières

- [1. Objectifs et vue d’ensemble](#1-objectifs-et-vue-densemble)
- [2. Analytics Engineering](#2-analytics-engineering)
  - [Définition et rôle](#définition-et-rôle)
  - [Positionnement des métiers](#positionnement-des-métiers)
  - [Data stack moderne](#data-stack-moderne)
- [3. ETL versus ELT](#3-etl-versus-elt)
- [4. Modélisation dimensionnelle de Kimball](#4-modélisation-dimensionnelle-de-kimball)
  - [Grain](#grain)
  - [Faits et dimensions](#faits-et-dimensions)
  - [Architecture en couches](#architecture-en-couches)
- [5. dbt : principes fondamentaux](#5-dbt--principes-fondamentaux)
  - [dbt Core et dbt Cloud](#dbt-core-et-dbt-cloud)
  - [Fonctionnement](#fonctionnement)
  - [Matérialisations](#matérialisations)
- [6. Installer et configurer dbt](#6-installer-et-configurer-dbt)
  - [BigQuery](#bigquery)
  - [PostgreSQL](#postgresql-alternative)
- [7. Structure d’un projet dbt](#7-structure-dun-projet-dbt)
- [8. Modèles, sources et seeds](#8-modèles-sources-et-seeds)
- [9. Tests et qualité](#9-tests-et-qualité)
- [10. Documentation, macros et packages](#10-documentation-macros-et-packages)
- [11. Cas pratique : NYC Taxi](#11-cas-pratique--nyc-taxi)
- [12. Déploiement et CI/CD](#12-déploiement-et-cicd)
- [13. Visualisation dans Looker Studio](#13-visualisation-dans-looker-studio)
- [14. Commandes essentielles](#14-commandes-essentielles)
- [15. Bonnes pratiques](#15-bonnes-pratiques)
- [16. FAQ et astuces](#16-faq-et-astuces)

---

## 1. Objectifs et vue d’ensemble

Le module 4 part du résultat du module Data Warehouse : les données **yellow taxi** et **green taxi** de New York, couvrant notamment **2019–2020**, sont chargées dans BigQuery. L’objectif est de les rendre utiles aux analystes : données nettoyées, modèle compréhensible, tests automatisés, documentation et tableaux de bord.

Le fil conducteur est :

```text
Sources brutes → entrepôt (BigQuery) → staging dbt → modèles intermédiaires
              → marts dimensionnels/factuels → BI (Looker Studio)
```

Le module officiel propose une piste cloud (**BigQuery + dbt Cloud**) et une piste locale (**DuckDB + dbt Core**). Les principes dbt restent les mêmes : le moteur SQL exécute les requêtes ; dbt organise les transformations, les dépendances, les tests et la documentation.

### À retenir

- dbt **ne déplace pas** généralement les données depuis les systèmes sources : il transforme des données déjà accessibles dans le warehouse.
- Un modèle dbt est une requête SQL versionnée et reproductible.
- `ref()` construit le graphe de dépendances et rend les modèles portables.
- La qualité n’est pas une étape finale : elle est codée dans le projet.

---

## 2. Analytics Engineering

### Définition et rôle

L’**Analytics Engineering** est la discipline qui transforme les données brutes en datasets analytiques fiables, cohérents et faciles à consommer. Elle applique des pratiques de développement logiciel au travail de transformation : Git, revue de code, tests, documentation, environnements et déploiement automatisé.

L’Analytics Engineer :

1. comprend les besoins métier et définit les métriques ;
2. nettoie et standardise les données ;
3. conçoit un modèle analytique (souvent dimensions + faits) ;
4. écrit les transformations SQL/Jinja dans dbt ;
5. met en place tests, documentation et observabilité ;
6. publie des tables ou vues prêtes pour la BI.

Il ne remplace ni le Data Engineer ni le Data Analyst : il crée une interface robuste entre les deux.

### Positionnement des métiers

| Rôle | Responsabilité principale | Exemples de livrables |
|---|---|---|
| **Data Engineer** | Ingestion, transport, stockage, orchestration et fiabilité de la plateforme | pipelines, jobs, tables brutes, sécurité, infrastructure |
| **Analytics Engineer** | Transformation et modélisation des données dans le warehouse | modèles dbt, marts, tests, métriques, documentation |
| **Data Analyst** | Exploration, interprétation et communication des résultats | analyses, rapports, dashboards, recommandations |
| **Data Scientist** | Modèles statistiques/prédictifs et expérimentation | features, modèles ML, évaluations |

La frontière varie selon l’organisation. Dans une petite équipe, une personne peut porter plusieurs casquettes.

### Data stack moderne

| Brique | Fonction | Exemples |
|---|---|---|
| **Sources opérationnelles** | Applications et systèmes qui produisent la donnée | PostgreSQL, API, SaaS, fichiers CSV |
| **Ingestion/EL** | Extraire puis charger dans le stockage analytique | scripts, Airbyte, Fivetran |
| **Data lake** | Stockage massif, souvent peu structuré, économique | GCS, S3, Azure Data Lake |
| **Data warehouse (DWH)** | Requêtes analytiques SQL, gouvernance et partage | BigQuery, Snowflake, Redshift, PostgreSQL |
| **Transformation/T** | Nettoyage, jointures, agrégations et modèle métier | dbt, SQL, Spark |
| **Orchestration** | Ordonnancement et dépendances des jobs | Airflow, Dagster, Prefect |
| **BI** | Exploration et visualisation | Looker Studio, Looker, Metabase, Tableau |
| **Catalogue/qualité** | Documentation, lineage, contrôles | dbt docs, catalogues, tests |

Un **lakehouse** combine certains avantages du data lake et du warehouse ; le choix dépend du volume, du coût, de la latence et des usages.

---

## 3. ETL versus ELT

### ETL

**ETL** signifie *Extract, Transform, Load* :

1. extraire les données de la source ;
2. les transformer dans un moteur ou une zone intermédiaire ;
3. charger le résultat dans le warehouse.

Avantages : données déjà nettoyées à l’arrivée, contrôle du schéma en amont. Limites : pipeline moins flexible, transformations parfois éloignées de l’entrepôt et calcul à dimensionner avant le chargement.

### ELT

**ELT** signifie *Extract, Load, Transform* :

1. extraire ;
2. charger les données brutes dans le lake/warehouse ;
3. transformer avec la puissance de calcul du warehouse.

Avantages : conservation du brut, itérations rapides en SQL, séparation claire ingestion/transformation, possibilité de reconstruire les modèles. Limites : coût des requêtes, gouvernance indispensable et risque de laisser s’accumuler des données inutilisées.

| Critère | ETL | ELT |
|---|---|---|
| Transformation | Avant le warehouse | Dans le warehouse |
| Données brutes conservées | Pas toujours | Oui, idéalement |
| Flexibilité analytique | Plus faible | Forte |
| Outil typique du module | — | dbt après chargement BigQuery |

dbt est principalement un outil de **T de l’ELT**. Il ne remplace pas nécessairement l’ingestion ni l’orchestrateur.

---

## 4. Modélisation dimensionnelle de Kimball

La modélisation de **Ralph Kimball** organise l’entrepôt autour de processus métier et de schémas en étoile. Elle favorise des requêtes simples et compréhensibles par la BI.

### Grain

Le **grain** est la phrase qui décrit ce que représente **une ligne**. Exemples :

- une ligne de `fact_trips` = un trajet taxi ;
- une ligne de `dim_zones` = une zone TLC ;
- une ligne d’une agrégation journalière = un couple date × zone.

Toujours déclarer le grain avant de choisir les colonnes. Mélanger deux grains dans une même table provoque doublons, sommes incorrectes et jointures dangereuses.

### Faits et dimensions

Une **table de faits** contient des événements mesurables, des clés étrangères vers les dimensions et des mesures numériques : distance, montant, durée, nombre de trajets.

Une **table de dimensions** décrit le contexte : zone, date, véhicule, client, méthode de paiement. Elle contient des attributs utilisés pour filtrer, regrouper et présenter les faits.

| Élément | Exemple NYC Taxi |
|---|---|
| Fait | Trajet |
| Mesures additives | `fare_amount`, `total_amount`, `trip_distance` |
| Mesures semi-additives | Solde à un instant donné (exemple général) |
| Dimension | Zone TLC, date, paiement |
| Clé étrangère | `pickup_location_id`, `dropoff_location_id` |
| Grain | Un trajet par ligne |

Une mesure est **additive** si elle peut être sommée sur toutes les dimensions. Une durée moyenne ou un ratio doit généralement être recalculé à partir de composants (sommes et comptes), pas additionné.

### Architecture en couches

```text
raw / sources
    ↓
staging : renommage, types, nettoyage léger, 1:1 avec une source
    ↓
intermediate / processing : logique de jointure et règles réutilisables
    ↓
presentation / marts : faits, dimensions et agrégats orientés métier
    ↓
BI / data products
```

- **Staging** : proche de la source, peu de logique métier, colonnes standardisées.
- **Processing/intermediate** : étapes de transformation plus complexes, parfois éphémères.
- **Presentation/marts** : contrat consommé par les analystes et les outils BI.

Le schéma en étoile place une table de faits au centre et les dimensions autour. Un schéma en flocon normalise davantage les dimensions, mais ajoute des jointures.

---

## 5. dbt : principes fondamentaux

### Qu’est-ce que dbt ?

**dbt (data build tool)** est un outil de transformation analytique. On écrit du SQL, on ajoute des instructions Jinja et des fichiers YAML ; dbt compile puis exécute le SQL dans le moteur cible.

dbt apporte :

- modularité et dépendances entre modèles ;
- contrôle de version avec Git ;
- tests de qualité ;
- documentation et lineage ;
- environnements de développement et de production ;
- exécution incrémentale et réutilisation de macros.

### dbt Core et dbt Cloud

| | dbt Core | dbt Cloud |
|---|---|---|
| Nature | CLI open source | Service managé autour de dbt |
| Exécution | Machine locale/CI/orchestrateur | Jobs et environnements hébergés |
| Configuration | Fichiers et secrets à gérer | Interface, credentials, schedules |
| Collaboration | Git + outils externes | Git, IDE, logs, alertes selon offre |
| Coût | Logiciel gratuit, infrastructure à fournir | Offre et limites selon plan |

dbt Core convient à l’apprentissage et à une chaîne CI/CD maîtrisée. dbt Cloud simplifie l’exécution planifiée, les logs et la collaboration. Le module propose explicitement BigQuery + dbt Cloud et DuckDB + dbt Core comme alternatives.

### Fonctionnement

Pour un modèle :

```sql
-- models/staging/stg_green_tripdata.sql
select
    cast(vendorid as integer) as vendor_id,
    cast(lpep_pickup_datetime as timestamp) as pickup_datetime,
    cast(total_amount as numeric) as total_amount
from {{ source('raw', 'green_tripdata') }}
```

1. dbt lit le fichier SQL et les configurations ;
2. Jinja est rendu et `ref()`/`source()` sont résolus ;
3. le SQL compilé est placé dans `target/compiled` ;
4. dbt calcule le graphe des dépendances ;
5. l’adaptateur envoie le SQL à BigQuery/PostgreSQL/DuckDB ;
6. la matérialisation crée ou met à jour la relation cible ;
7. les tests et la documentation exploitent ce même graphe.

Le dossier `target/` est généré : il ne remplace pas les fichiers source et n’est généralement pas commité.

### Matérialisations

| Matérialisation | Résultat | Usage |
|---|---|---|
| `view` | Vue SQL recalculée à la lecture | Petits modèles, logique légère |
| `table` | Table persistée reconstruite | Mart stable, requêtes fréquentes |
| `incremental` | Seules les nouvelles/évolutions sont traitées | Gros faits append-only ou quasi append-only |
| `ephemeral` | CTE injecté dans les modèles parents | Étape intermédiaire non exposée |

Une table accélère souvent la lecture mais consomme du stockage et du temps de construction. Une vue évite de stocker mais recalcule à chaque lecture. Une stratégie incrémentale doit définir la clé, le filtre et la gestion des retards/corrections.

---

## 6. Installer et configurer dbt

### Installation générale

Choisir l’adaptateur correspondant au warehouse :

```bash
# BigQuery
python -m pip install dbt-bigquery

# PostgreSQL (alternative)
python -m pip install dbt-postgres

# Vérifier l’installation
 dbt --version
```

Utiliser de préférence un environnement virtuel :

```bash
python -m venv .venv
source .venv/bin/activate       # Linux/macOS
python -m pip install --upgrade pip
python -m pip install dbt-bigquery
```

Une exécution Docker peut isoler les dépendances. Le principe est de monter le projet et les credentials sans copier de secret dans l’image :

```bash
docker run --rm -it \
  -v "$PWD":/usr/app \
  -v "$HOME/.dbt":/root/.dbt \
  ghcr.io/dbt-labs/dbt-bigquery:latest \
  debug
```

En production, pinner une version testée plutôt que d’utiliser aveuglément `latest`.

### BigQuery

Prérequis : projet GCP, API BigQuery activée, dataset cible et permissions appropriées. Avec un compte de service, conserver la clé hors du dépôt et référencer son chemin via une variable d’environnement ou le mécanisme secret de dbt Cloud.

#### `profiles.yml`

Par défaut, dbt cherche ce fichier dans `~/.dbt/profiles.yml` :

```yaml
ny_taxi:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: mon-projet-gcp
      dataset: dbt_dev
      threads: 4
      keyfile: "/chemin/vers/service-account.json"
      location: US
      priority: interactive
```

Variantes courantes : `method: oauth` en local, ou `method: service-account-json` selon la manière de fournir le secret. Ne jamais commiter le JSON, un token ou un mot de passe.

Tester la connexion :

```bash
dbt debug --profile ny_taxi --target dev
```

#### `dbt_project.yml`

```yaml
name: ny_taxi
version: '1.0.0'
config-version: 2

profile: ny_taxi

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  ny_taxi:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

Le `name` doit correspondre à la clé de projet et `profile` à une clé de `profiles.yml`.

### PostgreSQL (alternative)

```yaml
ny_taxi:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      user: analytics
      password: "{{ env_var('DBT_POSTGRES_PASSWORD') }}"
      port: 5432
      dbname: warehouse
      schema: dbt_dev
      threads: 4
```

Pour PostgreSQL, configurer le réseau, la base, les droits de création de schéma et les variables d’environnement. La syntaxe SQL doit respecter les particularités du moteur cible.

Créer un squelette :

```bash
dbt init ny_taxi
cd ny_taxi
dbt debug
```

---

## 7. Structure d’un projet dbt

```text
ny_taxi/
├── dbt_project.yml
├── packages.yml
├── profiles.yml             # souvent hors dépôt, dans ~/.dbt
├── models/
│   ├── staging/
│   │   ├── stg_green_tripdata.sql
│   │   ├── stg_yellow_tripdata.sql
│   │   └── schema.yml
│   └── marts/
│       ├── dim_zones.sql
│       ├── fact_trips.sql
│       └── schema.yml
├── seeds/
│   └── taxi_zone_lookup.csv
├── snapshots/               # si snapshots utilisés
├── macros/
├── tests/                   # tests SQL singuliers
├── analyses/
├── README.md
└── .gitignore
```

| Répertoire/fichier | Rôle |
|---|---|
| `models/` | Modèles SQL et YAML associés |
| `seeds/` | Petits CSV versionnés chargés par dbt |
| `tests/` | Tests SQL personnalisés (« singular ») |
| `macros/` | Fonctions Jinja/SQL réutilisables |
| `packages.yml` | Dépendances de packages dbt |
| `schema.yml` | Descriptions, sources et tests |
| `dbt_project.yml` | Configuration du projet et des modèles |
| `analyses/` | Requêtes exploratoires non matérialisées |

Le nom `schema.yml` est conventionnel : plusieurs fichiers YAML sont possibles.

---

## 8. Modèles, sources et seeds

### SQL + Jinja

Un modèle est du SQL valide après compilation. Jinja permet de paramétrer :

```sql
select
    {{ dbt_utils.generate_surrogate_key(['vendor_id', 'pickup_datetime']) }} as trip_id,
    vendor_id,
    pickup_datetime
from {{ ref('stg_green_tripdata') }}
```

Jinja permet aussi `if`, boucles, variables et macros. Garder le SQL lisible et éviter de générer des centaines de colonnes implicitement.

### `ref()`

`ref('modele')` pointe vers un modèle dbt :

```sql
select *
from {{ ref('stg_yellow_tripdata') }}
```

dbt remplace l’expression par le nom pleinement qualifié adapté à la cible et crée une dépendance. C’est préférable à écrire en dur `project.dataset.table`.

### `source()`

Déclarer une table externe au projet :

```yaml
version: 2
sources:
  - name: raw
    database: mon-projet-gcp
    schema: raw_nyc_taxi
    tables:
      - name: green_tripdata
      - name: yellow_tripdata
```

Puis l’utiliser :

```sql
select * from {{ source('raw', 'green_tripdata') }}
```

Cela documente l’origine, permet les tests de sources et rend le lineage visible.

### Staging : exemple

```sql
{{ config(materialized='view') }}

with source_data as (
    select *
    from {{ source('raw', 'green_tripdata') }}
)

select
    cast(vendorid as integer) as vendor_id,
    cast(ratecodeid as integer) as rate_code_id,
    cast(pulocationid as integer) as pickup_location_id,
    cast(dolocationid as integer) as dropoff_location_id,
    cast(lpep_pickup_datetime as timestamp) as pickup_datetime,
    cast(lpep_dropoff_datetime as timestamp) as dropoff_datetime,
    cast(passenger_count as integer) as passenger_count,
    cast(trip_distance as numeric) as trip_distance,
    cast(total_amount as numeric) as total_amount
from source_data
where lpep_pickup_datetime is not null
```

Le staging standardise les noms/types et réalise des contrôles légers ; il ne doit pas devenir un mart caché.

### Configurer une matérialisation

```sql
{{ config(
    materialized='incremental',
    unique_key='trip_id',
    on_schema_change='sync_all_columns'
) }}

select *
from {{ ref('stg_yellow_tripdata') }}
{% if is_incremental() %}
where pickup_datetime >= (select max(pickup_datetime) from {{ this }})
{% endif %}
```

`{{ this }}` désigne la relation construite. Le filtre doit être conçu selon les retards, corrections et fuseaux horaires du domaine.

### Seeds

Un **seed** est un fichier CSV de petite taille, versionné dans Git puis chargé par dbt :

```bash
dbt seed
```

Le fichier `seeds/taxi_zone_lookup.csv` contient le mapping des identifiants de zones TLC vers `zone`, `borough` et `service_zone`. On le référence avec :

```sql
select * from {{ ref('taxi_zone_lookup') }}
```

Les seeds conviennent aux référentiels stables et peu volumineux, pas aux gros flux ni aux données secrètes. On peut typer et documenter un seed dans YAML.

---

## 9. Tests et qualité

Un test dbt retourne les lignes qui violent une assertion. **Zéro ligne = test réussi**.

### Tests génériques

Dans `models/staging/schema.yml` :

```yaml
version: 2

models:
  - name: stg_green_tripdata
    description: "Trajets green taxi standardisés."
    columns:
      - name: vendor_id
        tests:
          - not_null
      - name: pickup_datetime
        tests:
          - not_null
      - name: trip_id
        tests:
          - unique
          - not_null
      - name: payment_type
        tests:
          - accepted_values:
              values: [1, 2, 3, 4, 5, 6]
      - name: pickup_location_id
        tests:
          - relationships:
              to: ref('dim_zones')
              field: location_id
```

Selon la version et les conventions du projet, la clé peut être `tests:` ou `data_tests:`. Vérifier la version de dbt et la documentation de l’adaptateur.

- `not_null` : la colonne est renseignée ;
- `unique` : chaque valeur est unique ;
- `accepted_values` : la valeur appartient à un ensemble ;
- `relationships` : les clés étrangères existent dans une relation parent.

Ne pas appliquer `unique` à une colonne qui n’est pas unique par grain. Pour une clé composite, créer un identifiant déterministe ou utiliser une macro adaptée.

### Test personnalisé (singular test)

`tests/no_negative_fares.sql` :

```sql
select
    trip_id,
    fare_amount
from {{ ref('fact_trips') }}
where fare_amount < 0
```

Le test échoue si la requête renvoie une ou plusieurs lignes. On peut aussi rendre la règle paramétrable avec une macro.

### Tests de modèle complets

```yaml
models:
  - name: fact_trips
    description: "Un enregistrement par trajet taxi, yellow ou green."
    columns:
      - name: trip_id
        tests: [unique, not_null]
      - name: pickup_location_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_zones')
              field: location_id
```

Tester aussi : dates cohérentes (`dropoff_datetime >= pickup_datetime`), montants plausibles, absence de doublons après union, couverture des jointures et volumes attendus.

### Tests de sources

```yaml
sources:
  - name: raw
    tables:
      - name: yellow_tripdata
        loaded_at_field: pickup_datetime
        freshness:
          warn_after: {count: 24, period: hour}
          error_after: {count: 48, period: hour}
```

La fraîcheur dépend de la source et doit être interprétée avec prudence pour des données historiques comme NYC Taxi.

---

## 10. Documentation, macros et packages

### Documentation

Descriptions de modèles, colonnes et sources vivent dans YAML :

```yaml
models:
  - name: dim_zones
    description: "Dimension des zones TLC utilisées pour les trajets."
    columns:
      - name: location_id
        description: "Identifiant TLC de la zone."
      - name: borough
        description: "Arrondissement de New York."
```

Générer puis servir la documentation :

```bash
dbt docs generate
dbt docs serve
```

La documentation contient le catalogue, les descriptions, les tests et le graphe de lineage. `dbt docs generate` peut nécessiter l’accès au warehouse pour collecter les métadonnées. Documenter le **pourquoi** et le grain, pas seulement le type SQL.

### Macros

Une macro est une fonction Jinja réutilisable :

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name, decimals=2) %}
    round(cast({{ column_name }} as numeric) / 100, {{ decimals }})
{% endmacro %}
```

Utilisation :

```sql
select {{ cents_to_dollars('fare_in_cents') }} as fare_amount
from {{ ref('stg_trips') }}
```

Exemple de logique conditionnelle :

```sql
{% macro payment_type_label(column_name) %}
case {{ column_name }}
  when 1 then 'Credit card'
  when 2 then 'Cash'
  else 'Other'
end
{% endmacro %}
```

Jinja fournit variables, conditions et boucles :

```sql
{% set measures = ['fare_amount', 'tip_amount', 'tolls_amount'] %}
select
    trip_id,
    {% for measure in measures %}
    sum({{ measure }}) as {{ measure }}{% if not loop.last %},{% endif %}
    {% endfor %}
from {{ ref('fact_trips') }}
group by trip_id
```

Éviter de cacher une logique métier critique dans trop de macros : elle devient plus difficile à lire et tester.

### Packages

`packages.yml` :

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: ">=1.1.0,<2.0.0"
```

Installer les dépendances :

```bash
dbt deps
```

`dbt_utils` propose des macros comme `generate_surrogate_key`, des tests et des utilitaires. Pinner une plage de versions, commiter le lockfile si généré par la version utilisée, et contrôler les licences/versions.

---

## 11. Cas pratique : NYC Taxi

### Données et objectif

Le projet utilise les données **green taxi** et **yellow taxi** de New York, en particulier le dataset historique **2019–2020**. Le FHV éventuellement rencontré au module précédent n’est pas utilisé dans ce projet dbt. Le référentiel géographique est `taxi_zone_lookup.csv` chargé comme seed.

### Sources et staging

Déclarer les tables brutes puis créer :

- `stg_green_tripdata` : noms/types harmonisés pour les trajets green ;
- `stg_yellow_tripdata` : noms/types harmonisés pour les trajets yellow.

Colonnes typiques harmonisées :

```text
vendor_id
rate_code_id
pickup_location_id / dropoff_location_id
pickup_datetime / dropoff_datetime
passenger_count
trip_distance
fare_amount, tip_amount, tolls_amount, total_amount
payment_type
```

Les schémas de source peuvent différer entre green et yellow : documenter les renommages et ne pas supposer que toutes les colonnes ont exactement la même signification.

### Dimension des zones

`dim_zones` utilise le seed :

```sql
select
    cast(locationid as integer) as location_id,
    borough,
    zone,
    service_zone
from {{ ref('taxi_zone_lookup') }}
```

Contrôles recommandés : `location_id` non nul et unique ; arrondissement et zone documentés ; correspondance avec les clés des faits.

### Table de faits

Le grain de `fact_trips` est **un trajet par ligne**. Exemple de construction :

```sql
{{ config(materialized='table') }}

with green as (
    select
        'Green' as service_type,
        vendor_id,
        pickup_datetime,
        dropoff_datetime,
        pickup_location_id,
        dropoff_location_id,
        passenger_count,
        trip_distance,
        fare_amount,
        tip_amount,
        total_amount,
        payment_type
    from {{ ref('stg_green_tripdata') }}
),

yellow as (
    select
        'Yellow' as service_type,
        vendor_id,
        pickup_datetime,
        dropoff_datetime,
        pickup_location_id,
        dropoff_location_id,
        passenger_count,
        trip_distance,
        fare_amount,
        tip_amount,
        total_amount,
        payment_type
    from {{ ref('stg_yellow_tripdata') }}
),

unioned as (
    select * from green
    union all
    select * from yellow
)

select
    {{ dbt_utils.generate_surrogate_key([
        'service_type', 'vendor_id', 'pickup_datetime',
        'dropoff_datetime', 'pickup_location_id', 'dropoff_location_id'
    ]) }} as trip_id,
    *
from unioned
```

Une clé surrogate basée uniquement sur les colonnes ci-dessus peut théoriquement entrer en collision si deux événements sont identiques ; le grain et la stratégie d’identification doivent être vérifiés sur les données réelles.

### Analyses utiles

Une fois la table bâtie :

```sql
select
    service_type,
    date_trunc(pickup_datetime, month) as month,
    count(*) as trips,
    sum(total_amount) as revenue,
    avg(trip_distance) as avg_distance
from {{ ref('fact_trips') }}
group by 1, 2
order by 1, 2
```

Pour BigQuery, adapter les fonctions de date si le type est `datetime` plutôt que `timestamp`. Toujours contrôler les unités, les valeurs nulles et les anomalies avant de publier un KPI.

---

## 12. Déploiement et CI/CD

### dbt Cloud jobs

Un **job** dbt Cloud contient typiquement :

1. environnement et version dbt ;
2. connexion au warehouse ;
3. dépôt Git et branche ;
4. commandes (`dbt deps`, `dbt seed`, `dbt build`) ;
5. calendrier ou déclencheur ;
6. notifications et logs.

Un job de production peut exécuter :

```bash
dbt deps
dbt build --target prod
```

`dbt build` construit les ressources sélectionnées et lance les tests associés (modèles, seeds, snapshots et tests selon sélection).

### Environnements

- **dev** : schéma personnel, petits échantillons et itérations rapides ;
- **CI** : environnement temporaire pour valider une pull request ;
- **prod** : schéma partagé, credentials restreints, données complètes.

Ne pas faire pointer le développement vers les tables de production avec des droits d’écriture. Utiliser des variables et secrets différents par environnement.

### CI/CD

Pipeline recommandé :

```text
Pull request → dbt deps → dbt compile → dbt build ciblé
             → revue → merge → job production planifié
```

Dans CI, sélectionner les modèles modifiés et leurs parents/enfants selon le besoin (`state:modified`, `--select`). Le job de production reconstruit les modèles nécessaires et publie les artefacts/logs.

### Scheduling

Planifier après la disponibilité des données sources. Pour une source quotidienne : ingestion → contrôle de fraîcheur → dbt → tests → notification → BI. dbt Cloud peut planifier ; Airflow/Dagster/Prefect peuvent orchestrer dbt dans une chaîne plus large.

---

## 13. Visualisation dans Looker Studio

Le cours utilise **Google Data Studio**, aujourd’hui appelé **Looker Studio**. Après construction des marts dans BigQuery :

1. ouvrir Looker Studio ;
2. ajouter une source BigQuery ;
3. sélectionner le projet, dataset et `fact_trips`/vues analytiques ;
4. vérifier types, dimensions et métriques ;
5. construire cartes, séries temporelles, tableaux et filtres ;
6. partager avec les bonnes permissions.

KPI possibles : nombre de trajets, revenu total, revenu moyen par trajet, distance moyenne, répartition green/yellow, évolution mensuelle et zones de départ/arrivée.

Bonnes pratiques : exposer des marts stables plutôt que les tables brutes, pré-agréger les gros volumes si nécessaire, vérifier les fuseaux horaires, distinguer somme et moyenne, et documenter la définition de chaque KPI.

---

## 14. Commandes essentielles

| Commande | Explication |
|---|---|
| `dbt init nom` | Crée un squelette de projet et les fichiers de départ |
| `dbt debug` | Vérifie installation, profil, credentials et connexion |
| `dbt seed` | Charge les CSV du dossier `seeds/` |
| `dbt run` | Compile et matérialise les modèles |
| `dbt test` | Exécute les tests définis |
| `dbt build` | Construit les ressources et exécute les tests associés |
| `dbt deps` | Installe les packages de `packages.yml` |
| `dbt docs generate` | Génère le catalogue et le graphe de documentation |
| `dbt docs serve` | Sert la documentation localement |
| `dbt compile` | Compile sans exécuter les modèles |
| `dbt ls` | Liste les ressources sélectionnables |

Exemples de sélection :

```bash
dbt run --select stg_green_tripdata
dbt build --select fact_trips+
dbt test --select tag:critical
dbt run --full-refresh --select fact_trips
dbt compile --select marts+
```

`+` sélectionne les parents/enfants selon sa position. `--full-refresh` reconstruit un modèle incrémental ; l’utiliser avec discernement sur un gros dataset.

Un cycle local typique :

```bash
dbt debug
dbt deps
dbt seed
dbt build
dbt docs generate
dbt docs serve
```

---

## 15. Bonnes pratiques

### Conception

- Définir le grain de chaque modèle dans sa description.
- Utiliser des noms explicites et cohérents (`stg_`, `dim_`, `fact_`).
- Garder le staging proche de la source ; placer la logique métier dans les marts.
- Préférer `ref()` et `source()` aux noms de tables codés en dur.
- Éviter `select *` dans les modèles de présentation : le contrat devient explicite.
- Standardiser les types, unités, fuseaux horaires et conventions de nommage.

### Qualité

- Tester les clés, relations, valeurs acceptées et invariants métier.
- Tester les sources critiques et leur fraîcheur quand cela a du sens.
- Contrôler doublons, nulls, dates impossibles et montants aberrants.
- Faire échouer tôt la CI plutôt que publier un mart douteux.
- Ajouter une description au modèle et aux colonnes importantes.

### Performance et coût

- Éviter de scanner inutilement les gros faits ; filtrer par partition/date.
- Choisir `view`, `table`, `incremental` ou `ephemeral` selon l’usage réel.
- Configurer partitions/clustering BigQuery pour les grosses tables.
- Surveiller les coûts des requêtes et la croissance des tables.
- Tester un incremental sur les insertions, mises à jour, retards et full refresh.

### Collaboration et sécurité

- Versionner SQL/YAML/macros et faire des pull requests.
- Ne pas versionner `profiles.yml`, clés JSON, tokens ni secrets.
- Pinner versions d’adaptateurs et packages ; documenter les upgrades.
- Utiliser des permissions minimales et des datasets séparés dev/prod.
- Lire le SQL compilé lors du debug : il montre ce que le warehouse exécute réellement.

---

## 16. FAQ et astuces

### dbt est-il un orchestrateur ?

Pas au sens complet. dbt connaît l’ordre des transformations dans son graphe, mais une plateforme d’orchestration gère aussi ingestion, dépendances externes, retries, SLA et alertes globales.

### Pourquoi mon modèle ne trouve-t-il pas une table ?

Vérifier `dbt debug`, le profil actif, le projet/dataset BigQuery, le nom de source, les permissions et la région BigQuery. Exécuter `dbt compile` puis inspecter `target/compiled`.

### `ref()` ou `source()` ?

`source()` pour une table extérieure au projet, `ref()` pour un modèle/seed/snapshot géré par dbt. Les deux construisent le lineage, mais représentent des contrats différents.

### Quand utiliser un seed ?

Pour un petit référentiel stable, comme `taxi_zone_lookup.csv`. Pour un fichier volumineux, fréquemment mis à jour ou sensible, utiliser un pipeline d’ingestion dédié.

### `dbt run` ou `dbt build` ?

`dbt run` construit les modèles. `dbt build` orchestre une construction plus complète des ressources sélectionnées et lance les tests associés. En production, `build` est souvent le choix par défaut, après avoir vérifié la sélection.

### Un test `unique` échoue : que faire ?

Revenir au grain. La colonne est-elle réellement une clé ? Existe-t-il des doublons légitimes ? Faut-il ajouter une clé composite, dédupliquer avec `row_number()`, ou corriger une jointure qui multiplie les lignes ? Ne pas supprimer le test simplement pour le faire passer.

### Comment déboguer Jinja ?

Commencer par `dbt compile`, consulter le SQL compilé, réduire la sélection avec `--select`, puis tester la requête directement dans le warehouse. Vérifier les guillemets, les virgules générées par les boucles et les variables non définies.

### Quand faire un `--full-refresh` ?

Après une modification incompatible du modèle incrémental, une correction historique ou une stratégie de clé changée. Prévenir sur les tables volumineuses : le coût peut être élevé et le temps d’exécution important.

### Comment éviter un mauvais KPI dans Looker Studio ?

Publier une colonne ou vue documentée, fixer le grain, définir les métriques (somme, moyenne, ratio), tester quelques valeurs manuellement et contrôler le fuseau horaire avant le partage.

### Mémo final

```text
Source déclarée → staging propre → ref() → mart au grain explicite
→ tests → documentation → build CI/production → BI
```

Si le modèle est compréhensible, testable, reproductible et adapté au grain métier, il remplit l’objectif de l’Analytics Engineering.

---

## Références

- [Dépôt officiel DataTalksClub — Module 4](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/master/04-analytics-engineering)
- [Documentation dbt](https://docs.getdbt.com/)
- [dbt Fundamentals](https://learn.getdbt.com/courses/dbt-fundamentals)
- [BigQuery](https://cloud.google.com/bigquery/docs)
- [Looker Studio](https://lookerstudio.google.com/)
