# dbt Learn — Advanced Testing

## Cours détaillé pour débuter comme Data Engineer junior

> Ce cours explique comment concevoir, écrire et diagnostiquer des tests dans un projet dbt. Il couvre les tests intégrés, les tests singuliers et génériques, les configurations avancées, les tests unitaires et le package **`dbt-expectations`**.  
> Les exemples sont faits pour être adaptés à ton entrepôt et à ta version de dbt ; les versions récentes de dbt peuvent notamment utiliser la syntaxe `arguments:` dans les fichiers YAML.

---

## 1. Pourquoi tester les données ?

Un pipeline peut s’exécuter sans erreur tout en produisant des données fausses. Par exemple :

- un modèle de commandes contient plusieurs lignes par commande ;
- un montant est négatif à cause d’une mauvaise conversion ;
- un client référencé par une commande n’existe pas dans la dimension client ;
- une nouvelle valeur apparaît dans `status` et casse un rapport ;
- le flux de données s’arrête, mais les anciens modèles restent accessibles.

Les tests transforment ces hypothèses en contrôles exécutables.

Exemples d’hypothèses :

- « `order_id` est unique. »
- « `customer_id` est toujours renseigné. »
- « Chaque commande correspond à un client connu. »
- « Le montant d’une commande est positif ou nul. »
- « Des données récentes arrivent chaque jour. »

Un test dbt n’est pas une preuve absolue que les données sont parfaites. Il vérifie **une règle précise**, sur **un périmètre précis**.

---

## 2. Le principe des data tests

Un **data test** est une requête qui recherche les données incorrectes.

- La requête retourne **zéro ligne** : le test passe.
- Elle retourne **une ou plusieurs lignes** : elle a trouvé des violations.

Par exemple, pour trouver les commandes au montant négatif :

```sql
select *
from {{ ref('fct_orders') }}
where order_total < 0
```

Cette requête ne retourne pas les commandes correctes : elle retourne précisément celles qui enfreignent la règle.

### Ce que dbt fait avec un test

À haut niveau, dbt :

1. compile le test ;
2. exécute la requête dans l’entrepôt ;
3. compte ou récupère les lignes en échec ;
4. marque le test comme réussi, avertissement ou échec selon sa configuration.

Les tests intégrés de dbt sont des tests génériques déclarés dans le YAML du projet. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

---

## 3. Comprendre la granularité avant de tester

La **granularité** indique ce que représente une ligne du modèle.

Exemples :

| Modèle | Une ligne représente… | Clé candidate |
|---|---|---|
| `fct_orders` | une commande | `order_id` |
| `fct_order_items` | un article d’une commande | `order_id` + `product_id` |
| `dim_customers` | un client | `customer_id` |
| `agg_daily_revenue` | une journée de revenus | `order_date` |

Avant d’ajouter `unique`, demande-toi :

> « Une valeur unique à quel niveau ? »

Dans `fct_order_items`, `order_id` peut se répéter. Tester cette colonne seule avec `unique` serait probablement incorrect. La clé métier pourrait plutôt être la combinaison `order_id` + `product_id`.

---

## 4. Les tests génériques intégrés à dbt

Les quatre tests génériques de base sont `unique`, `not_null`, `accepted_values` et `relationships`. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

### 4.1 `unique`

Vérifie qu’une valeur ne se répète pas.

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_id
        data_tests:
          - unique
```

À utiliser lorsque la colonne représente une clé unique à la granularité du modèle.

### 4.2 `not_null`

Vérifie que les valeurs ne sont pas nulles.

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_id
        data_tests:
          - not_null
```

Une clé primaire logique est généralement testée avec **les deux** règles :

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
```

`unique` et `not_null` vérifient deux propriétés différentes : une colonne peut être unique parmi ses valeurs renseignées, mais contenir des `NULL`.

### 4.3 `accepted_values`

Vérifie qu’une colonne ne contient que des valeurs définies.

```yaml
models:
  - name: fct_orders
    columns:
      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values:
                  - placed
                  - shipped
                  - completed
                  - returned
