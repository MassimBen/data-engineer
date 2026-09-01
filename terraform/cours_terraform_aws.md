# Cours Terraform — de débutant à avancé pour Data Engineer et MLOps

> **Objectif.** Passer d’une infrastructure créée manuellement à une infrastructure reproductible, revue et automatisée, puis savoir la mettre au service de la donnée et du machine learning. Les exemples AWS sont explicitement signalés et restent adaptables à un autre cloud.
>
> **Version.** Ce cours évite les comportements dépendant d’une version précise. Vérifier toujours la documentation officielle, les versions disponibles et les contraintes `required_version`/`required_providers` du projet avant d’exécuter un exemple.

## Comment utiliser ce cours

Suivre les niveaux dans l’ordre, exécuter les exercices dans un compte de test et détruire les ressources de démonstration. Un `plan` est une proposition : il faut le lire avant tout `apply`. Les exemples marqués **AWS — facturable** peuvent engendrer des coûts (stockage, requêtes, logs, réseau ou exécution) ; définir un budget et des alertes.

---

## 1. Prérequis et installation

### Prérequis techniques

- ligne de commande, Git et YAML/JSON ;
- bases réseau : DNS, CIDR, sous-réseaux, routage, TLS ;
- bases cloud : identité, régions, stockage objet, compute ;
- notions SQL, Python et pipelines de données utiles pour les cas Data/ML ;
- comprendre qu’un fichier `.tf` est du code déclaratif, pas une suite de commandes impératives.

### Installation et premier projet

Installer Terraform depuis les canaux officiels, vérifier la signature/checksum si votre organisation l’exige, puis vérifier :

```bash
terraform version
terraform -help
```

Créer un dossier Git, par exemple `infra/`, et un fichier `versions.tf` :

```hcl
terraform {
  # Exemple de borne ; ajuster après vérification de la version officielle.
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

> Les contraintes ci-dessus sont un **exemple à vérifier** : elles ne garantissent pas qu’une version soit encore la meilleure ou disponible. Committer le fichier de verrouillage généré (`.terraform.lock.hcl`) pour les plateformes réellement supportées.

Installer le provider et préparer le répertoire :

```bash
terraform init
terraform fmt -check -recursive
```

Pour AWS, préférer un profil local, une identité fédérée ou un rôle CI. Ne jamais mettre une clé dans HCL, Git, une image Docker ou une sortie de log.

---

## 2. Infrastructure as Code et HCL

### Pourquoi IaC ?

L’IaC décrit l’état souhaité : revue par pull request, historique Git, reproductibilité, détection des changements et réduction des manipulations manuelles. Terraform compare cet état souhaité, son état connu et l’infrastructure réelle, puis propose des actions.

Limites : il ne remplace ni la gouvernance cloud, ni les tests applicatifs, ni les sauvegardes, ni la gestion des données. Un `apply` peut supprimer une ressource : mettre en place permissions minimales et approbation.

### HCL en bref

```hcl
# Commentaire
variable "environment" {
  description = "Environnement de déploiement"
  type        = string
  default     = "dev"
}

locals {
  name_prefix = "data-${var.environment}"
}
```

HCL comprend des blocs (`resource`, `module`, `variable`), des attributs et des expressions. Les chaînes interpolées peuvent utiliser directement une expression : `"${local.name_prefix}-raw"` reste lisible, mais `"${...}"` est souvent inutile quand la valeur entière est l’expression.

---

## 3. Les blocs essentiels

### Provider

Un provider traduit les ressources Terraform vers une API. La configuration d’authentification doit venir de l’environnement, d’un profil ou d’un rôle, pas du dépôt.

```hcl
provider "aws" {
  region = var.aws_region
  # credentials = ...  # Ne pas renseigner ici.
}
```

### Resource

Une ressource est gérée par Terraform et reçoit un identifiant d’adresse : `aws_s3_bucket.raw`.

```hcl
resource "aws_s3_bucket" "raw" {
  bucket = "mon-projet-raw-${var.environment}-${var.account_suffix}"

  tags = merge(local.common_tags, {
    Purpose = "raw-data"
  })
}
```

**AWS — facturable et adaptable.** Ce bucket est un exemple de stockage objet ; un nom doit être globalement unique selon les règles du service. En production, ajouter chiffrement, blocage d’accès public, versioning, lifecycle et journalisation selon la politique de l’organisation.

### Data source

Une data source lit une information existante sans la créer :

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

output "account_id" {
  value     = data.aws_caller_identity.current.account_id
  sensitive = true
}
```

