# dbt : tests personnalisés et tests dans des packages

> **Objectif.** Cette fiche synthétise les notions du cours dbt Learn *Advanced Testing* consacrées aux **custom tests** et aux **tests fournis par des packages**. Les exemples sont conçus pour un projet dbt moderne ; adaptez la syntaxe YAML à la version de dbt utilisée.

## Préambule : comment dbt interprète un test

Un data test dbt est une requête SQL qui cherche les lignes **invalides**. La règle est donc contre-intuitive au début :

- la requête ne renvoie aucune ligne → **test réussi** ;
- la requête renvoie au moins une ligne → **test en échec** (les lignes renvoyées sont les contre-exemples) ;
- la requête renvoie un résultat non nul mais vide → le test passe.

Les tests peuvent être attachés à des modèles, colonnes, sources, seeds et snapshots. La commande principale est `dbt test`.

> **À retenir :** écrivez la requête comme une recherche de violations, pas comme une vérification qui renverrait `true` ou `false`.

---

# Partie 1 — Tests personnalisés (*Custom Tests*)

## 1. Tests singuliers (*singular tests*)

### Définition et emplacement

Un test singulier est une requête SQL écrite pour un cas précis. Elle est enregistrée dans le répertoire de tests du projet, généralement `tests/`, et porte l’extension `.sql`.

```text
dbt_project/
├── models/
├── macros/
├── tests/
│   ├── assert_orders_total_non_negative.sql
│   └── assert_payments_have_positive_amount.sql
└── dbt_project.yml
```

Le fichier SQL doit sélectionner les lignes qui enfreignent l’assertion. Il ne s’agit pas d’un modèle : dbt exécute la requête comme un test et n’en matérialise pas le résultat comme une table de production (sauf configuration explicite de stockage des échecs).

### Exemple : total de commande non négatif

```sql
-- tests/assert_orders_total_non_negative.sql
select
    order_id,
    total_amount
from {{ ref('fct_orders') }}
where total_amount < 0
```

S’il existe une commande avec `total_amount = -12.50`, la ligne est renvoyée et le test échoue. S’il n’existe aucune valeur négative, la requête est vide et le test réussit.

### Exemple : paiements positifs

```sql
-- tests/assert_payments_have_positive_amount.sql
select
    payment_id,
    order_id,
    amount,
    payment_status
from {{ ref('stg_payments') }}
where payment_status = 'completed'
  and (amount is null or amount <= 0)
```

### Exécuter un test singulier

Le nom sélectionnable est généralement le nom du fichier sans `.sql` :

```bash
dbt test --select assert_orders_total_non_negative
```

Commandes utiles pour plusieurs tests :

```bash
# Tous les tests singuliers dont le nom commence par assert_
dbt test --select "assert_*"

# Tous les tests d'un chemin
dbt test --select path:tests/
```

> **Bon réflexe :** renvoyez les colonnes utiles au diagnostic (`id`, date, statut, valeur observée), plutôt qu’un simple `select 1`.

## 2. Tests génériques personnalisés (*custom generic tests*)

### Définition

Un test générique est une requête paramétrée, réutilisable sur plusieurs modèles ou colonnes. Il est défini dans un bloc Jinja `{% test ... %}` et peut être appelé par son nom dans un fichier de propriétés YAML.

Les définitions peuvent se trouver dans :

- `tests/generic/` (dans le chemin de tests du projet) ;
- `macros/` (emplacement historique et toujours pris en charge).

Exemple de structure :

```text
tests/
└── generic/
    ├── test_is_even.sql
    └── test_positive_value.sql
```

Un test générique reçoit habituellement :

- `model` : la relation dbt testée (modèle, source, seed ou snapshot) ;
- `column_name` : la colonne testée lorsqu’il s’agit d’un test au niveau colonne.

### Exemple : `is_even`

