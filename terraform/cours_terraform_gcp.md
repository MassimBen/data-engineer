# Terraform sur Google Cloud — du débutant à MLOps Engineer

## 1. Objectif et chemin d’apprentissage

Terraform décrit l’infrastructure dans des fichiers texte reproductibles. Pour un Data Engineer junior, l’intérêt est de versionner les projets, les droits, le data lake, les entrepôts et les services qui alimentent les traitements. Pour évoluer vers MLOps, on ajoute les registres d’images, le déploiement d’API et les connexions vers Vertex AI.

Le cours suit cette progression :

1. installer Terraform et préparer un projet Google Cloud ;
2. comprendre HCL, providers, variables, sorties et état ;
3. construire un socle data (Cloud Storage, BigQuery, Pub/Sub) ;
4. gérer IAM, l’état distant, les modules et les environnements ;
5. déployer une image sur Cloud Run et relier les notions MLOps ;
6. automatiser avec GitHub Actions et Workload Identity Federation.

> **Attention aux coûts.** Certaines ressources sont facturées : stockage et opérations Cloud Storage, requêtes et stockage BigQuery, messages Pub/Sub, Artifact Registry, Cloud Run au-delà des quotas gratuits, Vertex AI et journaux. Utilisez un projet de test, des quotas, des budgets et détruisez les ressources d’exercice. Les identifiants entre `<...>` sont à remplacer.

## 2. Prérequis et installation

### Prérequis

- Bases de Linux, Git, YAML et SQL ;
- un compte Google Cloud avec un projet de test `<PROJECT_ID>` et la facturation activée si nécessaire ;
- permission de créer les ressources d’exercice, ou un administrateur pour les API et IAM ;
- un dépôt Git ;
- Terraform, Google Cloud CLI (`gcloud`) et, pour les exemples conteneurisés, Docker.

### Installation et première configuration

Installez Terraform depuis la distribution officielle adaptée à votre système, puis vérifiez :

```bash
terraform version
gcloud version
gcloud auth login
gcloud config set project <PROJECT_ID>
gcloud auth application-default login
```

`gcloud auth login` authentifie la CLI. `gcloud auth application-default login` crée des **Application Default Credentials (ADC)** locales utilisées par le provider Google. En CI, ne copiez pas ce fichier local : utilisez une identité fédérée (voir § 15). Pour un compte de service exécuté sur Google Cloud, utilisez l’identité attachée à la ressource plutôt qu’une clé exportée.

Créez un dossier de travail :

```bash
mkdir infra-gcp && cd infra-gcp
git init
```

## 3. Premier projet Terraform

### HCL, provider et ressources

HCL est le langage déclaratif de Terraform. Le provider traduit les ressources en appels à l’API Google Cloud. Un fichier `main.tf` minimal :

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_storage_bucket" "raw" {
  name                        = "<UNIQUE_BUCKET_NAME>"
  location                    = var.region
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  force_destroy               = false
}
```

Le nom d’un bucket est globalement unique. `force_destroy = false` évite de supprimer accidentellement des objets. Vérifiez la version du provider dans la documentation correspondant à votre version avant de produire en équipe.

### Variables et outputs

`variables.tf` :

```hcl
variable "project_id" {
  description = "Identifiant du projet Google Cloud"
  type        = string
  sensitive   = false
}

variable "region" {
  description = "Région des ressources régionales"
  type        = string
  default     = "europe-west1"
}

variable "labels" {
  description = "Étiquettes de coût et de propriété"
  type        = map(string)
  default     = { owner = "data-platform" }
}
```

`outputs.tf` :

```hcl
output "raw_bucket_name" {
  description = "Nom du bucket d’arrivée"
  value       = google_storage_bucket.raw.name
}
```

Passez les valeurs par `terraform.tfvars` (non secret) ou par variables d’environnement :

```hcl
project_id = "<PROJECT_ID>"
region     = "europe-west1"
```

Ajoutez `*.tfvars` contenant des secrets et `.terraform/` au `.gitignore`. Une variable `sensitive = true` masque l’affichage, mais sa valeur peut rester dans l’état : ne mettez donc jamais de secret dans une configuration Terraform si Secret Manager ou une identité suffit.

## 4. Cycle de travail et état

Commandes principales :

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
terraform show tfplan
terraform apply tfplan
terraform output
terraform state list
terraform destroy
```