Ne pas utiliser une data source comme mécanisme de secret : une valeur lue peut finir dans le state.

### Variables, outputs, locals

```hcl
variable "aws_region" {
  type        = string
  description = "Région AWS"
  default     = "eu-west-1"
}

variable "environment" {
  type        = string
  description = "dev, staging ou prod"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment doit être dev, staging ou prod."
  }
}

variable "tags" {
  type    = map(string)
  default = {}
}

locals {
  common_tags = merge(var.tags, {
    ManagedBy = "terraform"
    Environment = var.environment
  })
}

output "raw_bucket_name" {
  value       = aws_s3_bucket.raw.id
  description = "Nom du bucket brut"
}
```

Une variable est une entrée ; un output est une interface de module ou une information affichée ; un local factorise une expression. Ne pas confondre `sensitive = true` avec chiffrement : cela masque surtout l’affichage, le state reste à protéger.

---

## 4. Commandes et cycle de travail

1. **Écrire** et relire le changement.
2. `terraform fmt` — formatage.
3. `terraform init` — providers/modules/backend.
4. `terraform validate` — validité de configuration.
5. `terraform plan -out=tfplan` — proposition figée à revoir.
6. `terraform show tfplan` — inspection, notamment suppressions et changements de sécurité.
7. `terraform apply tfplan` — appliquer exactement le plan approuvé.
8. Tester et observer l’infrastructure.
9. `terraform destroy` uniquement pour un environnement éphémère approuvé.

Commandes utiles :

```bash
terraform providers
terraform state list
terraform state show aws_s3_bucket.raw
terraform output
terraform graph
terraform plan -refresh-only
```

`terraform plan -refresh-only` aide à constater une divergence sans proposer le changement de configuration. Les options et comportements évoluent : consulter `terraform <commande> -help`.

---

## 5. State, backend distant et verrouillage

Le **state** associe les adresses Terraform aux objets réels et mémorise des attributs. Il peut contenir des secrets ou des métadonnées sensibles. Ne pas committer `terraform.tfstate`, ses sauvegardes ou `.terraform/`.

### Backend distant

Un backend distant centralise le state, permet le travail d’équipe, les sauvegardes et souvent le verrouillage. Exemple **AWS — à adapter et potentiellement facturable** : stockage S3 et mécanisme de lock recommandé par la documentation actuelle du provider/backend. Le bucket doit être créé par une procédure bootstrap séparée ou une plateforme de provisioning, car Terraform ne peut pas utiliser comme backend une ressource qu’il doit d’abord créer dans le même cycle.

```hcl
terraform {
  backend "s3" {
    bucket       = "CHANGE-ME-tfstate"
    key          = "data-platform/dev/terraform.tfstate"
    region       = "eu-west-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

`CHANGE-ME` est volontaire : ne pas copier sans préparer le backend. Vérifier dans la documentation de votre version la méthode de verrouillage supportée ; certaines installations historiques utilisent un service dédié. Activer contrôle d’accès, chiffrement géré par clé si requis, versioning, rétention, audit et accès restreint.

Après changement de backend :

```bash
terraform init -reconfigure
# ou -migrate-state seulement après revue et sauvegarde
```

Ne jamais éditer le state à la main. Pour un state verrouillé, identifier le détenteur et la cause ; `force-unlock` est une opération exceptionnelle, à faire seulement après vérification qu’aucun apply ne tourne.

---

## 6. Modules, environnements et architecture

Un module est un dossier réutilisable avec `main.tf`, `variables.tf`, `outputs.tf` et éventuellement `versions.tf`. Son interface doit être petite, documentée et typée.

```hcl
module "raw_bucket" {
  source      = "./modules/bucket"
  name        = "mon-projet-raw-${var.environment}"
  environment = var.environment
  tags        = local.common_tags
}
```

Le module enfant déclare ses propres contraintes et renvoie ses outputs. Pour une registry distante, pin **source et version** ; vérifier le code du module et ses permissions.

### Structure recommandée

```text
infra/
  modules/{bucket,iam,pipeline}/
  envs/{dev,staging,prod}/
    main.tf variables.tf outputs.tf backend.tf
  policies/ tests/