```

À utiliser pour les statuts, catégories ou codes censés appartenir à une liste contrôlée.

**Point d’attention :** si la liste doit évoluer, une nouvelle valeur fera échouer le test. C’est parfois exactement ce que l’on veut — être alerté — mais l’équipe doit ensuite décider si la valeur est valide et mettre à jour le contrat.

### 4.4 `relationships`

Vérifie que chaque valeur d’une colonne correspond à une valeur présente dans une autre ressource.

```yaml
models:
  - name: fct_orders
    columns:
      - name: customer_id
        data_tests:
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_id
```

Cela signifie que les `customer_id` de `fct_orders` doivent exister dans `dim_customers`.

**Important :** une relation n’est pas la même chose qu’une contrainte `not_null`. Si les valeurs nulles sont interdites, ajoute aussi `not_null`.

```yaml
      - name: customer_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_id
```

---

## 5. Le fichier YAML et l’organisation des tests

On déclare souvent les tests à côté de la documentation des modèles, dans un fichier comme `models/marts/schema.yml`.

```yaml
version: 2

models:
  - name: fct_orders
    description: "Une ligne par commande."
    columns:
      - name: order_id
        description: "Identifiant unique de la commande."
        data_tests:
          - not_null
          - unique

      - name: customer_id
        description: "Identifiant du client."
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_id

      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values: ['placed', 'shipped', 'completed', 'returned']
```

> Si ton projet ou ta version de dbt utilise une syntaxe différente pour les arguments des tests, suis la documentation correspondant à cette version. L’important ici est la structure : un test est attaché à une ressource, à une colonne ou à un modèle.

---

## 6. Les tests singuliers

Un **test singulier** est un fichier SQL écrit pour une règle particulière. Il se trouve généralement dans le dossier `tests/`.

### 6.1 Exemple : vérifier qu’un montant n’est pas négatif

Fichier `tests/assert_order_total_non_negative.sql` :

```sql
select
    order_id,
    order_total
from {{ ref('fct_orders') }}
where order_total < 0
```

Si la requête retourne des lignes, le test échoue.

### 6.2 Ajouter du contexte utile

Un test doit aider à diagnostiquer le problème. Retourne les identifiants et les colonnes utiles :

```sql
select
    order_id,
    customer_id,
    order_date,
    order_total,
    status
from {{ ref('fct_orders') }}
where order_total < 0
```

### 6.3 Quand écrire un test singulier ?

Choisis un test singulier lorsque :

- la règle est spécifique à un modèle ;
- elle contient une logique SQL métier particulière ;
- tu ne penses pas la réutiliser telle quelle ailleurs ;
- le test est plus clair en SQL qu’en paramètres YAML.

### 6.4 Attention aux `NULL` en SQL

Les comparaisons SQL avec `NULL` ne fonctionnent pas comme une comparaison normale. Par exemple, `order_total < 0` ne sélectionne pas les lignes où `order_total` est `NULL`.

Si la règle exige aussi que le montant soit renseigné, teste cela explicitement, par exemple avec un test `not_null` sur la colonne ou une autre condition adaptée au besoin.

---

## 7. Les tests génériques personnalisés

Un **test générique personnalisé** est une règle SQL paramétrable et réutilisable. Il est défini avec `{% test ... %}` dans un fichier du dossier `tests/`.

### 7.1 Créer un test `is_non_negative`

Fichier `tests/generic/is_non_negative.sql` :

```sql
{% test is_non_negative(model, column_name) %}

select
    *
from {{ model }}
where {{ column_name }} < 0

{% endtest %}
```

Ici :

- `model` représente la relation testée ;
- `column_name` représente la colonne testée.

### 7.2 L’utiliser sur une colonne

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_total
        data_tests:
          - is_non_negative
```

Le même test peut ensuite s’appliquer à d’autres modèles ou à d’autres colonnes.

### 7.3 Quand créer un test générique ?

Un test générique est adapté lorsque :

- la même règle revient plusieurs fois ;
- elle peut être exprimée avec quelques paramètres ;
- les équipes ont besoin de l’appliquer de manière cohérente.

### 7.4 Singulier ou générique : comment choisir ?

