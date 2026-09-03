Voici le cours complet, assemblé en un seul fichier `.md`, avec une section **Optimisation Data Warehouse** entièrement retravaillée pour couvrir le Partitionnement et le Clustering (BigQuery, Snowflake, Redshift).

***
# 🐘 Cours de SQL Avancé & Optimisation Data Warehouse

Bienvenue dans ce cours complet de SQL Avancé. Ce guide est destiné aux développeurs, Data Analysts et Data Engineers qui maîtrisent déjà les bases (`SELECT`, `WHERE`, `JOIN`, `GROUP BY`) et souhaitent maîtriser la manipulation de données complexes et l'optimisation sur de gros volumes (Data Warehouses).

## 📚 Table des matières

1.  [Les Jointures Avancées](#1-les-jointures-avancées)
2.  [Les Fonctions de Fenêtrage (Window Functions)](#2-les-fonctions-de-fenêtrage-window-functions)
3.  [Les CTE (Common Table Expressions)](#3-les-cte-common-table-expressions)
4.  [Les CTE Récursives](#4-les-cte-récursives)
5.  [Agrégation Avancée (Grouping Sets)](#5-agrégation-avancée-grouping-sets)
6.  [Optimisation Data Warehouse (Partition & Cluster)](#6-optimisation-data-warehouse-partition--cluster)
7.  [Manipulation de Données Complexes](#7-manipulation-de-données-complexes)
8.  [Gestion des Transactions](#8-gestion-des-transactions)

---

## 1. Les Jointures Avancées

Au-delà du classique `INNER JOIN` et `LEFT JOIN`, il existe des jointures spécifiques pour des cas d'usage précis.

### Self Join (Jointure sur soi-même)
Permet de relier une table à elle-même, souvent pour des relations hiérarchiques (employés/managers).

```sql
SELECT 
    e1.nom AS employe, 
    e2.nom AS manager
FROM employes e1
LEFT JOIN employes e2 ON e1.manager_id = e2.id;
```

### Cross Join (Produit Cartésien)
Combine chaque ligne de la première table avec chaque ligne de la seconde. À utiliser avec prudence (volume élevé), utile pour générer des matrices de toutes les combinaisons possibles.

```sql
-- Exemple : Associer chaque produit à chaque vendeur pour un plan de campagne
SELECT produits.nom, vendeurs.nom
FROM produits
CROSS JOIN vendeurs;
```

### Full Outer Join
Retourne tous les enregistrements quand il y a une correspondance dans la table de gauche **ou** de droite.
*Note : Non supporté directement par MySQL (utilisez `UNION`), mais présent dans PostgreSQL et SQL Server.*

```sql
SELECT clients.nom, commandes.id
FROM clients
FULL OUTER JOIN commandes ON clients.id = commandes.client_id;
```

---

## 2. Les Fonctions de Fenêtrage (Window Functions)

Les fonctions de fenêtrage permettent d'effectuer des calculs sur un ensemble de lignes (une "fenêtre") liées à la ligne courante. Contrairement à `GROUP BY`, elles **ne réduisent pas** le nombre de lignes : vous conservez le détail des lignes tout en ajoutant des informations agrégées.

### 2.1. Anatomie d'une Window Function

La syntaxe standard est la suivante :

```sql
<fonction_window> OVER (
    [PARTITION BY colonne1, colonne2 ...]
    [ORDER BY colonne3 ...]
    [FRAME_CLAUSE]
)
```

1.  **`PARTITION BY`** (Optionnel) : Divise les données en groupes (comme des `GROUP BY` distincts). Les calculs redémarrent pour chaque partition.
2.  **`ORDER BY`** (Optionnel mais souvent requis) : Définit l'ordre des lignes à l'intérieur de la partition. Essentiel pour les classements (`RANK`) et les calculs glissants.
3.  **`FRAME_CLAUSE`** (Optionnel) : Définit précisément quelles lignes constituent la fenêtre par rapport à la ligne courante (ex: "les 3 lignes précédentes").

---

### 2.2. Les Fonctions de Classement (Ranking)

Elles permettent d'attribuer un rang à chaque ligne. La différence réside dans la gestion des ex-aequo.

| Fonction | Comportement avec Ex-aequo | Exemple de Résultat |
| :--- | :--- | :--- |
| **`ROW_NUMBER()`** | Pas d'ex-aequo. Numérotation unique (1, 2, 3, 4). | 1, 2, 3, 4 |
| **`RANK()`** | Ex-aequo autorisés, mais laisse des "trous" dans la suite. | 1, 2, 2, 4 |
| **`DENSE_RANK()`** | Ex-aequo autorisés, **sans** trous (le rang suivant suit l'ordre logique). | 1, 2, 2, 3 |
| **`NTILE(N)`** | Divise les lignes en N groupes égaux ( Buckets ). | 1, 1, 2, 2, 3, 3 (pour N=3) |

**Exemple :** Trouver le top 3 des ventes par département, avec gestion des ex-aequo.

```sql
SELECT 
    employe,
    departement,
    montant_vente,
    DENSE_RANK() OVER (
        PARTITION BY departement  -- Le classement recommence pour chaque département
        ORDER BY montant_vente DESC
    ) as rang_dept
FROM ventes;
```

---

### 2.3. Les Fonctions de Décalage (Offset)

Ces fonctions permettent d'accéder à des données d'autres lignes sans faire de `SELF JOIN`.

*   **`LAG(colonne, [offset], [default])`** : Valeur de la ligne **précédente**.
*   **`LEAD(colonne, [offset], [default])`** : Valeur de la ligne **suivante**.
*   **`FIRST_VALUE()`** : Première valeur de la fenêtre.
*   **`LAST_VALUE()`** : Dernière valeur de la fenêtre.

**Cas d'usage :** Calculer la croissance des ventes d'un mois sur l'autre (MoM - Month over Month).

```sql
SELECT 
    mois,
    ventes,
    LAG(ventes) OVER (ORDER BY mois) as ventes_mois_precedent,
    ventes - LAG(ventes) OVER (ORDER BY mois) as variation
FROM stats_mensuelles
ORDER BY mois;
```

---

### 2.4. Les Agrégats Fenêtrés (Window Aggregates)

On peut utiliser `SUM`, `AVG`, `MIN`, `MAX`, `COUNT` comme fonctions de fenêtrage.

#### A. Le "Running Total" (Cumul glissant)
C'est le calcul d'un total qui s'incrémente au fur et à mesure.

```sql
SELECT 
    date_commande,
    montant,
    SUM(montant) OVER (
        ORDER BY date_commande
        -- Par défaut : ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as cumul_total
FROM commandes;
```

#### B. La Moyenne Mobile (Moving Average)
C'est ici que le **`FRAME_CLAUSE`** devient crucial. On ne veut pas le total depuis le début, mais la moyenne sur les X dernières lignes.

```sql
SELECT 
    date,
    temperature,
    AVG(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW -- Moyenne sur la ligne actuelle et les 2 avant
    ) as moyenne_mobile_3_jours
from meteo
ORDER BY date;
```

**Comprendre les Frames :**
*   `UNBOUNDED PRECEDING` : Le début de la partition.
*   `CURRENT ROW` : La ligne actuelle.
*   `UNBOUNDED FOLLOWING` : La fin de la partition.
*   `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` : La ligne avant + actuelle + après.

---

## 3. Les CTE (Common Table Expressions)

Les CTE (ou clauses `WITH`) permettent de créer des résultats temporaires nommés pour rendre les requêtes plus lisibles et maintenables.

### Structure basique
```sql
WITH top_vendeurs AS (
    SELECT id, nom
    FROM employes
    WHERE ventes > 100000
)
SELECT v.nom, c.montant
FROM top_vendeurs v
JOIN commandes c ON v.id = c.vendeur_id;
```

### Avantages
*   **Lisibilité** : Remplace les sous-requêtes imbriquées difficiles à lire.
*   **Réutilisation** : Une CTE peut être référencée plusieurs fois dans la requête principale.

---

## 4. Les CTE Récursives

Permet de requêter des données hiérarchiques (arbres, graphes) comme des organigrammes ou des systèmes de fichiers.

### Exemple : Organigramme
```sql
WITH RECURSIVE hierarchy AS (
    -- 1. Point de départ (Le patron)
    SELECT id, nom, manager_id, 1 as niveau
    FROM employes
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- 2. Partie récursive (Les subordonnés)
    SELECT e.id, e.nom, e.manager_id, h.niveau + 1
    FROM employes e
    JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy;
```

---

## 5. Agrégation Avancée (Grouping Sets)

Les fonctions de groupement permettent de générer plusieurs niveaux d'agrégation en une seule requête, remplaçant plusieurs `UNION ALL`.

### ROLLUP
Génère des sous-totaux et un total général.

```sql
SELECT 
    region, 
    produit, 
    SUM(ventes) as total_ventes
FROM sales
GROUP BY ROLLUP (region, produit);
-- Résultat : Total par (Region, Produit) + Total par Region + Total Général
```

### CUBE
Génère tous les sous-totaux possibles (super-cube).

```sql
SELECT 
    region, 
    produit, 
    SUM(ventes) as total_ventes
FROM sales
GROUP BY CUBE (region, produit);
-- Résultat : Tous les combos possibles + totaux
```

### GROUPING SETS
Permet de définir exactement quels groupes vous voulez, sans tout calculer.

```sql
SELECT region, produit, SUM(ventes)
FROM sales
GROUP BY GROUPING SETS (
    (region),      -- Total par région
    (produit)      -- Total par produit
);
```

---

## 6. Optimisation Data Warehouse (Partition & Cluster)

Dans un Data Warehouse (BigQuery, Snowflake, Redshift, Azure Synapse), les données sont stockées en colonne (Columnar Storage). Le coût principal est le nombre de données **scannées (lues)**. Optimiser signifie réduire ce scan.

### 6.1. Le Partitionnement (Partitioning)
Le partitionnement divise physiquement une grande table en segments plus petits, basés sur la valeur d'une colonne (souvent une date).

**But :** Permettre au moteur de requête de sauter (pruning) les segments qui ne contiennent pas les données recherchées.

#### Types de partitionnement
1.  **Par temps (Time-based)** : Le plus courant. Partitionnement par jour, mois ou année.
2.  **Par plage d'entiers (Integer Range)** : Pour des IDs spécifiques.
3.  **Par temps d'ingestion (Ingestion time)** : Basé sur le moment où les données sont arrivées dans la table (ex: `_PARTITIONDATE` dans BigQuery).

#### Bonnes pratiques
*   **Toujours** partitionner les tables de faits (logs, ventes, événements) sur une colonne de date (`DATE` ou `TIMESTAMP`).
*   Ne pas partitionner sur une colonne à forte cardinalité avec peu de valeurs par partition (ex: "Pays" si vous avez 200 pays et 10 milliards de lignes, c'est bien. Si vous avez 195 pays et 1000 lignes, c'est inutile).

**Exemple BigQuery :**
```sql
-- Création d'une table partitionnée par jour
CREATE TABLE ma_project.dataset.ventes (
    id INT64,
    client_id INT64,
    montant FLOAT64,
    date_vente DATE
)
PARTITION BY date_vente
OPTIONS (
    partition_expiration_days = 365 -- Nettoyage auto
);

-- Requête optimisée (Scan seulement la partition du 01/01/2023)
SELECT * FROM ma_project.dataset.ventes
WHERE date_vente = '2023-01-01';
```

---

### 6.2. Le Clustering (Clustering)
Le clustering trie les données *à l'intérieur* de chaque partition selon les valeurs d'une ou plusieurs colonnes.

**But :** Si vous filtrez sur une colonne qui n'est pas la colonne de partitionnement, ou si vous faites un `GROUP BY` fréquent, le clustering permet au moteur de ne lire que les blocs de stockage pertinents.

**Différence clé :**
*   *Partition* : Découpe la table en gros blocs (Ex: Livres par année).
*   *Cluster* : Trie les livres à l'intérieur de l'année par Auteur, puis par Titre.

#### Quand utiliser le Clustering ?
*   Sur des colonnes fréquemment utilisées dans les filtres (`WHERE`) ou les `GROUP BY` / `ORDER BY`.
*   Sur des colonnes avec une cardinalité faible ou moyenne (ex: "ID Client", "Région", "Statut").
*   Sur des tables déjà partitionnées (le clustering s'applique *dans* chaque partition).

**Exemple BigQuery :**
```sql
-- Création avec clustering
CREATE OR REPLACE TABLE ma_project.dataset.evenements (
    event_date DATE,
    user_id INT64,
    event_type STRING,
    details STRING
)
PARTITION BY event_date
CLUSTER BY user_id, event_type; -- Trie par user_id, puis par event_type dans chaque jour

-- Requête optimisée : 
-- BigQuery scanne uniquement la partition '2023-01-01', 
-- et dans cette partition, uniquement les blocs contenant 'click'.
SELECT * 
FROM ma_project.dataset.evenements
WHERE event_date = '2023-01-01'  -- Utilise le Partitionnement
  AND event_type = 'click';      -- Utilise le Clustering
```

### 6.3. Résumé des Stratégies Coûts

| Scénario | Stratégie Recommandée | Impact |
| :--- | :--- | :--- |
| Table de Logs (400M lignes/jour) | `PARTITION BY date` + `CLUSTER BY user_id` | Réduit le scan aux dates et utilisateurs ciblés. |
| Table de Référence (Petite, < 1Go) | Pas de partition, pas de cluster. | Le coût de lecture est déjà minime. |
| Requête récente (Dernières 24h) | Filtrer sur la pseudo-colonne d'ingestion (`_PARTITIONDATE`). | Rapide, évite de scanner l'historique. |

---

## 7. Manipulation de Données Complexes

### Upsert (INSERT ou UPDATE)
Insérer une ligne si elle n'existe pas, ou la mettre à jour si elle existe. La syntaxe varie selon les SGBD.

*PostgreSQL / SQLite :*
```sql
INSERT INTO clients (id, nom)
VALUES (1, 'Alice')
ON CONFLICT (id) 
DO UPDATE SET nom = EXCLUDED.nom;
```

*MySQL :*
```sql
INSERT INTO clients (id, nom)
VALUES (1, 'Alice')
ON DUPLICATE KEY UPDATE nom = 'Alice';
```

### Traitement du JSON
La plupart des SGBD modernes supportent le JSON nativement.

```sql
-- Sélectionner une valeur dans un objet JSON (Postgres/BigQuery)
SELECT data->>'prenom' as prenom
FROM users
WHERE data->>'actif' = 'true';

-- Flattener un tableau JSON (BigQuery)
SELECT *
FROM table,
UNNEST(json_column.array) as element
```

---

## 8. Gestion des Transactions

Le SQL n'est pas seulement question de lecture, c'est aussi garantir l'intégrité des données (propriétés ACID).

### Contrôle de base
```sql
BEGIN; -- Début de la transaction

UPDATE compte SET solde = solde - 100 WHERE id = 1;
UPDATE compte SET solde = solde + 100 WHERE id = 2;

COMMIT; -- Valider les changements
-- ROLLBACK; -- Annuler en cas d'erreur
```

### Isolation des transactions
Le niveau d'isolation détermine la visibilité des changements entre transactions simultanées.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- ou SERIALIZABLE (le plus strict, évite les phénomènes de lecture sale, etc.)
```

---

## 📝 Conclusion

Ce cours couvre les briques essentielles pour passer de niveau en SQL, de l'analytique avancée avec les Window Functions jusqu'à l'optimisation critique pour les Data Warehouses modernes (Partitionnement/Clustering).

La clé de la maîtrise est la pratique : testez vos plans d'exécution, mesurez vos coûts de scan sur BigQuery, et structurez vos données pour vos requêtes les plus fréquentes.

## 🚀 À vous de jouer
Fork ce repo, ajoutez vos propres exemples et créez des challenges SQL pour la communauté !

---
*Document généré pour la formation Advanced SQL & Data Engineering.*