```

Séparer les environnements par state et configuration d’exécution est généralement plus sûr que de compter uniquement sur des workspaces. Les **workspaces** permettent plusieurs states avec la même configuration, mais ne constituent pas une frontière de sécurité : éviter d’y mélanger production et développement si les droits ou les risques diffèrent.

---

## 7. Collections, conditions, fonctions et dépendances

### `for_each`, `count` et `dynamic`

```hcl
variable "project" {
  type        = string
  description = "Identifiant du projet"
}

variable "buckets" {
  type = map(object({ purpose = string }))
}

resource "aws_s3_bucket" "data" {
  for_each = var.buckets
  bucket   = "${var.project}-${each.key}-${var.environment}"
  tags     = merge(local.common_tags, { Purpose = each.value.purpose })
}
```

Préférer `for_each` quand chaque objet possède une clé stable. `count` convient à une présence oui/non ou à une liste très simple ; déplacer un index peut provoquer des remplacements.

Un `dynamic` génère un bloc répété, mais il ne faut pas l’utiliser pour rendre illisible une ressource :

```hcl
variable "allowed_cidrs" {
  type        = set(string)
  description = "CIDR autorisés, fournis par la configuration réseau"
}

resource "aws_security_group" "job" {
  name = "${local.name_prefix}-job"

  dynamic "ingress" {
    for_each = var.allowed_cidrs
    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }
}
```

Cet exemple **AWS — facturable selon usage** ne doit pas exposer un service à `0.0.0.0/0` sans justification.

### Fonctions et expressions

Fonctions fréquentes : `merge`, `try`, `coalesce`, `contains`, `lookup`, `length`, `toset`, `flatten`, `jsonencode`, `yamldecode`, `templatefile`. Utiliser `terraform console` pour les essayer :

```text
> merge({team = "data"}, {env = "dev"})
> [for x in ["raw", "curated"] : upper(x)]
```

Expressions conditionnelles :

```hcl
locals {
  instance_type = var.environment == "prod" ? "CHANGE-ME-prod-size" : "CHANGE-ME-dev-size"
}
```

Une dépendance implicite est créée par une référence (`bucket = aws_s3_bucket.raw.id`). Utiliser `depends_on` seulement quand la dépendance n’est pas exprimable par une valeur, car il élargit le graphe et ralentit les plans.

---

## 8. Import, renommage et ressources existantes

Pour adopter une ressource existante : écrire d’abord une configuration compatible, puis utiliser la commande d’import supportée par la version installée et vérifier le plan. Les blocs `import` peuvent rendre le processus déclaratif, mais leur syntaxe et les capacités d’import diffèrent selon les providers.

```hcl
import {
  to = aws_s3_bucket.raw
  id = "nom-du-bucket-existant"
}
```

Ne jamais importer en production sans sauvegarde, droits et plan relu. Pour renommer une adresse sans recréer l’objet :

```hcl
moved {
  from = aws_s3_bucket.old
  to   = aws_s3_bucket.raw
}
```

Supprimer un bloc `moved` seulement après avoir confirmé que tous les states concernés l’ont traité, selon la stratégie d’équipe.

---

## 9. Secrets, sécurité et conformité

- Utiliser IAM à privilèges minimaux, rôles temporaires, OIDC en CI, MFA et séparation des comptes.
- Injecter les secrets via un gestionnaire dédié (AWS Secrets Manager/SSM, Vault ou équivalent) ; Terraform peut encore stocker une valeur si elle passe par un attribut de ressource.
- Marquer les outputs sensibles, limiter `terraform output`, protéger le backend et chiffrer les sauvegardes.
- Activer chiffrement au repos/en transit, blocage public du stockage, logs et rétention selon la classification.
- Éviter `local-exec`/`remote-exec`, privilèges root et commandes shell non idempotentes.
- Scanner Git et CI : secrets, IaC et dépendances. Examiner chaque exception.

Exemple de variable sans valeur par défaut secrète :

```hcl
variable "alert_email" {
  type        = string
  description = "Adresse injectée par un mécanisme approuvé"
  sensitive   = true
}
```

Une valeur sensible ne doit pas être mise dans `terraform.tfvars` committé. Ajouter `*.tfvars`, `*.tfstate*` et `.terraform/` aux règles Git, tout en vérifiant que la CI reçoit ses entrées de façon contrôlée.

---

## 10. Validation, tests, linting et CI/CD

### Contrôles locaux

```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
terraform plan -out=tfplan
```

Ajouter selon les standards de l’équipe : `tflint` (règles provider), `tfsec`/`Trivy` (sécurité), `checkov` (policies), `terraform-docs` (interfaces). Les commandes et options exactes doivent être vérifiées dans les versions installées.

Les tests natifs Terraform, lorsque disponibles dans la version utilisée, vérifient des modules avec des fichiers de test ; pour une validation intégrée, utiliser Terratest ou un équivalent. Tester aussi les policies (OPA/Conftest), sans appeler des APIs coûteuses dans chaque pull request.

### Pipeline type

1. format et validation sans backend ;
2. scans sécurité, secrets et licence ;
3. plan avec identité CI en lecture et state distant verrouillé ;
4. publication du plan comme artefact protégé ;
5. approbation humaine selon l’environnement ;
6. apply du plan approuvé, avec OIDC et permissions minimales ;
7. tests smoke, observabilité et rapport.

Ne pas faire `apply` depuis un poste personnel en production. Le plan doit être réalisé dans un contexte comparable à l’apply (commit, variables, provider lockfile). Protéger les logs car un plan peut révéler des informations sensibles.

---

## 11. Coûts, drift et debugging

### Coûts

Taguer propriétaire, centre de coût, environnement et expiration. Utiliser budgets/alertes cloud, estimation pré-merge (Infracost ou outil équivalent), quotas et TTL pour les environnements de test. Rappeler qu’un bucket, une base, des logs, des adresses IP ou du trafic peuvent être facturables même si l’exemple paraît petit.

### Drift

Le drift est une différence entre configuration, state et réalité. Programmer des plans en lecture contrôlée et traiter les changements manuels : soit les réintégrer au code, soit les annuler par apply, soit adapter la politique. Après incident, faire `plan -refresh-only`, sauvegarder les preuves et éviter les corrections improvisées dans le state.

### Debugging

- lire l’adresse complète et la partie « forces replacement » ;
- exécuter `terraform show`, `terraform state show` et `terraform providers` ;
- vérifier région, compte, profil, quotas, permissions et variables ;
- utiliser temporairement `TF_LOG` avec prudence, car les logs peuvent contenir des secrets ;
- isoler un module dans un environnement jetable ;
- vérifier les notes de version et issues officielles du provider ;
- après échec partiel, relire le state et le plan avant de relancer.

---

## 12. Cas Data Engineering

### Stockage en zones raw/curated

Architecture générique : stockage objet `raw/`, `curated/`, `sandbox/`, chiffrement, lifecycle, catalogage et droits séparés. **AWS — facturable :** S3 peut être remplacé par GCS/Azure Blob ; un catalogue et des crawlers peuvent engendrer des coûts.

```hcl
variable "zones" {
  type    = set(string)
  default = ["raw", "curated"]
}