```sql
-- tests/generic/test_is_even.sql
{% test is_even(model, column_name) %}

    select
        {{ column_name }} as value,
        count(*) as occurrences
    from {{ model }}
    where {{ column_name }} is not null
      and mod({{ column_name }}, 2) <> 0
    group by {{ column_name }}

{% endtest %}
```

La fonction `mod` peut varier selon le data warehouse ; par exemple, certains moteurs préfèrent `%` ou `{{ dbt_utils.safe_divide(...) }}`. Le principe reste le même : les valeurs impaires sont renvoyées comme lignes en échec.

Déclaration dans `models/schema.yml` :

```yaml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: item_count
        data_tests:
          - is_even
```

Dans des versions ou projets plus anciens, la clé historique `tests:` est encore courante. `data_tests:` est la forme explicite recommandée par les versions récentes.

### Exemple : `positive_value`

```sql
-- tests/generic/test_positive_value.sql
{% test positive_value(model, column_name) %}

    select
        *
    from {{ model }}
    where {{ column_name }} is null
       or {{ column_name }} <= 0

{% endtest %}
```

```yaml
version: 2

models:
  - name: fct_payments
    columns:
      - name: amount
        data_tests:
          - positive_value
```

### Exemple : test avec un argument de comparaison

Un test générique peut recevoir des arguments supplémentaires. Le test ci-dessous vérifie qu’une valeur est comprise dans un intervalle inclusif :

```sql
-- tests/generic/test_accepted_range.sql
{% test accepted_range(model, column_name, min_value=0, max_value=100) %}

    select
        *
    from {{ model }}
    where {{ column_name }} is null
       or {{ column_name }} < {{ min_value }}
       or {{ column_name }} > {{ max_value }}

{% endtest %}
```

Appel YAML avec la forme moderne imbriquée sous `arguments:` :

```yaml
version: 2

models:
  - name: fct_orders
    columns:
      - name: discount_percent
        data_tests:
          - accepted_range:
              arguments:
                min_value: 0
                max_value: 100
```

Selon la version de dbt et le réglage de compatibilité, la forme historique au niveau supérieur peut également être rencontrée :

```yaml
# Syntaxe historique, à utiliser seulement si votre version/projet la requiert
- accepted_range:
    min_value: 0
    max_value: 100
```

Les arguments standard `model` et `column_name` sont injectés par dbt : ils ne sont pas répétés dans le YAML.

### Valeurs par défaut des arguments

Dans la signature précédente, `min_value=0` et `max_value=100` sont des valeurs par défaut. Ainsi, ce test est valide :

```yaml
columns:
  - name: completion_percent
    data_tests:
      - accepted_range
```

Il utilise implicitement `[0, 100]`. Une instance peut surcharger une seule valeur :

```yaml
columns:
  - name: rating
    data_tests:
      - accepted_range:
          arguments:
            min_value: 1
            max_value: 5
```

Les valeurs par défaut rendent le test pratique, mais doivent rester explicites dans la documentation du projet : un défaut trop permissif peut masquer une erreur de configuration.

## 3. Configurer un test générique personnalisé

Un bloc `config()` dans la définition générique permet d’établir des défauts communs à toutes ses instances. Ici, les lignes en échec sont stockées et le test est traité comme un avertissement par défaut :

```sql
-- tests/generic/test_positive_value.sql
{% test positive_value(model, column_name) %}
    {{ config(
        severity='warn',
        store_failures=true
    ) }}

    select *
    from {{ model }}
    where {{ column_name }} is null
       or {{ column_name }} <= 0
{% endtest %}
```

Une instance peut remplacer cette configuration dans le YAML :

```yaml
columns:
  - name: amount
    data_tests:
      - positive_value:
          config:
            severity: error
            limit: 100
```

Les configurations peuvent aussi être placées dans `dbt_project.yml`. En cas de conflit, la configuration la plus spécifique (instance YAML, puis définition du test, puis projet) est généralement prioritaire. Vérifiez la documentation de la version de dbt utilisée pour les clés disponibles (`severity`, `warn_if`, `error_if`, `where`, `limit`, `store_failures`, etc.).

