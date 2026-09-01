# Module 1 — Conteneurisation et Infrastructure as Code (Docker, Terraform, GCP)

> **Data Engineering Zoomcamp — DataTalksClub**
> Fiche de révision du Module 1 : Docker, Docker Compose, Terraform et Google Cloud Platform

---

## 📑 Table des matières

1. [Introduction et environnement de travail](#1-introduction-et-environnement-de-travail)
2. [Introduction à Docker](#2-introduction-à-docker)
3. [Dockerfile et construction d'images](#3-dockerfile-et-construction-dimages)
4. [Pipeline de données dans un conteneur](#4-pipeline-de-données-dans-un-conteneur)
5. [Docker Compose : Postgres + pgAdmin](#5-docker-compose--postgres--pgadmin)
6. [Ingestion des données NYC Taxi dans Postgres](#6-ingestion-des-données-nyc-taxi-dans-postgres)
7. [Requêtes SQL sur les données Taxi](#7-requêtes-sql-sur-les-données-taxi)
8. [Introduction à Google Cloud Platform](#8-introduction-à-google-cloud-platform)
9. [Terraform : Infrastructure as Code](#9-terraform--infrastructure-as-code)
10. [Terraform avec GCP : Bucket et BigQuery](#10-terraform-avec-gcp--bucket-et-bigquery)
11. [Bonnes pratiques](#11-bonnes-pratiques)
12. [Aide-mémoire](#12-aide-mémoire)

---

## 1. Introduction et environnement de travail

### 1.1 Objectifs du module

- Conteneuriser un pipeline de données avec **Docker**
- Orchestrer plusieurs conteneurs avec **Docker Compose**
- Provisionner une infrastructure cloud avec **Terraform**
- Découvrir **GCP** : Cloud Storage (data lake) et BigQuery (data warehouse)

### 1.2 Architecture cible du module

```
┌─────────────────────────────────────────────────┐
│              Docker Compose (local)             │
│  ┌──────────┐   ┌──────────┐   ┌─────────────┐  │
│  │ Postgres │◄─►│ pgAdmin  │   │ ingest_data │  │
│  │  :5432   │   │  :8080   │   │  (Python)   │  │
│  └──────────┘   └──────────┘   └─────────────┘  │
└─────────────────────────────────────────────────┘
              │ Terraform
              ▼
┌─────────────────────────────────────────────────┐
│                     GCP                         │
│   ┌──────────────┐      ┌──────────────────┐    │
│   │ Cloud Storage│      │     BigQuery     │    │
│   │ (data lake)  │      │ (data warehouse) │    │
│   └──────────────┘      └──────────────────┘    │
└─────────────────────────────────────────────────┘
```

### 1.3 Prérequis

- Docker + Docker Compose
- Compte GCP avec crédits gratuits (300$)
- `gcloud` CLI, Terraform
- Python 3.x, pandas, sqlalchemy, psycopg2

---

## 2. Introduction à Docker

### 2.1 Pourquoi Docker ?

- **Reproductibilité** : le même environnement partout ("works on my machine" → résolu)
- **Isolation** : chaque service dans son conteneur
- **Portabilité** : local → cloud sans modification
- **Reproductibilité des pipelines de données** : dépendances figées

### 2.2 Concepts clés

| Concept | Description | Analogie |
|---------|-------------|----------|
| **Image** | Modèle immuable (OS + code + dépendances) | Classe / recette |
| **Conteneur** | Instance en cours d'exécution d'une image | Objet / plat cuisiné |
| **Dockerfile** | Recette pour construire une image | Code source de l'image |
| **Registry** | Dépôt d'images (Docker Hub, GAR) | Bibliothèque de recettes |
| **Volume** | Stockage persistant monté dans un conteneur | Disque externe |
| **Network** | Réseau virtuel entre conteneurs | Réseau local |

### 2.3 Commandes essentielles

```bash
# Lister les images / conteneurs
docker images
docker ps                # en cours d'exécution
docker ps -a             # tous (y compris arrêtés)

# Construire et lancer
docker build -t mon_image:tag .
docker run -it mon_image:tag
docker run -it --rm python:3.11 bash    # conteneur jetable interactif

# Cycle de vie
docker stop <id>
docker rm <id>
docker rmi <image>
docker logs <id>
docker exec -it <id> bash   # entrer dans un conteneur actif

# Nettoyage
docker system prune
```

---

## 3. Dockerfile et construction d'images

### 3.1 Structure d'un Dockerfile

```dockerfile
# Image de base
FROM python:3.11-slim

# Installer des dépendances système
RUN apt-get update && apt-get install -y wget

# Définir le répertoire de travail
WORKDIR /app

# Installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code
COPY ingest_data.py .

# Point d'entrée
ENTRYPOINT ["python", "ingest_data.py"]
```

### 3.2 Instructions principales

| Instruction | Rôle |
|-------------|------|
| `FROM` | Image de base |
| `RUN` | Exécute une commande au **build** |
| `COPY` / `ADD` | Copie des fichiers dans l'image |
| `WORKDIR` | Répertoire de travail |
| `ENV` | Variable d'environnement |
| `EXPOSE` | Documente un port |
| `CMD` | Commande par défaut (surchargeable au `run`) |
| `ENTRYPOINT` | Commande fixe (les args du `run` s'y ajoutent) |

> ⚠️ **`CMD` vs `ENTRYPOINT`** : `ENTRYPOINT` fixe le programme ; `docker run image arg1 arg2` passe `arg1 arg2` en paramètres de l'entrypoint. C'est le pattern utilisé dans le cours pour passer les arguments CLI au script d'ingestion.

### 3.3 Build et run

```bash
docker build -t taxi_ingest:v001 .

docker run -it \
  --network=pg-network \
  taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pg-database \
    --port=5432 \
    --db=ny_taxi \
    --table_name=yellow_taxi_trips \
    --url=${URL}
```

---

## 4. Pipeline de données dans un conteneur

### 4.1 Script d'ingestion (`ingest_data.py`)

Le script du cours : télécharge un CSV des données NYC Taxi → l'insère dans Postgres par chunks avec pandas + SQLAlchemy.

```python
import argparse
import pandas as pd
from sqlalchemy import create_engine
from time import time

def main(params):
    user, password = params.user, params.password
    host, port, db = params.host, params.port, params.db
    table_name, url = params.table_name, params.url

    engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')

    df_iter = pd.read_csv(url, iterator=True, chunksize=100000)

    for i, chunk in enumerate(df_iter):
        t_start = time()
        chunk.tpep_pickup_datetime = pd.to_datetime(chunk.tpep_pickup_datetime)
        chunk.tpep_dropoff_datetime = pd.to_datetime(chunk.tpep_dropoff_datetime)
        chunk.to_sql(name=table_name, con=engine, if_exists='append')
        print(f'chunk {i} inséré en {time() - t_start:.2f}s')

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Ingestion CSV vers Postgres')
    parser.add_argument('--user')
    parser.add_argument('--password')
    parser.add_argument('--host')
    parser.add_argument('--port')
    parser.add_argument('--db')
    parser.add_argument('--table_name')
    parser.add_argument('--url')
    main(parser.parse_args())
```

### 4.2 Points clés

- **`chunksize=100000`** : lecture du CSV par morceaux → mémoire maîtrisée (fichier de plusieurs Go)
- **`if_exists='append'`** : ajoute les chunks à la table (le 1er chunk peut utiliser `'replace'`)
- **Conversion des dates** : `pd.to_datetime()` pour le typage correct
- **Paramétrage CLI** avec `argparse` → réutilisable pour n'importe quel mois/fichier

### 4.3 Bonnes pratiques de conteneurisation d'un pipeline

- ✅ Un conteneur = un rôle (ingestion ≠ base de données)
- ✅ Paramètres passés par **arguments** ou **variables d'environnement**
- ✅ Traitement par **chunks** pour les gros fichiers
- ✅ Logs dans stdout (visibles via `docker logs`)

---

## 5. Docker Compose : Postgres + pgAdmin

### 5.1 Sans Compose : réseau manuel (approche pédagogique du cours)

```bash
# Créer un réseau
docker network create pg-network

# Postgres
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  --network=pg-network \
  --name pg-database \
  postgres:13

# pgAdmin
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 \
  --network=pg-network \
  --name pgadmin \
  dpage/pgadmin4
```

> 🔑 **Point clé** : dans un même réseau Docker, les conteneurs se joignent par leur **nom** (`pg-database`), pas par `localhost` !

### 5.2 Avec Docker Compose

```yaml
# docker-compose.yaml
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=ny_taxi
    volumes:
      - ny_taxi_postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - pg-network

  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    ports:
      - "8080:80"
    networks:
      - pg-network

volumes:
  ny_taxi_postgres_data:

networks:
  pg-network:
    name: pg-network
```

```bash
docker compose up -d      # lancer en arrière-plan
docker compose down       # arrêter et supprimer les conteneurs
docker compose down -v    # + supprimer les volumes (⚠️ perte des données)
docker compose logs -f    # suivre les logs
```

> 💡 Docker Compose crée automatiquement un réseau commun : les services se joignent par leur **nom de service** (`pgdatabase`).

---

## 6. Ingestion des données NYC Taxi dans Postgres

### 6.1 Via notebook (approche interactive du cours)

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')

# Structure de la table (0 lignes)
df = pd.read_csv('yellow_tripdata_2021-01.csv', nrows=100)
df.head(0).to_sql(name='yellow_taxi_data', con=engine, if_exists='replace')

# Ingestion par chunks
df_iter = pd.read_csv('yellow_tripdata_2021-01.csv', iterator=True, chunksize=100000)
for chunk in df_iter:
    chunk.tpep_pickup_datetime = pd.to_datetime(chunk.tpep_pickup_datetime)
    chunk.tpep_dropoff_datetime = pd.to_datetime(chunk.tpep_dropoff_datetime)
    chunk.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')
```

### 6.2 Données de référence : zones

```python
df_zones = pd.read_csv('taxi_zone_lookup.csv')
df_zones.to_sql(name='zones', con=engine, if_exists='replace')
```

La table `zones` (LocationID → Borough, Zone) sert aux **jointures** dans les requêtes d'analyse.

---

## 7. Requêtes SQL sur les données Taxi

Requêtes classiques du homework du module :

```sql
-- Nombre de courses le 15 janvier 2021
SELECT COUNT(*)
FROM yellow_taxi_data
WHERE CAST(tpep_pickup_datetime AS DATE) = '2021-01-15';

-- Jour avec la course la plus longue
SELECT tpep_pickup_datetime, trip_distance
FROM yellow_taxi_data
ORDER BY trip_distance DESC
LIMIT 1;

-- Jointure avec les zones (nom de la zone de pickup)
SELECT z."Zone", COUNT(*) as trips
FROM yellow_taxi_data t
JOIN zones z ON t."PULocationID" = z."LocationID"
GROUP BY z."Zone"
ORDER BY trips DESC
LIMIT 10;

-- Pourboire le plus élevé par zone de dépôt
SELECT zdo."Zone", MAX(t.tip_amount) as max_tip
FROM yellow_taxi_data t
JOIN zones zpu ON t."PULocationID" = zpu."LocationID"
JOIN zones zdo ON t."DOLocationID" = zdo."LocationID"
WHERE zpu."Zone" = 'East Harlem North'
GROUP BY zdo."Zone"
ORDER BY max_tip DESC;
```

> 🔑 Vérifier avec `\d yellow_taxi_data` (dans `psql`) ou `pg_get_columns()` que les types de colonnes sont corrects (timestamps, pas texte !).

### Connexion en ligne de commande

```bash
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

---

## 8. Introduction à Google Cloud Platform

### 8.1 Concepts

| Service | Rôle dans le cours |
|---------|-------------------|
| **Cloud Storage (GCS)** | Data lake — stockage objet des fichiers bruts |
| **BigQuery** | Data warehouse — requêtes analytiques (cf. Module 3) |
| **IAM** | Gestion des identités et permissions |
| **Service Account** | Identité machine pour les applications (pas un humain) |

### 8.2 Service Account

```bash
# Créer un service account
gcloud iam service-accounts create de-zoomcamp-sa \
  --display-name "DE Zoomcamp Service Account"

# Attribution des rôles (dans la console ou en CLI)
# - Storage Admin        (gérer GCS)
# - Storage Object Admin (lire/écrire les objets)
# - BigQuery Admin       (gérer BigQuery)

# Créer et télécharger la clé JSON
gcloud iam service-accounts keys create ~/keys/gcp-key.json \
  --iam-account de-zoomcamp-sa@PROJECT_ID.iam.gserviceaccount.com

# Authentification
export GOOGLE_APPLICATION_CREDENTIALS=~/keys/gcp-key.json
gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
```

> ⚠️ **Ne jamais commiter la clé JSON** dans Git ! L'ajouter au `.gitignore`.

---

## 9. Terraform : Infrastructure as Code

### 9.1 Pourquoi Terraform ?

- **Infrastructure as Code (IaC)** : infrastructure décrite dans des fichiers versionnés
- **Reproductibilité** : même infra déployée en dev/staging/prod
- **Plan avant application** : prévisualisation des changements
- **Multi-cloud** : GCP, AWS, Azure... avec la même syntaxe (HCL)
- **Destroy facile** : démontage propre de l'infra (évite les factures surprises !)

### 9.2 Workflow Terraform

```
terraform init  →  terraform plan  →  terraform apply  →  terraform destroy
   (plugins)      (prévisualiser)     (créer/modifier)     (tout supprimer)
```

| Commande | Rôle |
|----------|------|
| `terraform init` | Initialise le projet, télécharge les providers |
| `terraform fmt` | Formate les fichiers `.tf` |
| `terraform validate` | Vérifie la syntaxe |
| `terraform plan` | Affiche les changements prévus (rien n'est appliqué) |
| `terraform apply` | Applique les changements (demande confirmation) |
| `terraform destroy` | Supprime toutes les ressources gérées |
| `terraform state list` | Liste les ressources dans le state |
| `terraform show` | Affiche le state courant |

### 9.3 Le fichier de state

- `terraform.tfstate` : **état réel** de l'infrastructure gérée par Terraform
- Permet à Terraform de savoir ce qui existe déjà (diff plan ↔ réalité)
- En équipe : **remote state** (bucket GCS/S3) — jamais en local partagé par Git

---

## 10. Terraform avec GCP : Bucket et BigQuery

### 10.1 Structure du projet

```
terraform/
├── main.tf       # provider + ressources
└── variables.tf  # variables paramétrables
```

### 10.2 `variables.tf`

```hcl
variable "credentials" {
  description = "Chemin vers la clé du service account"
  default     = "./keys/my-creds.json"
}

variable "project" {
  description = "ID du projet GCP"
  default     = "de-zoomcamp-123456"
}

variable "region" {
  description = "Région GCP"
  default     = "europe-west1"
}

variable "location" {
  description = "Location multi-régionale"
  default     = "EU"
}

variable "bq_dataset_name" {
  description = "Nom du dataset BigQuery"
  default     = "demo_dataset"
}

variable "gcs_bucket_name" {
  description = "Nom du bucket GCS (globalement unique !)"
  default     = "de-zoomcamp-123456-terra-bucket"
}

variable "gcs_storage_class" {
  description = "Classe de stockage"
  default     = "STANDARD"
}
```

### 10.3 `main.tf`

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  credentials = file(var.credentials)
  project     = var.project
  region      = var.region
}

# Data Lake : bucket Cloud Storage
resource "google_storage_bucket" "demo-bucket" {
  name          = var.gcs_bucket_name
  location      = var.location
  storage_class = var.gcs_storage_class
  force_destroy = true          # permet de supprimer même non vide

  lifecycle_rule {
    condition {
      age = 1                   # jours
    }
    action {
      type = "AbortIncompleteMultipartUpload"
    }
  }
}

# Data Warehouse : dataset BigQuery
resource "google_bigquery_dataset" "demo_dataset" {
  dataset_id = var.bq_dataset_name
  location   = var.location
  delete_contents_on_destroy = true
}
```

### 10.4 Déploiement

```bash
cd terraform/
export GOOGLE_APPLICATION_CREDENTIALS=./keys/my-creds.json

terraform init
terraform plan
terraform apply        # taper "yes"
# ... vérifier dans la console GCP ...
terraform destroy      # ⚠️ toujours nettoyer pour éviter les coûts !
```

---

## 11. Bonnes pratiques

### Docker
- ✅ Images **slim** (`python:3.11-slim`) pour réduire la taille
- ✅ `.dockerignore` (comme `.gitignore`) pour exclure données et secrets
- ✅ Volumes nommés pour la **persistance** des données
- ✅ `--rm` pour les conteneurs jetables de test

### GCP / Sécurité
- ✅ **Principe du moindre privilège** : rôles IAM minimaux
- ✅ Clés de service account **hors Git** (`.gitignore`)
- ✅ Toujours **destroy** les ressources de test

### Terraform
- ✅ Variables pour tout ce qui change (projet, région, noms)
- ✅ `plan` **avant** chaque `apply`
- ✅ State distant en équipe

---

## 12. Aide-mémoire

```bash
# === DOCKER ===
docker build -t img:tag .
docker run -it --rm --network=pg-network img:tag [args]
docker network create pg-network
docker compose up -d / down / logs -f

# === POSTGRES ===
pgcli -h localhost -p 5432 -u root -d ny_taxi

# === GCP ===
gcloud auth activate-service-account --key-file=key.json
export GOOGLE_APPLICATION_CREDENTIALS=~/keys/key.json

# === TERRAFORM ===
terraform init → fmt → validate → plan → apply → destroy
```

---

## ❓ Questions de révision

1. **Différence entre image et conteneur ?**
   → Image = modèle immuable ; conteneur = instance en cours d'exécution de cette image.

2. **Pourquoi `--network=pg-network` et pas `localhost` pour joindre Postgres ?**
   → Dans un réseau Docker, les conteneurs communiquent par leur **nom** ; `localhost` d'un conteneur désigne ce conteneur lui-même.

3. **`CMD` vs `ENTRYPOINT` ?**
   → `ENTRYPOINT` fixe le programme exécuté ; `CMD` fournit des arguments par défaut surchargeables. `docker run img args` remplace le `CMD` mais ajoute les args à l'`ENTRYPOINT`.

4. **Pourquoi lire le CSV par chunks ?**
   → Un fichier de plusieurs Go ne tient pas en mémoire ; le traitement par morceaux de 100 000 lignes garde une empreinte RAM constante.

5. **À quoi sert `terraform plan` ?**
   → À prévisualiser les ressources qui seront créées/modifiées/supprimées, sans rien appliquer — sécurité avant le `apply`.

6. **Pourquoi `force_destroy = true` sur le bucket du cours ?**
   → Pour permettre à `terraform destroy` de supprimer le bucket même s'il contient des objets (pratique en environnement d'apprentissage, ⚠️ dangereux en prod).

---

> 📚 **Ressources** :
> - [Repo GitHub du Zoomcamp — Module 1](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/01-docker-terraform)
> - [Documentation Docker](https://docs.docker.com/)
> - [Documentation Terraform](https://developer.hashicorp.com/terraform/docs)
> - [Documentation GCP](https://cloud.google.com/docs)