resource "aws_s3_object" "zone_marker" {
  for_each = var.zones
  bucket   = module.raw_bucket.name
  key      = "${each.value}/.keep"
  content  = "managed-by-terraform\n"
}
```

Dans un vrai système, séparer les buckets ou les prefixes selon les frontières de sécurité et utiliser des policies testées. Terraform crée la fondation ; il ne doit pas charger les données métier.

### IAM et pipeline de données

Un rôle d’exécution de pipeline doit recevoir uniquement les actions nécessaires sur les préfixes nécessaires. La ressource de pipeline peut référencer le bucket, le rôle et les outputs du réseau ; les paramètres applicatifs et secrets viennent d’un coffre. Décrire la chaîne par étapes : ingestion → contrôle qualité → transformation → catalogue → exposition.

Éviter une policy `*` « pour que cela marche ». Tester une policy avec un simulateur ou une policy-as-code, et faire réviser les droits.

---

## 13. Cas MLOps réaliste

Une plateforme MLOps sépare au minimum :

- **artefacts** : modèles, jeux de features, métriques et manifests versionnés dans un stockage objet ;
- **registry** : alias/stages et métadonnées dans un service de registry (géré ou open source) ;
- **pipeline** : entraînement reproductible, validation, approbation et promotion ;
- **serving** : endpoint batch ou temps réel, autoscaling, réseau privé si possible, logs et rollback ;
- **gouvernance** : lineage, qualité, biais, sécurité, coût et traçabilité du modèle en production.

Terraform provisionne les rôles, buckets, queues, clusters/endpoints, policies, observabilité et connexions. Il ne remplace pas le code d’entraînement ni le registry transactionnel. Ne pas mettre le binaire du modèle dans le state : publier l’artefact dans un emplacement contrôlé puis transmettre une URI et un digest immuables à la pipeline.

Exemple générique d’interface de pipeline :

```hcl
variable "model_artifact_uri" {
  type        = string
  description = "URI immuable d’un artefact validé, jamais un secret"
}