## 4. Singular vs générique : comparaison

| Critère | Test singulier | Test générique personnalisé |
|---|---|---|
| Forme | Requête SQL autonome | Bloc `{% test ... %}` paramétré |
| Emplacement courant | `tests/*.sql` | `tests/generic/` ou `macros/` |
| Réutilisation | Faible : conçu pour un cas précis | Forte : plusieurs modèles/colonnes |
| Appel | `dbt test --select nom_du_fichier` | Nom du test dans `data_tests:` / `tests:` |
| Arguments | Variables Jinja éventuellement écrites en dur | `model`, `column_name` et arguments propres |
| Cas idéal | Assertion métier unique, complexe ou transversale | Règle répétitive et standardisable |
| Diagnostic | La requête peut cibler une relation précise | Le même SQL produit les contre-exemples pour chaque instance |
| Configuration | `config()` dans le fichier SQL ou projet | `config()` dans le bloc, YAML de l’instance ou projet |

> **Règle de décision :** commencez par un test singulier lorsque l’assertion n’a qu’un consommateur. Convertissez-le en test générique lorsque la même logique doit être appliquée à plusieurs relations.

---

# Partie 2 — Tests dans les packages (*Tests in Packages*)

## 1. Pourquoi utiliser un package de tests ?

Les packages évitent de réécrire et de maintenir les mêmes assertions :

1. **DRY (*Don’t Repeat Yourself*)** : une définition réutilisée au lieu de nombreux fichiers similaires ;
2. **réutilisabilité** : une même suite sur les modèles, sources ou seeds ;
3. **maturité collective** : tests, macros SQL multi-warehouses et documentation entretenus par une communauté ;
4. **vitesse de mise en œuvre** : on exprime une intention dans le YAML sans développer le SQL à chaque fois.

Un package n’élimine pas la responsabilité de choisir les bonnes règles : un test générique mal paramétré peut être aussi trompeur qu’un test maison.

## 2. Installer un package : `packages.yml`, `dbt deps` et `dbt_packages`

Le fichier `packages.yml` (au même niveau que `dbt_project.yml`) décrit les dépendances :

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.1

  - package: metaplane/dbt_expectations
    version: 0.10.10
```

On peut aussi référencer un dépôt Git avec une révision explicite :

```yaml
packages:
  - git: "https://github.com/dbt-labs/dbt-utils.git"
    revision: 1.3.1
```

Puis installer ou mettre à jour les packages :

```bash
dbt deps
```

Par défaut, dbt place le code dans `dbt_packages/`. Ce répertoire est souvent ignoré par Git : on versionne `packages.yml` plutôt que le code copié. Après modification des versions, relancez `dbt deps` et contrôlez les dépendances transitives.

## 3. Package `dbt_expectations`

`dbt_expectations` est inspiré de **Great Expectations** : il expose des assertions de qualité de données exécutées directement dans le data warehouse via dbt. Le package propose notamment des tests de plage, de motif, d’ensemble de valeurs et de cardinalité.

> **Attention compatibilité :** l’implémentation et le dépôt maintenu peuvent évoluer. Consultez la version épinglée et sa documentation avant de déployer ; certaines versions s’appuient sur `dbt_utils`.

### Exemples YAML

```yaml
version: 2

models:
  - name: fct_orders
    data_tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          arguments:
            min_value: 1
            max_value: 100000000

    columns:
      - name: order_status
        data_tests:
          - dbt_expectations.expect_column_values_to_be_in_set:
              arguments:
                value_set: ['pending', 'paid', 'cancelled', 'refunded']

      - name: discount_percent
        data_tests:
          - dbt_expectations.expect_column_values_to_be_between:
              arguments:
                min_value: 0
                max_value: 100
                strictly: false

      - name: order_number
        data_tests:
          - dbt_expectations.expect_column_values_to_match_regex:
              arguments:
                regex: '^[A-Z]{2}-[0-9]{8}$'