- `init` installe les providers et initialise le backend ;
- `plan` prévisualise les changements ;
- `apply` les exécute après revue ;
- `state` est la correspondance entre la configuration et les objets réels.

Ne modifiez pas manuellement le fichier d’état et ne le versionnez pas. Utilisez un verrouillage et des droits minimaux. `import` permet d’adopter une ressource existante, mais il faut ensuite écrire une configuration qui décrit réellement ses attributs :

```bash
terraform import google_storage_bucket.raw <BUCKET_NAME>
```

## 5. Activer les APIs Google Cloud

Les APIs sont activées au niveau du projet. Cette ressource pratique demande les droits suffisants et peut créer une dépendance initiale :

```hcl
variable "services" {
  type = set(string)
  default = [
    "storage.googleapis.com",
    "bigquery.googleapis.com",
    "pubsub.googleapis.com",
    "iam.googleapis.com",
    "run.googleapis.com",
    "artifactregistry.googleapis.com",
    "aiplatform.googleapis.com",
  ]
}

resource "google_project_service" "enabled" {
  for_each           = var.services
  project            = var.project_id
  service            = each.value
  disable_on_destroy = false
}
```

Ne désactivez pas automatiquement une API partagée par d’autres équipes. En production, convenez de la propriété de cette liste et appliquez les changements pendant une fenêtre contrôlée.

## 6. Socle data : Cloud Storage, BigQuery et Pub/Sub

### Data lake Cloud Storage

```hcl
resource "google_storage_bucket" "lake" {
  name                        = "<UNIQUE_LAKE_BUCKET>"
  project                     = var.project_id
  location                    = "EU"
  storage_class               = "STANDARD"
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  versioning { enabled = true }
  lifecycle_rule {
    condition { age = 30 }
    action    { type = "Delete" }
  }
  labels = var.labels
}
```

La règle supprime les objets après 30 jours : adaptez-la à la rétention légale et aux besoins de restauration. Un bucket régional, dual-régional ou multi-régional doit respecter la résidence et le coût attendus. Pour séparer les zones, préférez des préfixes (`raw/`, `curated/`, `features/`) ou plusieurs buckets selon les exigences de sécurité.

### BigQuery : dataset et table simple

```hcl
resource "google_bigquery_dataset" "analytics" {
  dataset_id                 = "analytics"
  project                    = var.project_id
  location                   = "EU"
  delete_contents_on_destroy = false
  labels                     = var.labels
}

resource "google_bigquery_table" "events" {
  dataset_id = google_bigquery_dataset.analytics.dataset_id
  project    = var.project_id
  table_id   = "events"
  schema = jsonencode([
    { name = "event_id", type = "STRING", mode = "REQUIRED" },
    { name = "event_ts", type = "TIMESTAMP", mode = "REQUIRED" },
    { name = "payload",  type = "JSON",      mode = "NULLABLE" },
  ])
}
```

Les requêtes et le stockage BigQuery peuvent coûter. Utilisez des partitions et du clustering adaptés aux volumes, limitez les scans, et ne laissez pas `delete_contents_on_destroy = true` sur un dataset de production.

### Pub/Sub : topic et abonnement

```hcl
resource "google_pubsub_topic" "events" {
  name    = "events"
  project = var.project_id
}

resource "google_pubsub_subscription" "events_worker" {
  name                       = "events-worker"
  topic                      = google_pubsub_topic.events.id
  project                    = var.project_id
  ack_deadline_seconds       = 30
  message_retention_duration = "604800s"
}
```

Ajoutez une dead-letter topic, des filtres et une stratégie de retry pour un flux réel. Les messages conservés et délivrés peuvent engendrer des coûts ; dimensionnez la rétention.

## 7. IAM et comptes de service

Un compte de service représente une application, pas une personne. Accordez le rôle le plus étroit possible au niveau le plus bas possible :

