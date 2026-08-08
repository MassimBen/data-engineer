Voici la version mise à jour du `README.md`. J'ai considérablement approfondi les sections **Window Functions** (avec des exemples concrets issus du jeu de données Taxi du Zoomcamp) et **BigQuery ML** (avec les étapes d'évaluation et les types de modèles), tout en gardant le formatage propre pour GitHub.

***
N'hésite pas à rajouter un badge en haut de ton README comme ça :
`![BigQuery](https://img.shields.io/badge/BigQuery-Module_3-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)` 
pour le rendre plus visuel !
```markdown
# 📚 BigQuery - Cheatsheet & Notes (DataTalksClub DE Zoomcamp - Module 3)

Ce document est un résumé exhaustif du Module 3 du Data Engineering Zoomcamp. Il couvre les fondamentaux du Data Warehousing, l'architecture interne de BigQuery, l'optimisation des coûts et les fonctionnalités avancées.

---

## 🏢 1. Data Warehouse et BigQuery

### OLTP vs OLAP
*   **OLTP (Online Transaction Processing) :** Bases de données transactionnelles (PostgreSQL, MySQL). Conçues pour écrire et mettre à jour des données rapidement (ex: ajout au panier). Modèle très normalisé.
*   **OLAP (Online Analytical Processing) :** Entrepôts de données (BigQuery, Snowflake). Conçus pour lire et agréger d'immenses volumes (ex: ventes par mois). Modèle dénormalisé.

### Schema-on-Write vs Schema-on-Read
*   **Schema-on-Write (OLTP) :** Le schéma est vérifié *avant* d'écrire. Si la donnée ne correspond pas, elle est rejetée.
*   **Schema-on-Read (BigQuery) :** On charge les données brutes très rapidement. Le schéma est appliqué *au moment de la lecture* (lors de la requête SQL).

---

## ⚙️ 2. Internals of BigQuery (Sous le capot)

*   **Disaggregated Architecture :** Le stockage et la puissance de calcul sont gérés par des services indépendants.
*   **Stockage : Colossus.** Système de fichiers distribué de Google. Les données sont répliquées automatiquement.
*   **Format Interne : Capacitor.** BigQuery convertit tout dans son propre format colonnaire compressé. Cela permet le **Predicate Pushdown** (ne lire que les colonnes/lignes pertinentes sur le disque).
*   **Calcul : Dremel.** Moteur massivement parallèle en arbre. Le nœud racine divise la requête, les feuilles lisent sur Colossus, et les résultats remontent.

> 💡 **La Règle d'Or :** En mode "On-Demand", tu paies strictement pour la quantité de données **lues (scannées)** par Dremel. (5$ par To lu).

---

## 🗂️ 3. Partitioning vs Clustering

Puisqu'on paie à la lecture, l'objectif est de réduire la taille de la donnée lue.

### Le Partitionnement (Physical Separation)
*   **Usage :** Uniquement sur des colonnes de **Date** ou **Timestamp** (`PARTITION BY DATE(date_col)`).
*   **Pourquoi :** Si tu filtres sur `WHERE date = '2023-10-01'`, Dremel ne lit que le dossier de cette journée.
*   **Limites :** Max 4000 partitions. Ne **JAMAIS** partitionner par heure ou par pays.

### Le Clustering (Logical Sorting)
*   **Usage :** Sur des colonnes de filtrage fréquentes (String, Int) : `CLUSTER BY country, status`.
*   **Pourquoi :** Trie les données *à l'intérieur* de la partition. BQ ira directement au bloc contenant 'FR' sans lire le reste de la journée.
*   **Avantage :** Géré automatiquement par BQ en arrière-plan.

### ⚖️ Comparatif Rapide

| Caractéristique | Partitionnement | Clustering |
| :--- | :--- | :--- |
| **Analogie** | Dossiers physiques distincts | Tri alphabétique dans un dossier |
| **Types de colonnes** | Uniquement `DATE` / `TIMESTAMP` | `STRING`, `INT64`, `BOOL`, etc. |
| **Limites** | Max 4000 partitions par table | Aucune limite stricte |

**La Combinaison Gagnante :** `PARTITION BY DATE(date_column) CLUSTER BY country;`

---

## 📥 4. Ingestion et Formats de Données

| Format | Efficacité BQ | Cas d'usage |
| :--- | :--- | :--- |
| **CSV/JSON** | ❌ Mauvaise | À éviter en prod (BQ doit tout lire/deviner). |
| **Parquet** | ✅ Excellente | **Le standard BQ.** Lecture rapide. |
| **Avro** | ✅ Très bonne | Idéal pour le streaming (schéma intégré). |

**Micro-batching :** Préférer `Pub/Sub -> Dataflow -> Parquet sur GCS -> bq load` plutôt que l'API Streaming (`insertAll`) qui coûte cher.
**Bonnes pratiques d'ingestion :**
*   **Éviter le streaming natif (`insertAll`) en continu :** C'est cher à l'octet.
*   **Préférer le Micro-batching :** Streamer vers Pub/Sub -> Dataflow (fenêtres de 5 min) -> Écrire en Parquet sur GCS -> `bq load` (Batch).

---

## 🚀 5. Advanced Topics (Fonctionnalités Avancées)

### A. Nested & Repeated Fields (STRUCT et ARRAY)
On évite les `JOIN` coûteux en imbriquant les données (Dénormalisation).
*   **`UNNEST()` :** Aplatir un ARRAY.
    ```sql
    -- CROSS JOIN (virgule) : Supprime la ligne parente si l'array est vide
    SELECT t.id, tag FROM taxis t, UNNEST(t.tags) AS tag
    -- LEFT JOIN : Garde le taxi même s'il n'a pas de tags
    SELECT t.id, tag FROM taxis t LEFT JOIN UNNEST(t.tags) AS tag
    ```
*   **Manipulation :** `ARRAY_AGG()`, `my_array[SAFE_OFFSET(5)]` (évite les crashs out of bounds).

La dénormalisation est reine dans BQ. On évite les `JOIN` coûteux en imbriquant les données directement dans la ligne.

*   **Repeated (ARRAY) :** Une liste de valeurs (ex: plusieurs tags pour un article).
*   **Nested (STRUCT) :** Un objet contenant plusieurs champs (ex: une adresse contenant rue, ville, code_postal).

**La fonction magique : `UNNEST()`**
Pour interroger un ARRAY, on doit l'aplatir.
```sql
-- Syntaxe avec CROSS JOIN (implicite avec la virgule)
-- ⚠️ Si le taxi n'a pas de trajet, il disparait du résultat !
SELECT taxi_id, trajet.prix
FROM taxis, UNNEST(trajets) AS trajet