```

Ces tests expriment respectivement :

| Test | Assertion |
|---|---|
| `expect_column_values_to_be_between` | Chaque valeur se trouve dans une plage (avec option de bornes strictes) |
| `expect_column_values_to_match_regex` | Chaque valeur respecte une expression régulière |
| `expect_table_row_count_to_be_between` | Le nombre de lignes de la table est dans un intervalle |
| `expect_column_values_to_be_in_set` | Chaque valeur appartient à un ensemble autorisé |

Les paramètres exacts (`regex`, `value_set`, `strictly`, gestion des `NULL`, etc.) dépendent de la version du package. Utilisez `dbt ls`, la documentation du package installé et ses fichiers dans `dbt_packages/` comme source de vérité.

## 4. Package `dbt_utils`

`dbt_utils` fournit des macros et des tests génériques largement utilisés. Exemples de tests disponibles selon les versions :

- `not_constant` : la colonne ne doit pas contenir une seule valeur distincte ;
- `expression_is_true` : une expression SQL doit être vraie pour chaque ligne ;
- `unique_combination_of_columns` : la combinaison de colonnes doit être unique ;
- `accepted_range` : les valeurs doivent se trouver dans une plage ;
- `mutually_exclusive_ranges` : des plages ou indicateurs ne doivent pas se chevaucher selon les règles configurées.

### Exemple d’utilisation

```yaml
version: 2

models:
  - name: fct_orders
    data_tests:
      - dbt_utils.unique_combination_of_columns:
          arguments:
            combination_of_columns:
              - customer_id
              - order_date

    columns:
      - name: order_status
        data_tests:
          - dbt_utils.not_constant

      - name: order_total
        data_tests:
          - dbt_utils.expression_is_true:
              arguments:
                expression: "order_total >= 0"
          - dbt_utils.accepted_range:
              arguments:
                min_value: 0
                max_value: 1000000

      - name: order_date
        data_tests:
          - not_null
```

Exemple indicatif de ranges mutuellement exclusifs — vérifiez la signature exacte de la version installée avant de le copier tel quel :

```yaml
models:
  - name: int_customer_segments
    data_tests:
      - dbt_utils.mutually_exclusive_ranges:
          arguments:
            lower_bound_column: min_spend
            upper_bound_column: max_spend
            gaps: not_allowed
            zero_length_range_allowed: false
```

### `arguments:` et versions de dbt

Les versions récentes de dbt recommandent de regrouper les arguments sous `arguments:`. Des projets plus anciens acceptent encore des arguments au niveau supérieur de l’instance. Le comportement est lié à la version de dbt et à des flags de compatibilité : standardisez votre style sur la version de votre environnement, puis validez avec `dbt parse`.

## 5. Écrire et partager son propre package de tests

Un package de tests est un projet dbt publiable. Une structure minimale peut être :

```text
mon-dbt-tests/
├── dbt_project.yml
├── README.md
├── macros/
│   └── test_helpers.sql
└── tests/
    └── generic/
        ├── test_positive_value.sql
        └── test_accepted_range.sql
```

Exemple de `dbt_project.yml` :

```yaml
name: mon_dbt_tests
version: 1.0.0
config-version: 2

profile: mon_dbt_tests

model-paths: [models]
macro-paths: [macros]
test-paths: [tests]

models:
  mon_dbt_tests:
    +materialized: ephemeral
