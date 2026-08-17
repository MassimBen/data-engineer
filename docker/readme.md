Ah, au temps pour moi ! Voici le résumé détaillé du **Module 1 du Data Engineering Zoomcamp** de DataTalksClub. 

> **Note :** Comme pour la version ML, le programme officiel de 2026 n'étant pas encore publié, ce résumé est basé sur la structure classique et fondamentale du Module 1 du DE Zoomcamp : **Introduction et Prérequis (Docker, Postgres, Python et Ingestion de données)**. Le format est prêt à être copié-collé sur GitHub.

***

# Module 1 : Introduction et Prérequis (Docker, Postgres & Python)

Ce premier module pose les fondations de l'ingénierie des données. Il couvre la configuration de l'environnement de travail, l'utilisation de Docker pour conteneuriser une base de données PostgreSQL, l'écriture de scripts Python pour ingérer des données en masse, et la compréhension des principes de base de l'architecture Data.

## 📑 Table des matières
1. [Concepts de l'Ingénierie de la Donnée (Data Engineering)](#1-concepts-de-lingénierie-de-la-donnée-data-engineering)
2. [Mise en place de l'environnement (Docker & Docker Compose)](#2-mise-en-place-de-lenvironnement-docker--docker-compose)
3. [Exécution de PostgreSQL avec Docker](#3-exécution-de-postgresql-avec-docker)
4. [Ingestion de données avec Python](#4-ingestion-de-données-avec-python)
5. [Révisions SQL (Requêtes de base)](#5-révisions-sql-requêtes-de-base)
6. [Travaux Pratiques (Homework)](#6-travaux-pratiques-homework)

---

## 1. Concepts de l'Ingénierie de la Donnée (Data Engineering)

Le module commence par une vue d'ensemble du rôle du Data Engineer.
- **Le pipeline de données :** Extraction (Extract), Transformation (Transform), Chargement (Load) - ETL/ELT.
- **Outils principaux :** Bases de données relationnelles (SQL), traitement distribué (Spark, dbt), orchestration (Airflow, Prefect), stockage cloud (GCP, AWS).
- **Objectif du cours :** Construire un pipeline de bout en bout, de l'ingestion brute jusqu'à l'analyse dans un outil de BI ou un Entrepôt de données (Data Warehouse).

## 2. Mise en place de l'environnement (Docker & Docker Compose)

Pour éviter les problèmes de "ça marche sur ma machine", le cours utilise Docker dès le premier jour.
- **Docker :** Permet d'emballer une application et ses dépendances dans un conteneur isolé.
- **Docker Compose :** Outil pour définir et gérer des applications Docker multi-conteneurs. 
- **Architecture locale :** Nous lançons deux conteneurs simultanément :
  1. `pg-database` : La base de données PostgreSQL.
  2. `pg-admin` : Une interface web graphique (GUI) pour interagir avec la base de données.
- **Réseaux (Networks) et Volumes :** Utilisation d'un volume Docker pour persister les données de Postgres même si le conteneur est détruit.

## 3. Exécution de PostgreSQL avec Docker

- **Variables d'environnement :** Configuration du mot de passe root (`POSTGRES_PASSWORD`), de l'utilisateur (`POSTGRES_USER`) et du nom de la base (`POSTGRES_DB`).
- **Mappage des ports :** Exposition du port 5432 du conteneur sur le port 5432 de la machine hôte pour permettre la connexion locale.
- **Connexion via pgAdmin :** Accès à l'interface web via `localhost:8080` et configuration de la connexion au serveur Postgres en utilisant le nom du conteneur Docker comme nom d'hôte (`pg-database` au lieu de `localhost`, grâce au réseau Docker partagé).

## 4. Ingestion de données avec Python

Il est fastidieux d'importer des fichiers CSV manuellement. Le module montre comment automatiser cela avec un script Python.
- **Dataset utilisé :** Les données des taxis jaunes de New York (NYC TLC Trip Record Data).
- **Bibliothèques clés :** `pandas` pour la manipulation de données, `sqlalchemy` pour se connecter à la base de données, et `psycopg2` comme adaptateur PostgreSQL.
- **Lecture par morceaux (Chunking) :** Les fichiers CSV massifs (souvent des millions de lignes) ne tiennent pas en mémoire RAM. Le script utilise `pd.read_csv(..., iterator=True)` et `chunksize` pour lire le fichier par blocs et les insérer progressivement dans la base.
- **Création de table :** Génération automatique du schéma SQL (DDL) à partir du DataFrame Pandas via `df.head(0).to_sql()`.

## 5. Révisions SQL (Requêtes de base)

Une fois les données ingérées, un rapide rappel de SQL est effectué pour vérifier que tout fonctionne.
- **Requêtes d'agrégation :** `COUNT`, `GROUP BY`, `ORDER BY`.
- **Exemple typique :** Compter le nombre de trajets par zone de taxi (PULocationID) pour une date donnée.
- **Fonctions de date :** Extraction du jour ou de l'heure à partir d'un timestamp pour effectuer des agrégations temporelles.

## 6. Travaux Pratiques (Homework)

Le devoir consolide les compétences acquises dans le module :
- Lancement de l'environnement Docker (Postgres + pgAdmin).
- Téléchargement d'un fichier CSV spécifique (données de taxi d'un mois précis).
- Écriture et exécution d'un script Python `ingest_data.py` pour charger ces données dans la base.
- Exécution de requêtes SQL spécifiques sur les données ingérées pour répondre aux questions du devoir.
- (Optionnel) Création d'une table de référence (zones de taxi) et réalisation de jointures (`JOIN`).

---

## 🛠️ Commandes et Code utiles

**1. Lancer l'environnement avec Docker Compose :**
```bash
# Assurez-vous d'avoir un fichier docker-compose.yml dans le dossier
docker-compose up -d
```

**2. Exemple de fichier `docker-compose.yml` :**
```yaml
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=ny_taxi
    volumes:
      - "./ny_taxi_postgres_data:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    ports:
      - "8080:80"
```

**3. Script Python d'ingestion (extrait) :**
```python
import pandas as pd
from sqlalchemy import create_engine
from time import time
import argparse

def main(params):
    user = params.user
    password = params.password
    host = params.host
    port = params.port
    db = params.db
    table_name = params.table_name
    url = params.url
    csv_name = 'output.csv'

    # Téléchargement du CSV
    os.system(f"wget {url} -O {csv_name}")

    # Connexion à Postgres
    engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')

    # Ingestion par morceaux (chunks)
    df_iter = pd.read_csv(csv_name, iterator=True, chunksize=100000)
    df = next(df_iter)

    df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
    df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)

    # Création de la table
    df.head(n=0).to_sql(name=table_name, con=engine, if_exists='replace')

    # Insertion des données
    df.to_sql(name=table_name, con=engine, if_exists='append')

    while True:
        try:
            df = next(df_iter)
            df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
            df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
            df.to_sql(name=table_name, con=engine, if_exists='append')
            print("Chunk inséré...")
        except StopIteration:
            print("Ingestion terminée !")
            break

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Ingest CSV data to Postgres')
    parser.add_argument('--user', help='Postgres user')
    parser.add_argument('--password', help='Postgres password')
    parser.add_argument('--host', help='Postgres host')
    parser.add_argument('--port', help='Postgres port')
    parser.add_argument('--db', help='Postgres database name')
    parser.add_argument('--table_name', help='Destination table name')
    parser.add_argument('--url', help='CSV file URL')
    args = parser.parse_args()
    main(args)
```

**4. Exécuter le script d'ingestion :**
```bash
python ingest_data.py \
  --user=root \
  --password=root \
  --host=localhost \
  --port=5432 \
  --db=ny_taxi \
  --table_name=yellow_taxi_trips \
  --url="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"
```

---
*Ceci est un résumé basé sur le curriculum standard de DataTalksClub Data Engineering Zoomcamp. Pour suivre le cours officiel, consultez le dépôt [DataTalksClub/data-engineering-zoomcamp](https://github.com/DataTalksClub/data-engineering-zoomcamp) sur GitHub.*