| Situation | Bon point de départ |
|---|---|
| Règle particulière à une table | Test singulier |
| Règle réutilisable sur plusieurs colonnes | Test générique |
| Vérification courante prise en charge par dbt | Test intégré |
| Vérification déjà fournie par un package | Test du package, après validation |

Ne crée pas un test générique simplement pour éviter d’écrire quelques lignes SQL. Commence par t’assurer que la règle métier est comprise et correctement testée.

---

## 8. Configurer les tests : sévérité, seuils et filtres

Les options et la syntaxe exacte peuvent dépendre de la version de dbt et du test utilisé. Vérifie la documentation et la compatibilité dans ton projet.

### 8.1 `severity`

La sévérité permet de traiter certaines violations comme des avertissements plutôt que comme des échecs bloquants.

Exemple de configuration sur un test :

```yaml
data_tests:
  - unique:
      config:
        severity: warn
```

**Comment raisonner :**

- `error` convient aux règles importantes qui doivent bloquer la suite ;
- `warn` peut convenir à un contrôle informatif ou à une règle temporairement non bloquante.

Ne transforme pas tous les échecs en avertissements : tu risques de ne plus remarquer les vrais problèmes.

### 8.2 `warn_if` et `error_if`

Ces options servent, selon la configuration dbt utilisée, à fixer des seuils de résultat auxquels le test produit un avertissement ou une erreur.

Exemple illustratif :

```yaml
data_tests:
  - unique:
      config:
        severity: error
        error_if: "> 10"
        warn_if: "> 0"
```

L’intention pourrait être : avertir dès qu’il existe des doublons, mais ne bloquer qu’au-delà d’un seuil. **Ne copie pas cet exemple sans vérifier le comportement exact et le besoin métier** : un petit nombre de doublons peut déjà être critique.

### 8.3 `where`

Un filtre `where` permet de tester une population limitée lorsqu’un test n’a de sens que sur une partie des données.

Exemple illustratif :

```yaml
data_tests:
  - not_null:
      config:
        where: "order_date >= '2025-01-01'"
```

On pourrait choisir de contrôler uniquement les commandes récentes si les données historiques sont incomplètes. Mais il faut documenter la raison du filtre : sinon, il peut masquer une partie des anomalies.

### 8.4 `store_failures`

`store_failures` peut permettre de conserver les lignes en échec dans l’entrepôt plutôt que de les consulter uniquement dans le résultat du test.

```yaml
data_tests:
  - is_non_negative:
      config:
        store_failures: true
```

Cela peut faciliter l’investigation, mais les données enregistrées, leur emplacement et leur durée de conservation dépendent de la configuration de ton projet. Les échecs peuvent contenir des données sensibles : respecte les règles de sécurité de l’entreprise.

---

## 9. Le package `dbt-expectations`

### 9.1 À quoi sert-il ?

**`dbt-expectations`** est un package qui étend la collection de tests disponibles dans dbt, avec des contrôles inspirés de Great Expectations. Il permet notamment d’écrire des tests de plages de valeurs, de fraîcheur, de volumes, de structure, de comparaisons et de propriétés statistiques. Le dépôt actuellement maintenu est celui de `metaplane/dbt-expectations`. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

Un package n’est pas une raison d’ajouter des tests au hasard : il faut toujours décider quelle règle métier on veut contrôler.

### 9.2 Installer le package

Dans `packages.yml` :

```yaml
packages:
  - package: metaplane/dbt_expectations
    version: "0.10.9"
```

La version `0.10.9` figure dans le README du dépôt consulté ; avant de l’utiliser, vérifie la version publiée la plus récente et sa compatibilité avec ton projet. Le dépôt indique une prise en charge de dbt 1.7 ou plus et documente plusieurs adaptateurs. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

Puis récupère les dépendances :

```bash
dbt deps
```

Et vérifie que dbt les reconnaît :

```bash
dbt parse
```

### 9.3 Tester qu’une valeur se trouve dans un intervalle

Exemple : vérifier que le montant d’une commande est compris entre 0 et 10 000.

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_total
        data_tests:
          - dbt_expectations.expect_column_values_to_be_between:
              arguments:
                min_value: 0
                max_value: 10000