variable "model_sha256" {
  type        = string
  description = "Digest attendu de l’artefact"
  validation {
    condition     = can(regex("^[0-9a-fA-F]{64}$", var.model_sha256))
    error_message = "Fournir un SHA-256 hexadécimal."
  }
}

module "ml_pipeline" {
  source             = "./modules/ml-pipeline"
  model_artifact_uri = var.model_artifact_uri
  model_sha256       = var.model_sha256
  execution_role_arn = module.ml_role.arn
}
```

**AWS — potentiellement facturable et à adapter.** Le nom exact d’un service de pipeline/registry/serving varie ; ne pas supposer qu’un endpoint de démonstration est gratuit. La pipeline doit vérifier le digest, lancer des tests de qualité et ne promouvoir que sur approbation. Le serving doit exposer métriques de latence/erreurs, coût, dérive des données et version active.

---

## 14. Progression par niveaux

### Niveau 1 — Fondations

Lire HCL, créer un provider, variable, resource et output ; comprendre `init`, `plan`, `apply`, `destroy` ; formater et valider. **Mini-projet :** un module de bucket générique dans un compte sandbox, avec tags et output.

### Niveau 2 — Collaboration

Remote backend, lock, state, modules, inputs typés, outputs, Git et CI plan. **Mini-projet :** zones raw/curated et rôle de lecture, avec policy scan et budget.

### Niveau 3 — Production Data

Réseau, IAM avancé, `for_each`, import, `moved`, drift, observabilité, séparation des states et environnements. **Mini-projet :** pipeline ingestion → transformation, sans secret dans Terraform.

### Niveau 4 — MLOps

Artefacts immuables, registry, promotion, endpoint, rollback, lineage, tests de données/modèle, coûts et SLO. **Mini-projet :** infrastructure de pipeline ML qui déploie un serving avec digest, rôle minimal, logs et variables d’environnement séparées.

### Niveau 5 — Avancé

Design de modules et contrats, providers multiples/aliases, policy-as-code, platform engineering, disaster recovery du state, migrations sans interruption et revue de capacité. Documenter les décisions d’architecture (ADR).

---

## 15. Exercices avec corrigés succincts

### Exercice 1 — Variable sûre

Créer une variable `environment` limitée à `dev`, `staging`, `prod` et l’ajouter aux tags.

**Corrigé :** utiliser `type = string`, un bloc `validation` avec `contains`, puis `merge(var.tags, { Environment = var.environment })` dans un local.

### Exercice 2 — Collection stable

Créer un bucket logique pour `raw` et `curated` sans recréer `raw` si l’ordre de la collection change.

**Corrigé :** utiliser `for_each = toset(var.zones)` ou une map à clés stables, pas `count` sur une liste réordonnable.

### Exercice 3 — Adoption

Une file existante doit être gérée par Terraform sans être recréée.

**Corrigé :** écrire la resource, importer son identifiant selon la syntaxe officielle, lancer un plan, compléter la configuration puis ajouter `moved` uniquement pour les renommages d’adresse.

### Exercice 4 — Secret

Un développeur propose `password = "..."` dans un `.tf`.

**Corrigé :** refuser ; utiliser un coffre/secret manager ou une entrée CI éphémère, réduire les outputs et protéger le state. Vérifier si le provider stocke tout de même la valeur dans le state.

### Exercice 5 — MLOps

Déployer un modèle sans que Terraform puisse modifier silencieusement le mauvais artefact.

**Corrigé :** passer une URI versionnée et un SHA-256 validé, vérifier le digest dans le pipeline, garder le registry et l’approbation hors du state, et instrumenter le serving.

---

## 16. Aide-mémoire

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan
terraform apply tfplan
terraform state list
terraform state show ADDRESS
terraform plan -refresh-only
terraform console
```

