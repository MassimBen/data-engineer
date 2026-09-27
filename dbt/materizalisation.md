# dbt Learn — Materializations

## Résumé

Une **materialization** définit la manière dont dbt crée et conserve le résultat d’un modèle dans l’entrepôt de données. Le choix influence notamment les performances des requêtes, le temps de construction et l’espace de stockage utilisé. dbt propose notamment les materializations `view`, `table`, `incremental`, `ephemeral` et `materialized_view`. Leur disponibilité et leur comportement peuvent varier selon l’entrepôt de données utilisé. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/sql-models?version=2&utm_source=openai))

## Les principaux types

| Materialization | Fonctionnement | À utiliser lorsque… |
|---|---|---|
| `view` | Crée une vue qui exécute la requête du modèle lorsqu’on la consulte. | On veut commencer simplement ou que les données doivent rester fraîches à chaque lecture. |
| `table` | Stocke le résultat du modèle sous forme de table. dbt la reconstruit lors de l’exécution. | Les requêtes en aval sont lentes ou le modèle est coûteux à recalculer à chaque lecture. |
| `incremental` | Ajoute ou met à jour les données traitées depuis la dernière exécution, au lieu de reconstruire toute la table. | Le modèle traite de gros volumes de données et peut identifier les nouvelles lignes à charger. |
| `ephemeral` | Ne crée pas d’objet distinct dans l’entrepôt : dbt intègre le SQL du modèle dans les modèles qui le référencent. | Le modèle sert de composant intermédiaire simple, utilisé par d’autres modèles. |
| `materialized_view` | Crée une vue matérialisée, dont le résultat est conservé selon les fonctionnalités de l’entrepôt. | L’entrepôt prend en charge cette fonctionnalité et qu’elle convient au besoin. |

## Configurer une materialization

On peut configurer la materialization directement dans le fichier du modèle :

```sql
{{ config(materialized='table') }}

select
    customer_id,
    count(*) as order_count
from {{ ref('stg_orders') }}
group by customer_id
```

Ou pour plusieurs modèles, dans `dbt_project.yml` :

```yaml
models:
  mon_projet:
    +materialized: view
    marts:
      +materialized: table
```

La configuration définie dans le modèle peut remplacer celle héritée du projet ou de son dossier. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/sql-models?version=2&utm_source=openai))

## Modèles incrémentaux

Un modèle incrémental évite de retraiter toutes les données à chaque exécution. Il faut notamment définir une logique permettant de sélectionner les nouvelles données ou celles qui ont changé.

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id'
    )
}}

select
    order_id,
    customer_id,
    updated_at
from {{ ref('stg_orders') }}

{% if is_incremental() %}
where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

`is_incremental()` permet d’appliquer un filtre lors d’une exécution incrémentale. Le SQL doit toutefois rester valide lorsque cette condition n’est pas satisfaite, par exemple lors de la première création ou d’une reconstruction complète. La clé `unique_key` aide dbt à reconnaître les lignes à mettre à jour, si elle est nécessaire pour la stratégie choisie. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/incremental-models?utm_source=openai))

## Choisir la bonne materialization

- Commencer avec une **vue** peut être adapté pour développer simplement.
- Passer à une **table** si les requêtes en aval sont trop lentes.
- Choisir **incremental** lorsque le volume ou le temps de traitement justifie de ne traiter que les données nouvelles ou modifiées.
- Utiliser **ephemeral** pour intégrer un modèle intermédiaire dans ses modèles dépendants, sans créer de relation séparée.
- Choisir une **vue matérialisée** seulement si l’entrepôt et le cas d’usage s’y prêtent.

La documentation dbt recommande de commencer par des vues et d’envisager des tables lorsque des problèmes de performance apparaissent. ([docs.getdbt.com](https://docs.getdbt.com/guides/bigquery?step=1&utm_source=openai))

## À retenir

- La materialization détermine **comment le résultat d’un modèle est stocké ou rendu disponible**.
- Le choix se fait modèle par modèle et peut aussi être défini par dossier dans `dbt_project.yml`.
- Les modèles incrémentaux nécessitent une logique de sélection des données à traiter et doivent être testés avec attention.
- Les capacités exactes peuvent dépendre de l’adaptateur et de l’entrepôt utilisés.

## Commandes utiles

```bash
dbt run                         # Exécuter les modèles
dbt run --full-refresh          # Reconstruire entièrement les modèles concernés
dbt compile                     # Vérifier le SQL compilé
```

> `--full-refresh` est particulièrement utile lorsqu’on veut reconstruire un modèle incrémental depuis zéro.

## Ressources

- [Documentation dbt — SQL models et configurations](https://docs.getdbt.com/docs/build/sql-models)
- [Documentation dbt — Incremental models](https://docs.getdbt.com/docs/build/incremental-models)

### Sources

- Documentation officielle dbt sur les modèles et les materializations. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/sql-models?version=2&utm_source=openai))
- Documentation officielle dbt sur les modèles incrémentaux. ([docs.getdbt.com](https://docs.getdbt.com/docs/build/incremental-models?utm_source=openai))
- Guide officiel dbt indiquant quand envisager le passage d’une vue à une table. ([docs.getdbt.com](https://docs.getdbt.com/guides/bigquery?step=1&utm_source=openai))