```hcl
resource "google_service_account" "pipeline" {
  account_id   = "de-pipeline"
  display_name = "Pipeline data"
  project      = var.project_id
}

resource "google_project_iam_member" "pipeline_bigquery" {
  project = var.project_id
  role    = "roles/bigquery.jobUser"
  member  = "serviceAccount:${google_service_account.pipeline.email}"
}

resource "google_storage_bucket_iam_member" "pipeline_lake" {
  bucket = google_storage_bucket.lake.name
  role   = "roles/storage.objectUser"
  member = "serviceAccount:${google_service_account.pipeline.email}"
}
```

Évitez les rôles larges au niveau projet, les comptes partagés et les clés JSON persistantes. Auditez les bindings, séparez les comptes de planification et d’exécution, et utilisez des contraintes d’organisation. Les permissions de déploiement doivent être distinctes des permissions de lecture des données.

## 8. Backend d’état distant dans GCS

Créez d’abord, dans un bootstrap contrôlé, un bucket dédié à l’état. Il ne doit pas être géré par le même état qu’il héberge :

```hcl
resource "google_storage_bucket" "tfstate" {
  name                        = "<UNIQUE_TFSTATE_BUCKET>"
  project                     = var.project_id
  location                    = "EU"
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  versioning { enabled = true }
}
```

Puis, dans les configurations qui utilisent ce bucket :

```hcl
terraform {
  backend "gcs" {
    bucket = "<UNIQUE_TFSTATE_BUCKET>"
    prefix = "terraform/data-platform"
  }
}
```

Le backend GCS fournit le stockage et le verrouillage nécessaires au travail partagé. Le nom du bucket n’est pas interpolable dans un bloc backend : fournissez-le au `terraform init -backend-config=...` ou remplacez le champ. Limitez l’accès au bucket et activez sa versionisation.

## 9. Modules et environnements dev/prod

Un module regroupe une responsabilité réutilisable. Exemple d’appel :

```hcl
module "lake" {
  source     = "./modules/lake"
  project_id = var.project_id
  name       = "<UNIQUE_BUCKET_NAME>"
  location   = var.region
  labels     = var.labels
}
```

Un module doit exposer des variables documentées, des outputs utiles et peu de valeurs implicites. Épinglez les versions de modules et de providers.

Deux approches valables pour `dev` et `prod` sont des dossiers séparés avec un backend et des variables propres, ou des workspaces pour des variantes homogènes. Pour des données critiques, préférez des états séparés et des projets séparés :

```text
infra/
  modules/lake/{main.tf,variables.tf,outputs.tf}
  environments/dev/{backend.tf,main.tf,terraform.tfvars}
  environments/prod/{backend.tf,main.tf,terraform.tfvars}
```

Ne faites pas dépendre `prod` d’un état local de `dev`. Utilisez des projets, comptes de service, quotas et politiques distincts. Exigez une approbation humaine avant l’application en production.

## 10. Artifact Registry et Cloud Run

### Registre d’images

```hcl
resource "google_artifact_registry_repository" "containers" {
  location      = var.region
  repository_id = "containers"
  description   = "Images des services data et ML"
  format        = "DOCKER"
  project       = var.project_id
}
```

Les images et leur conservation consomment du stockage. Nettoyez les versions inutiles selon une politique de rétention, sans supprimer une version encore déployée.

### Service Cloud Run

```hcl
resource "google_cloud_run_v2_service" "predict" {
  name     = "predict-api"
  location = var.region
  project  = var.project_id
  ingress  = "INGRESS_TRAFFIC_INTERNAL_ONLY"

  template {
    service_account = google_service_account.pipeline.email
    containers {
      image = "<REGION>-docker.pkg.dev/<PROJECT_ID>/containers/predict:<IMMUTABLE_TAG>"
      resources { limits = { cpu = "1", memory = "512Mi" } }
      ports { container_port = 8080 }
    }
  }
}
```