-- Syntaxe sécurisée avec LEFT JOIN
-- ✅ Le taxi reste même s'il n'a pas de trajet
SELECT taxi_id, trajet.prix
FROM taxis
LEFT JOIN UNNEST(trajets) AS trajet
```
### B. Manipulation avancée des ARRAYs
*   **Créer un array :** `ARRAY_AGG(column_name)`
*   **Accéder à un élément :** `my_array[OFFSET(0)]` (index 0) ou `my_array[ORDINAL(1)]` (index 1).
*   **Accéder en sécurité :** `SAFE_OFFSET(5)` (ne crash pas si l'index n'existe pas, retourne NULL).

### B. Données Géospatiales (`GEOGRAPHY`)
*   Type natif pour la Terre (sphère).
*   `ST_GeogPoint(longitude, latitude)` ⚠️ *Attention : c'est Long puis Lat !*
*   Fonctions utiles : `ST_Distance()`, `ST_Contains()`.
*   BigQuery possède un type natif `GEOGRAPHY` qui comprend les formes sur une sphère (la Terre), contrairement aux simples coordonnées (`FLOAT`).
*   **Création :** `ST_GeogPoint(longitude, latitude)` ⚠️ *Attention, c'est Long puis Lat, pas Lat/Lon !*
*   **Fonctions utiles :** `ST_Distance()`, `ST_Contains()`, `ST_Within()`.
*   **Outil :** [BigQuery Geo Viz](https://bigquerygeoviz.withgoogle.com/) pour visualiser les résultats sur une carte directement.

### C. Time Travel & Point-in-Time
BQ garde un historique de toutes les modifications.
*   Permet de requêter une table telle qu'elle était dans le passé (jusqu'à 7 jours par défaut).
```sql
SELECT * FROM `my_table`
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 2 HOUR)
```
*   *Utile pour :* Récupérer des données supprimées par erreur ou analyser l'état d'un système à un instant T.
### D. Materialized Views
Stocke le résultat d'une agrégation et se **met à jour automatiquement** de façon incrémentale. *Limites : Pas de JOIN avec la table source, pas de `NOW()`.*
Stocke physiquement le résultat d'une requête d'agrégation et se **met à jour automatiquement** de façon incrémentale quand de nouvelles données arrivent dans la table de base.
*   *Avantage :* Transforme une requête qui scanne 1 To en une requête qui scanne 1 Go.
*   *Limites :* Pas de `JOIN`, pas de fonctions non déterministes (`NOW()`), inutile si la table de base est mise à jour en permanence.


### E. Information Schema
Pour interroger les métadonnées de ton projet (utile pour l'automatisation et le débogage).
```sql
-- Voir toutes les colonnes d'un dataset
SELECT * FROM `my_project.my_dataset.INFORMATION_SCHEMA.COLUMNS`;
```

---

## 🧠 6. Deep Dive : Window Functions (Fonctions de Fenêtrage)

C'est l'outil le plus puissant pour l'analyse de séries temporelles (ex: trajets de taxis). 
**Le problème :** Un `GROUP BY` écrase les lignes. Un `SELF-JOIN` (pour comparer une ligne à la précédente) oblige BQ à lire la table **deux fois** (donc tu paies deux fois la facture).
**La solution :** Les Window Functions lisent la table **une seule fois** et calculent des valeurs relatives à la ligne courante.

#### La Syntaxe
```sql
FONCTION() OVER (
  PARTITION BY colonne_pour_grouper 
  ORDER BY colonne_pour_trier 
  frame_clause
)
```

#### Les 3 grandes familles (avec exemples concrets Taxi)

**1. Les fonctions de décalage (Offset) : `LAG` et `LEAD`**
Permettent de regarder la ligne précédente ou suivante *dans un même groupe*.
*Cas d'usage :* Calculer le temps d'attente d'un taxi entre deux trajets.
```sql
SELECT 
  taxi_id,
  pickup_datetime,
  LAG(pickup_datetime) OVER (PARTITION BY taxi_id ORDER BY pickup_datetime) AS prev_trip_time,
  -- Calcul du temps d'attente entre le trajet N et le trajet N-1
  TIMESTAMP_DIFF(pickup_datetime, 
    LAG(pickup_datetime) OVER (PARTITION BY taxi_id ORDER BY pickup_datetime), 
    MINUTE) AS wait_time_minutes