```

Le package propose également l’option `strictly` pour configurer le caractère strict ou inclusif de l’intervalle. Vérifie la documentation du test avant de fixer cette option. ([github.com](https://github.com/metaplane/dbt-expectations/blob/main/README.md?utm_source=openai))

> Une plage arbitraire crée du bruit. Un montant maximal doit venir d’une hypothèse métier ou d’une analyse de données, pas d’un nombre choisi au hasard.

### 9.4 Vérifier la fraîcheur des données

On peut contrôler qu’une colonne de date contient des données récentes :

```yaml
models:
  - name: fct_orders
    columns:
      - name: order_date
        data_tests:
          - dbt_expectations.expect_row_values_to_have_recent_data:
              arguments:
                datepart: day
                interval: 1
```

Le test de fraîcheur doit correspondre à la fréquence réelle du flux. Si le modèle est normalement alimenté une fois par semaine, une attente quotidienne risque de produire de faux incidents. La documentation du package décrit également un test de fraîcheur par groupes. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

### 9.5 Vérifier qu’une table n’est pas vide

```yaml
models:
  - name: fct_orders
    data_tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          arguments:
            min_value: 1
```

C’est utile pour repérer une table vide ou une rupture majeure de volume. Pour des volumes naturellement variables, évite de choisir un maximum arbitraire.

### 9.6 Comparer des agrégats entre deux tables

Le package permet de comparer des agrégats d’une table avec ceux d’une autre, avec éventuellement un regroupement ou une tolérance.

```yaml
models:
  - name: fct_orders
    data_tests:
      - dbt_expectations.expect_table_aggregation_to_equal_other_table:
          arguments:
            expression: "count(*)"
            compare_model: ref('orders_reference')
            group_by:
              - order_date
```

Ce type de test peut aider à rapprocher un modèle transformé de sa référence. Il faut toutefois s’assurer que les deux tables couvrent la même population, les mêmes dates et les mêmes règles d’exclusion. Le package documente également des options comme une expression de comparaison, des regroupements distincts et des tolérances. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

### 9.7 Vérifier une clé composée

Si la clé est formée de plusieurs colonnes, `dbt-expectations` propose un test de combinaison unique :

```yaml
models:
  - name: fct_order_items
    data_tests:
      - dbt_expectations.expect_compound_columns_to_be_unique:
          arguments:
            column_list:
              - order_id
              - product_id
```

Avant de choisir cette règle, confirme qu’une commande ne peut réellement contenir le même produit qu’une seule fois. Dans certains systèmes, un même produit peut apparaître sur plusieurs lignes.

### 9.8 Quelques familles de tests disponibles

Le package répertorie notamment des contrôles pour :

- la forme de la table : présence de colonnes, nombre de colonnes ou de lignes ;
- les valeurs nulles, uniques, les types et les ensembles de valeurs ;
- les plages numériques et les motifs de chaînes ;
- les comparaisons entre colonnes ou entre tables ;
- la présence de données récentes ;
- certaines variations statistiques ou temporelles.

La disponibilité d’un test et ses arguments se vérifient dans le README du package, car les options diffèrent selon les expectations. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

### 9.9 Package ou test personnalisé ?

| Besoin | Choix raisonnable |
|---|---|
| Vérifier qu’une colonne n’est pas nulle | Test intégré `not_null` |
| Vérifier une règle courante déjà couverte par le package | `dbt-expectations` |
| Vérifier une règle propre au métier de l’entreprise | Test singulier ou générique personnalisé |
| Vérifier la logique SQL produite sur des entrées connues | Unit test dbt |

---

## 10. Les unit tests dbt : tester la logique SQL

Les **unit tests** ne sont pas la même chose que les data tests.

| Type | Ce qu’il vérifie principalement |
|---|---|
| Data test | Une propriété des données réellement produites |
| Unit test | Le résultat attendu d’une logique SQL, sur des entrées contrôlées |

Les unit tests dbt permettent d’exécuter la logique d’un modèle SQL sur de petits jeux de données définis à l’avance. Ils sont disponibles dans dbt à partir de la version 1.8 selon la documentation. Ils sont particulièrement utiles pour les transformations complexes : `CASE WHEN`, fenêtres, calculs de dates ou règles avec des cas limites. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/unit-tests?name=Fusion&version=2.0&utm_source=openai))

### Exemple de configuration illustrative

```yaml
unit_tests:
  - name: test_completed_order_has_completed_status
    model: fct_orders
    given:
      - input: ref('stg_orders')
        rows:
          - { order_id: 1, status: "completed", order_total: 100 }
    expect:
      rows:
        - { order_id: 1, status: "completed", order_total: 100 }