L’accès interne, les limites, la concurrence, le minimum d’instances et le timeout doivent refléter la latence et le budget. Ne rendez pas le service public sans authentification et sans revue. Utilisez un digest d’image plutôt qu’un tag mutable lorsque la reproductibilité est importante.

## 11. Vertex AI : ce que Terraform doit gérer

Terraform peut provisionner les fondations d’un cycle ML : APIs, buckets, comptes de service, datasets, registres, réseaux, jobs ou endpoints selon les ressources du provider disponibles. Le code d’entraînement, les tests de modèle, les métriques et les données restent généralement dans le dépôt et les pipelines appropriés.

Principes utiles :

- stocker les données et artefacts dans des emplacements conformes à la résidence ;
- donner à l’exécution Vertex AI un compte de service dédié, avec accès minimal à Cloud Storage et BigQuery ;
- versionner le code, les images et les modèles ;
- enregistrer paramètres, métriques, validation et approbation avant promotion ;
- protéger les endpoints, surveiller les erreurs, la dérive et les coûts de prédiction ;
- vérifier dans la documentation du provider la ressource exacte avant de l’utiliser : ne pas supposer qu’un composant géré est disponible simplement parce que le service existe.

## 12. Secrets et sécurité sans clés JSON

Ne mettez jamais de mot de passe, token ou clé privée dans Git, `tfvars`, une image ou l’état. Utilisez Secret Manager et injectez le secret au runtime :

```hcl
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password"
  project   = var.project_id
  replication { auto {} }
}

resource "google_secret_manager_secret_iam_member" "runtime" {
  project   = var.project_id
  secret_id = google_secret_manager_secret.db_password.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.pipeline.email}"
}
```

La valeur du secret doit être créée par un processus contrôlé, pas écrite dans le code Terraform. Faites tourner les secrets, restreignez les accès et surveillez les lectures. En local, ADC suffit ; sur Google Cloud, l’identité attachée suffit ; en CI, utilisez Workload Identity Federation.

## 13. CI/CD GitHub Actions avec Workload Identity Federation

Le principe est une fédération OIDC courte durée : GitHub obtient un jeton, Google Cloud vérifie les attributs autorisés, puis le workflow agit comme un compte de service. Il n’y a pas de clé JSON longue durée à stocker.

Le pool et le provider de fédération sont une fondation d’organisation. Leur configuration exacte doit être revue avec les contraintes de votre organisation ; liez le provider à un dépôt et une branche ou un environnement précis, jamais à tous les dépôts par défaut. Exemple de workflow :

```yaml
name: terraform
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL_ID>/providers/<PROVIDER_ID>"
          service_account: "terraform-ci@<PROJECT_ID>.iam.gserviceaccount.com"
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.8.5
      - run: terraform -chdir=environments/dev init -input=false
      - run: terraform -chdir=environments/dev fmt -check -recursive
      - run: terraform -chdir=environments/dev validate
      - run: terraform -chdir=environments/dev plan -input=false -out=tfplan
```

Pour `apply`, déclenchez un job séparé après approbation d’environnement et réutilisez un plan produit et contrôlé. Épinglez les actions à des versions ou des SHA selon la politique de l’équipe. Donnez au compte CI seulement les droits requis par l’état et les ressources ; un compte CI de production séparé est recommandé.

## 14. Tests, validation et qualité

Avant une revue :

```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
terraform plan -var-file=dev.tfvars
```

Complétez avec :

- analyse statique (par exemple TFLint ou un scanner de configuration approuvé) ;
- tests de règles IAM, labels, régions, chiffrement et absence de ressources publiques ;
- tests d’intégration dans un projet éphémère, puis destruction ;
- `terraform test` pour les modules lorsque la version de Terraform et les tests utilisés le permettent ;
- revue du plan : créations, destructions, changements de permissions et estimations de coût.

Ne considérez pas `validate` comme un test d’existence des APIs ou de bon fonctionnement applicatif : il vérifie principalement la cohérence de la configuration.

## 15. Bonnes pratiques