```

Pour partager le package :

1. créez un dépôt GitHub avec README, licence, changelog et tests d’intégration ;
2. rendez les tests compatibles avec les warehouses visés (ou documentez les limites) ;
3. publiez une version/release et utilisez une révision immuable ;
4. ajoutez le package dans `packages.yml`, puis exécutez `dbt deps` ;
5. si le package est enregistré sur dbt Hub (lorsque disponible pour votre écosystème), utilisez sa référence officielle et sa matrice de compatibilité.

La qualité d’un package se mesure autant à la documentation des arguments et defaults qu’au SQL lui-même.

## 6. Bonnes pratiques

- **Épingler les versions** : utilisez `version` pour un package publié ou une `revision` Git immuable. Évitez une branche mouvante en production.
- **Contrôler la compatibilité** : vérifiez la version de dbt, l’adapter (Snowflake, BigQuery, Postgres, etc.) et les dépendances transitives (`dbt_utils`, notamment).
- **Valider après mise à jour** : lancez `dbt deps`, puis `dbt parse`, un sous-ensemble de `dbt test` et enfin la suite complète.
- **Ne pas surcharger** : tous les tests ne doivent pas être des erreurs bloquantes. Réservez les tests coûteux ou très volatils aux endroits où ils apportent une vraie valeur.
- **Documenter l’intention** : un nom de colonne ne suffit pas toujours ; ajoutez une description du test et de ses bornes.
- **Limiter le bruit** : utilisez `severity`, `warn_if`, `error_if`, `where` ou des tests ciblés lorsque le contexte métier le justifie.
- **Renvoyer des contre-exemples utiles** : les lignes d’échec doivent permettre d’identifier rapidement la cause.
- **Tester les tests** : testez un package sur des données volontairement valides et invalides avant de le partager.

---

# Points clés à retenir

1. Un test dbt renvoie les **lignes qui violent** une règle ; zéro ligne signifie succès.
2. Un test singulier est idéal pour une assertion spécifique ; un test générique est paramétré et réutilisable.
3. Les tests génériques se définissent dans `tests/generic/` ou `macros/`, puis se déclarent en YAML.
4. Les arguments supplémentaires se placent de préférence sous `arguments:` ; les defaults doivent être documentés.
5. `dbt_expectations` et `dbt_utils` accélèrent la mise en place de contrôles éprouvés, mais leurs versions et signatures doivent être vérifiées.
6. Épinglez les packages, vérifiez la compatibilité et évitez de multiplier les tests sans valeur opérationnelle.

# Commandes utiles — aide-mémoire

```bash
# Installer les packages déclarés dans packages.yml
dbt deps

# Vérifier que le projet et les tests se parsèment correctement
dbt parse

# Exécuter tous les tests
dbt test

# Exécuter un test singulier par son nom
dbt test --select assert_orders_total_non_negative

# Exécuter les tests d'un modèle
dbt test --select fct_orders

# Exécuter un test générique par son nom
dbt test --select is_even

# Sélectionner les tests génériques (sélecteur disponible selon la version dbt)
dbt test --select "test_type:generic"

# Exécuter les tests de données, en excluant les tests unitaires sur les versions qui les distinguent
dbt test --select "test_type:data"

# Tester les sources
dbt test --select "source:*"

# Afficher les ressources sélectionnées avant exécution
dbt ls --select "test_type:generic"

# Exécuter les modèles sélectionnés et leurs tests
dbt build --select fct_orders
```

> **Conseil final :** dans un pipeline CI, combinez `dbt deps`, `dbt parse`, une sélection de tests rapide sur les modèles modifiés, puis une suite complète planifiée. Cela donne un retour rapide sans sacrifier la couverture.

## Références

- [dbt — Writing custom generic tests](https://docs.getdbt.com/best-practices/writing-custom-generic-tests)
- [dbt — Data tests](https://docs.getdbt.com/docs/build/data-tests)
- [dbt — Packages](https://docs.getdbt.com/docs/build/packages)
- [dbt — Data test configurations](https://docs.getdbt.com/reference/data-test-configs)
- [dbt-utils — dépôt GitHub](https://github.com/dbt-labs/dbt-utils)
- [dbt-expectations — dépôt GitHub](https://github.com/metaplane/dbt-expectations)