```

> La forme précise de l’entrée et du résultat dépend du modèle et de la version de dbt. Un unit test doit couvrir la logique que tu veux vérifier, pas simplement reproduire toutes les colonnes d’un modèle sans raison.

### Quand écrire un unit test ?

Par exemple :

- quand le SQL contient des branches `CASE WHEN` importantes ;
- quand un calcul de date ou une logique de fenêtre est délicat ;
- quand tu veux couvrir un cas limite difficile à rencontrer en production ;
- avant une refactorisation importante ;
- après un bug déjà observé pour éviter sa réapparition.

La documentation dbt recommande de concentrer ces tests sur la logique SQL et les cas où des données d’exemple permettent de vérifier le résultat attendu. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/unit-tests?name=Fusion&version=2.0&utm_source=openai))

### Quand utiliser plutôt un data test ?

Utilise un data test pour vérifier, après construction du modèle, que les données réelles respectent une règle : identifiants uniques, valeurs acceptées, relation entre tables, plage de valeurs, etc.

---

## 11. Tester les sources et la fraîcheur

Les modèles ne sont pas les seules ressources à surveiller. On peut également définir des tests et des contrôles de fraîcheur sur les sources.

Exemple de source documentée avec une date de fraîcheur :

```yaml
version: 2

sources:
  - name: raw
    tables:
      - name: orders
        loaded_at_field: loaded_at
        freshness:
          warn_after:
            count: 12
            period: hour
          error_after:
            count: 24
            period: hour
```

La logique est différente d’un simple test sur un modèle :

- un test de données vérifie une règle sur les lignes ;
- la fraîcheur d’une source vérifie si des données sont arrivées dans les délais attendus.

Les délais doivent refléter le fonctionnement réel de l’ingestion et les engagements de service du pipeline.

---

## 12. Exécuter les tests

### Exécuter les tests de données

```bash
dbt test
```

### Construire et tester les ressources

```bash
dbt build
```

`dbt build` permet d’exécuter les ressources sélectionnées dans l’ordre de dépendance, notamment les modèles et leurs tests. Les unit tests sont eux aussi exécutables dans le cadre du processus dbt selon les options et la version utilisées. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/unit-tests?name=Fusion&version=2.0&utm_source=openai))

### Sélectionner un modèle

```bash
dbt test --select fct_orders
```

### Sélectionner un test par son nom

```bash
dbt test --select assert_order_total_non_negative
```

### Quelques commandes utiles au quotidien

```bash
dbt deps
dbt parse
dbt test
dbt build --select fct_orders
```

Dans un projet réel, vérifie toujours la syntaxe des sélecteurs et les options compatibles avec la version installée.

---

## 13. Comment investiguer un test en échec

Un échec n’indique pas automatiquement que le test est mauvais ou que les données sont mauvaises. Il indique que la règle a détecté une violation — à toi de trouver sa cause.

### Étape 1 — Lire la règle

Demande-toi :

- Qu’est-ce que le test affirme ?
- Quelles lignes sont censées le faire échouer ?
- Le test est-il attaché au bon modèle et à la bonne colonne ?

### Étape 2 — Examiner le SQL compilé

dbt compile les modèles Jinja et les références `ref()` avant d’envoyer le SQL à l’entrepôt. Examiner le SQL compilé aide à comprendre ce qui a réellement été exécuté.

### Étape 3 — Récupérer les lignes en échec

Utilise les résultats affichés par dbt et, si c’est utile et configuré, stocke les échecs avec `store_failures`.

Cherche notamment :

- les identifiants des lignes ;
- les valeurs concernées ;
- leur date de chargement ;
- les champs qui éclairent le contexte métier.

### Étape 4 — Remonter dans la chaîne de transformation

Avec `ref()` et les sources, remonte de la table testée vers ses entrées. Essaie de déterminer si l’anomalie :

- était déjà présente dans la source ;
- vient d’une jointure ou d’un filtre ;
- résulte d’une conversion ou d’un calcul ;
- correspond à un changement légitime des règles métier.

### Étape 5 — Décider de l’action

Après analyse, la bonne action peut être :

- corriger les données amont ;
- corriger le modèle dbt ;
- mettre à jour la liste des valeurs acceptées ;
- préciser une exception métier documentée ;
- revoir le seuil de fraîcheur ou de volume ;
- corriger le test si l’assertion était erronée.

Ne modifie pas le test uniquement pour le faire passer avant d’avoir compris le problème.

---

## 14. Tester ses propres tests

Un test peut avoir une erreur logique et passer alors que les données sont mauvaises. Il faut donc parfois vérifier qu’il fait bien ce qu’on attend.

Pour un test personnalisé, essaie au minimum de raisonner sur deux cas :

| Données en entrée | Résultat attendu |
|---|---|
| Données qui respectent la règle | Test réussi |
| Données qui violent la règle | Test en échec |

C’est particulièrement important pour :

- les conditions avec `NULL` ;
- les filtres `where` ;
- les règles de plage ;
- les clés composées ;
- les calculs avec des dates ;
- les tests statistiques ;
- les jointures dans les tests singuliers.

Les unit tests peuvent également aider à vérifier des transformations SQL sur des exemples contrôlés, mais ils ne remplacent pas les data tests exécutés sur les données réelles. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/unit-tests?name=Fusion&version=2.0&utm_source=openai))

---

## 15. Exemple complet : construire les tests d’un modèle de commandes

Supposons que `fct_orders` contienne une ligne par commande.

### 15.1 Définir les règles

- `order_id` doit être renseigné et unique ;
- `customer_id` doit être renseigné et exister dans `dim_customers` ;
- `status` doit appartenir à une liste ;
- `order_total` doit être positif ou nul ;
- une donnée récente doit être disponible.

### 15.2 Déclarer les tests

```yaml
version: 2