FROM trips
```

**2. Les fonctions d'agrégation classiques (`SUM`, `AVG`, `COUNT`)**
Au lieu d'écraser les lignes, elles ajoutent une colonne avec le total du groupe.
*Cas d'usage :* Quel est le pourcentage du revenu total du taxi que représente ce trajet spécifique ?
```sql
SELECT 
  trip_id,
  fare_amount,
  SUM(fare_amount) OVER (PARTITION BY taxi_id) AS total_revenue_taxi,
  -- Division pour avoir le %
  (fare_amount / SUM(fare_amount) OVER (PARTITION BY taxi_id)) * 100 AS pct_revenue
FROM trips
```

**3. Les fonctions de numérotation : `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`**
Attribuent un numéro à chaque ligne dans un groupe trié.
*Cas d'usage :* Trouver le trajet le plus cher pour chaque taxi.
```sql
SELECT * FROM (
  SELECT 
    trip_id,
    fare_amount,
    ROW_NUMBER() OVER (PARTITION BY taxi_id ORDER BY fare_amount DESC) as rank
  FROM trips
) WHERE rank = 1 -- Garde seulement la ligne n°1 de chaque taxi
```

#### La clause `ROWS BETWEEN` (Frames)
Par défaut, si tu utilises `SUM` avec un `ORDER BY`, la fenêtre calcule du début de la partition **jusqu'à** la ligne courante (cumul). Tu peux changer ce comportement :
```sql
-- Moyenne mobile : La ligne courante + la précédente + la suivante
AVG(fare_amount) OVER (
  PARTITION BY taxi_id 
  ORDER BY pickup_datetime 
  ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
) AS moving_avg_fare
```

---

## 🤖 7. Deep Dive : BigQuery ML (BQML)

Pourquoi exporter 1 To de données vers un notebook Python (coût de sortie GCS + coût de calcul VM) alors que BQ peut faire le Machine Learning **directement là où sont les données** ?

#### Le Workflow BQML en 3 étapes

**Étape 1 : Entraîner le modèle**
On utilise `CREATE OR REPLACE MODEL` suivi d'options. BQ choisit automatiquement les hyperparamètres optimaux.
```sql
CREATE OR REPLACE MODEL `my_dataset.taxi_tip_model`
OPTIONS(
  model_type='linear_reg', -- Régression linéaire
  input_label_cols=['tip_amount'], -- Ce qu'on veut prédire
  l1_reg=0.1 -- Optionnel : régularisation
) AS
SELECT
  fare_amount,
  trip_distance,
  passenger_count,
  tip_amount -- La colonne cible DOIT être dans le SELECT