- Commencer par un petit périmètre et lire le plan ;
- épingler Terraform et les providers ;
- formater, valider et scanner dans la CI ;
- utiliser des labels (`owner`, `environment`, `cost_center`) ;
- séparer projets et états pour les environnements sensibles ;
- documenter propriétaires, dépendances, régions, rétention et procédure de reprise ;
- protéger le state et limiter ses lecteurs ;
- choisir des noms stables et éviter les changements forcés ;
- utiliser `prevent_destroy` avec discernement sur des données critiques ;
- mettre en place budgets, alertes, quotas et logs ;
- effectuer une revue de sécurité avant toute exposition réseau ou attribution IAM.

## 16. Dépannage

- **ADC introuvables** : vérifier `gcloud auth application-default login`, le projet actif et l’identité réellement utilisée ; en CI, vérifier le provider fédéré et `id-token: write`.
- **Permission denied** : identifier l’appel et le principal, vérifier le rôle au bon niveau, l’API et les contraintes d’organisation. Attendre parfois la propagation IAM.
- **API non activée** : contrôler `google_project_service.enabled` et les autorisations d’activation.
- **Bucket déjà existant** : un nom est global ; choisir un nom unique ou importer l’objet existant, puis aligner la configuration.
- **Diff perpétuel** : comparer la configuration et les valeurs par défaut du provider, les changements opérés hors Terraform et les règles de normalisation.
- **State verrouillé** : vérifier qu’aucune exécution n’est active ; ne forcer le déverrouillage qu’après identification du verrou et de l’exécution interrompue.
- **Échec Cloud Run** : vérifier l’image, le port 8080, les permissions de lecture Artifact Registry, les logs et les paramètres CPU/mémoire.
- **Coût inattendu** : inspecter stockage, scans BigQuery, rétention Pub/Sub, instances Cloud Run, images et jobs Vertex AI ; arrêter et détruire les ressources de test.

## 17. Exercices progressifs

1. Installer les outils, créer un provider, une variable `project_id` et un output.
2. Activer Storage et BigQuery, créer un bucket privé versionné avec labels.
3. Créer un dataset et une table partitionnée ; expliquer comment limiter les scans.
4. Ajouter un topic et un abonnement Pub/Sub avec une dead-letter topic.
5. Créer un compte de service de pipeline et limiter ses droits à un bucket et un dataset.
6. Migrer le state vers GCS, tester deux exécutions concurrentes et documenter le verrouillage.
7. Transformer le bucket en module et déployer `dev` puis `prod` dans deux états.
8. Déployer Cloud Run avec une image immuable et un compte de service sans accès public.
9. Ajouter fmt, validate, plan et une approbation de déploiement dans GitHub Actions.

## 18. Deux mini-projets

### Mini-projet A — ingestion événementielle

Construire une base de plateforme : bucket `raw`, dataset BigQuery, topic et abonnement Pub/Sub, compte de service d’ingestion et state GCS. Livrables : module réutilisable, environnements `dev`/`prod`, labels, README, plan revu, test d’écriture contrôlé et destruction de l’environnement de test. Critères : aucune ressource publique, IAM minimal, rétention explicitement justifiée et budget documenté.

### Mini-projet B — API de prédiction

Créer un dépôt qui contient un module Artifact Registry, un compte de service d’exécution, un service Cloud Run déployé avec une image immuable et les fondations Vertex AI nécessaires à l’enregistrement des artefacts. Ajouter une CI avec fédération OIDC, validation Terraform, plan sur demande et promotion approuvée. Livrables : diagramme simple, procédure de rollback, métriques à surveiller et estimation des coûts. Ne stocker aucune clé JSON et ne placer aucune donnée sensible dans l’image ou l’état.

## 19. Référence rapide

```bash
# Préparer
 gcloud config set project <PROJECT_ID>
 gcloud auth application-default login

# Développer
 terraform fmt -recursive
 terraform init
 terraform validate
 terraform plan -out=tfplan
 terraform apply tfplan

# Inspecter
 terraform show
 terraform output
 terraform state list

# Nettoyer un environnement de test
 terraform destroy
```

Avant chaque `apply`, demander : **quel projet, quel compte, quelles ressources vont changer, quelles données peuvent être supprimées, et quel est le coût attendu ?**