models:
  - name: fct_orders
    description: "Une ligne par commande."

    columns:
      - name: order_id
        description: "Identifiant unique de la commande."
        data_tests:
          - not_null
          - unique

      - name: customer_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_id

      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values: ['placed', 'shipped', 'completed', 'returned']

      - name: order_total
        data_tests:
          - dbt_expectations.expect_column_values_to_be_between:
              arguments:
                min_value: 0

      - name: order_date
        data_tests:
          - dbt_expectations.expect_row_values_to_have_recent_data:
              arguments:
                datepart: day
                interval: 1

    data_tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          arguments:
            min_value: 1
```

### 15.3 Ajouter une règle métier particulière

Fichier `tests/assert_completed_orders_have_completion_date.sql` :

```sql
select
    order_id,
    status,
    completed_at
from {{ ref('fct_orders') }}
where status = 'completed'
  and completed_at is null
```

Cette règle est écrite en test singulier parce qu’elle exprime une condition métier propre à ce modèle.

### 15.4 Exécuter et interpréter

```bash
dbt test --select fct_orders
```

Si le test de relations échoue, par exemple, tu chercheras quels `customer_id` sont absents de `dim_customers`. Si le test de fraîcheur échoue, tu vérifieras le dernier chargement, la fréquence d’ingestion et le calendrier attendu.

---

## 16. Stratégie de tests adaptée à une équipe

Pour un modèle important, une stratégie raisonnable peut suivre ces couches :

1. **Tests de contrat et de colonnes** : présence, valeurs autorisées, types ou structure selon le besoin.
2. **Tests de clés** : unicité et absence de valeurs nulles pour les identifiants appropriés.
3. **Tests de relations** : cohérence des clés étrangères entre modèles.
4. **Tests métier** : montants, statuts, dates et règles spécifiques.
5. **Tests de fraîcheur et de volume** : continuité des arrivées de données.
6. **Unit tests** : logique SQL complexe ou cas limites.
7. **CI** : exécuter les tests pertinents lors des changements pour repérer rapidement une régression.

Chaque test a un coût d’exécution et un coût de maintenance. Choisis ceux qui protègent une règle importante et produisent une information exploitable.

---

## 17. Erreurs fréquentes des débutants

### 1. Tester `unique` sans connaître la granularité

Une colonne qui se répète n’est pas nécessairement erronée. Il faut savoir ce que représente une ligne.

### 2. Confondre unicité et absence de `NULL`

Une clé doit souvent être testée avec `unique` **et** `not_null`.

### 3. Penser que `relationships` interdit les `NULL`

Ce test vérifie surtout la correspondance des valeurs référencées ; ajoute `not_null` si la règle l’exige.

### 4. Mettre tous les tests en `warn`

Un avertissement peut être ignoré. Utilise une sévérité qui correspond réellement à la criticité de la règle.

### 5. Filtrer les tests sans expliquer pourquoi

Un `where` peut être légitime, mais il réduit la population testée. Documente son intention.

### 6. Copier une expectation sans vérifier ses arguments

Les tests de `dbt-expectations` ont des paramètres différents. Vérifie la documentation du test précis et la compatibilité avec ton projet. ([github.com](https://github.com/metaplane/dbt-expectations?utm_source=openai))

### 7. Choisir des seuils arbitraires

Une plage, un volume minimal ou un seuil de fraîcheur doit avoir une justification métier ou opérationnelle.

### 8. Écrire des tests impossibles à diagnostiquer

Un test qui échoue sans exposer de clé ni de contexte peut être difficile à corriger. Retourne les colonnes utiles ou utilise une configuration adaptée.

### 9. Confondre unit tests et data tests

Les premiers contrôlent une logique sur des entrées définies ; les seconds contrôlent des données réellement produites. Les deux répondent à des questions différentes. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/unit-tests?name=Fusion&version=2.0&utm_source=openai))

---

## 18. Parcours de travail conseillé pour un DE junior

Quand tu ajoutes un modèle ou modifies une transformation :

1. Écris en une phrase ce que représente une ligne.
2. Identifie sa clé ou sa clé composée.
3. Liste les règles importantes avec le métier ou ton équipe.
4. Ajoute les tests dbt intégrés adaptés.
5. Cherche dans `dbt-expectations` si une règle courante est déjà disponible.
6. Écris un test singulier pour les règles propres au modèle.
7. Rends un test générique si la règle doit être réutilisée.
8. Ajoute des unit tests si la logique SQL est complexe ou comporte des cas limites.
9. Lance les tests.
10. Analyse les échecs avant de modifier la règle ou d’ignorer l’erreur.
11. Documente les seuils, filtres et exceptions.

---

## 19. Checklist avant de valider tes tests

- [ ] La granularité du modèle est documentée ou comprise.
- [ ] Les clés sont testées de façon cohérente.
- [ ] Les `NULL` sont traités explicitement.
- [ ] Les relations entre modèles sont vérifiées lorsqu’elles sont nécessaires.
- [ ] Les valeurs acceptées reflètent une règle métier réelle.
- [ ] Les tests `dbt-expectations` utilisent les bons arguments.
- [ ] Les seuils de fraîcheur et de volume sont justifiés.
- [ ] Les tests personnalisés retournent des lignes utiles au diagnostic.
- [ ] Les échecs ont été examinés, pas seulement désactivés.
- [ ] Les unit tests couvrent la logique complexe lorsqu’ils apportent une vraie valeur.

---

## 20. À retenir

> Un bon test dbt commence par une règle claire. Il teste la bonne population, au bon niveau de granularité, et rend ses échecs compréhensibles.

Pour choisir le bon mécanisme :

- **test intégré** pour une règle standard ;
- **`dbt-expectations`** pour un contrôle courant disponible dans le package ;
- **test singulier** pour une règle SQL particulière ;
- **test générique personnalisé** pour une règle réutilisable ;
- **unit test** pour vérifier une logique de transformation sur des entrées contrôlées.

---

## Ressources

- [Documentation dbt — Data tests](https://docs.getdbt.com/docs/build/data-tests)
- [Documentation dbt — Unit tests](https://docs.getdbt.com/docs/build/unit-tests)
- [Dépôt `metaplane/dbt-expectations`](https://github.com/metaplane/dbt-expectations)