FROM
  `my_dataset.cleaned_trips`
WHERE
  tip_amount IS NOT NULL;
```

**Étape 2 : Évaluer le modèle**
Ne jamais faire confiance à un modèle sans vérifier ses métriques. BQ génère une table de métriques.
```sql
SELECT * FROM ML.EVALUATE(MODEL `my_dataset.taxi_tip_model`);
-- Résultat : RMSE, MAE, R2, etc.
```

**Étape 3 : Prédire (Inférence)**
On applique le modèle sur de nouvelles données (qui n'ont pas la colonne cible).
```sql
SELECT * FROM ML.PREDICT(
  MODEL `my_dataset.taxi_tip_model`,
  (
    SELECT fare_amount, trip_distance, passenger_count 
    FROM `my_dataset.new_trips_today`
  )
);
```
*Note : La requête de prédiction ajoute automatiquement des colonnes comme `predicted_tip_amount` et `predicted_tip_amount_probs` au résultat.*

#### Types de modèles supportés
*   **Prédiction numérique :** `linear_reg` (Régression linéaire), `boosted_tree_regressor` (Arbres - très puissant).
*   **Classification :** `logistic_reg` (Binaire/Multiclasse), `boosted_tree_classifier`.
*   **Clustering :** `kmeans` (Pour segmenter des clients, par exemple).
*   **Time Series :** `ARIMA_PLUS` (Nouveau, très performant pour les prévisions de ventes).
*   **IA Générative / Deep Learning :** BQML permet d'importer des modèles TensorFlow sauvegardés ou d'appeler des endpoints Vertex AI directement en SQL.

#### Pourquoi c'est révolutionnaire pour un DE ?
1.  **Zero ETL pour le ML :** Pas de pipeline Airflow compliqué pour envoyer les données à un Data Scientist.
2.  **Familiarité :** N'importe quel Data Analyst sachant écrire du SQL peut créer un modèle prédictif.
3.  **Performance :** BQ parallélise l'entraînement sur des milliers de nœuds Dremel en quelques minutes, même sur des milliards de lignes.

---

## 🛡️ 8. Checklist des Bonnes Pratiques Absolues (DE)

### Schéma et Modélisation
- [ ] **Ne JAMAIS utiliser `STRING` par défaut.** Utiliser `INT64`, `BOOL`, `TIMESTAMP`.
- [ ] **Dénormaliser :** Préférer de larges tables avec des `STRUCT/ARRAY` pour éviter les `JOIN`.
- [ ] **Utiliser `GEOGRAPHY`** pour les coordonnées GPS.

### Performance et Réduction de la Facture
- [ ] **Ne JAMAIS faire `SELECT *`.** Lister explicitement les colonnes.
- [ ] **Toujours Partitionner par Date** (`PARTITION BY DATE()`).
- [ ] **Toujours Clusteriser** sur les colonnes de filtre fréquentes (`CLUSTER BY`).
- [ ] **Ne pas sur-partitionner.** (Pas par heure).
- [ ] **Ne pas clusteriser sur des clés uniques** (ex: `user_id`).
- [ ] **Utiliser `APPROX_COUNT_DISTINCT`** au lieu de `COUNT(DISTINCT)` sur les gros volumes.

### Requêtage
- [ ] **Filtrer tôt :** Mettre les `WHERE` sur les partitions en priorité.
- [ ] **Préférer `UNNEST`** (avec LEFT JOIN) aux jointures multiples sur des tables imbriquées.
- [ ] **Préférer les Window Functions** (`LAG`, `ROW_NUMBER`) aux `SELF-JOIN` pour éviter de scanner la table deux fois.
- [ ] **Utiliser `SAFE_OFFSET` ou `SAFE_ORDINAL`** pour éviter les crashes lors de la manipulation des Arrays.



### Architecture
- [ ] **Stocker en Parquet sur GCS** avant de charger dans BQ.
- [ ] **Tirer parti du Cache :** Éviter `NOW()` ou `RAND()` dans les requêtes récurrentes pour profiter du cache automatique de 24h.
- [ ] **Utiliser BQML** pour les cas d'usage ML standards afin d'éviter les coûts d'extraction des données.
- [ ] **Materialized Views :** Les utiliser pour les agrégations lourdes et stables.
```




















**Conseil pour ton GitHub :** 
N'hésite pas à rajouter un badge en haut de ton README comme ça :
`![BigQuery](https://img.shields.io/badge/BigQuery-Module_3-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)` 
pour le rendre plus visuel !