Avant un merge :

- [ ] contraintes Terraform/providers et lockfile vérifiés ;
- [ ] plan lu : pas de suppression/remplacement inattendu ;
- [ ] state distant, verrouillage, sauvegarde et accès testés ;
- [ ] aucun secret, token ou artefact modèle dans Git/outputs ;
- [ ] permissions minimales, chiffrement, réseau et logs revus ;
- [ ] coûts, tags, budgets et expiration prévus ;
- [ ] fmt, validate, tests, lint et scans passants ;
- [ ] rollback/import/migration documentés ;
- [ ] documentation du module et outputs à jour.

---

## 17. Feuille de route vers MLOps Engineer

1. **Semaines 1–2 :** HCL, cycle Terraform, Git, cloud de base ; recréer une petite fondation en sandbox.
2. **Semaines 3–4 :** state distant, modules, IAM, CI plan/apply contrôlé ; écrire un module documenté.
3. **Semaines 5–6 :** stockage Data, pipeline, qualité, policy-as-code, coûts et drift ; présenter un diagramme et un ADR.
4. **Semaines 7–8 :** artefacts et registry, pipeline ML, digest, promotion et serving ; relier infra, code et observabilité.
5. **Ensuite :** fiabilité, sécurité, multi-comptes, disaster recovery, optimisation des coûts et mentoring.

Pour chaque étape, produire un dépôt reproductible, un README, un diagramme, un plan d’architecture, des tests et une courte analyse de coûts. La compétence visée n’est pas de connaître toutes les ressources par cœur : c’est de concevoir une interface sûre, comprendre le state, lire un plan et livrer une plateforme observable et réversible.

## Références à consulter

- Documentation officielle Terraform : langage, CLI, state, modules et tests.
- Registry officielle du provider choisi et changelog de sa version verrouillée.
- Documentation officielle du cloud pour IAM, chiffrement, stockage, quotas et tarifs.
- Documentation des outils de linting, policy-as-code, coûts et tests retenus par l’équipe.

Toujours vérifier la syntaxe, les versions, les noms de services, les tarifs et les recommandations de sécurité dans ces sources avant une mise en production.
