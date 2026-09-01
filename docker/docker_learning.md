# Docker pour la data et le MLOps

> Cours progressif en français — du premier conteneur à la mise en production d’une stack ML.

Ce dépôt accompagne un **data engineer junior** vers les responsabilités d’un **MLOps engineer confirmé**. Chaque module alterne concepts, commandes reproductibles et travaux pratiques. Les labs sont autonomes et peuvent être exécutés depuis la racine du dépôt.

## Objectifs pédagogiques

À la fin du parcours, vous saurez :

- expliquer les différences entre image, conteneur, registre, volume, réseau et orchestration ;
- écrire des Dockerfiles reproductibles, rapides à construire et sûrs ;
- composer une plateforme locale PostgreSQL, pgAdmin, ingestion, MLflow et MinIO ;
- conteneuriser un ETL, une API FastAPI et un entraînement GPU ;
- diagnostiquer logs, réseau, stockage, performances et vulnérabilités ;
- automatiser build, tests, scan et publication avec GitHub Actions ;
- comprendre comment passer de Docker Compose à Kubernetes.

## Prérequis

Terminal, Git, bases Python et SQL, notions de HTTP/REST. Aucun Docker préalable. Un ordinateur avec 8 Go de RAM (16 Go recommandés pour les stacks ML), 20 Go libres et une connexion Internet sont recommandés. Les exemples ont été testés conceptuellement avec Docker Engine 26+ / Docker Compose v2 ; vérifiez la version installée avec `docker version`.

## Plan du cours

| Module | Thème | Livrable pratique |
|---|---|---|
| [00](modules/00-introduction.md) | Pourquoi Docker en data/MLOps | diagnostic « ça marche chez moi » |
| [01](modules/01-installation-premiers-pas.md) | Installation et commandes | premier conteneur et cycle de vie |
| [02](modules/02-images-registres.md) | Images et Docker Hub | inspecter, taguer, publier |
| [03](modules/03-dockerfile.md) | Construire une image | image Python d’un script |
| [04](modules/04-cycle-vie-stockage.md) | Cycle de vie et persistance | volume PostgreSQL + sauvegarde |
| [05](modules/05-reseaux.md) | Réseaux Docker | service Python → PostgreSQL |
| [06](modules/06-compose.md) | Compose multi-services | PostgreSQL + pgAdmin + ingestion |
| [07](modules/07-bonnes-pratiques-data.md) | Images data optimisées | multi-stage, cache, uv/venv |
| [08](modules/08-data-engineering.md) | ETL, Airflow, MinIO, scheduling | pipeline de données local |
| [09](modules/09-mlops.md) | Serving, MLflow, Jupyter | API de modèle et tracking |
| [10](modules/10-gpu.md) | CUDA et GPU | entraînement PyTorch conteneurisé |
| [11](modules/11-securite-production.md) | Sécurité et production | scan, secrets, limites |
| [12](modules/12-ci-cd-orchestration.md) | CI/CD et Kubernetes | workflow GitHub Actions + manifeste |

## Utiliser le cours

1. Clonez le dépôt puis ouvrez chaque module dans l’ordre.
2. Copiez le lab associé, ou exécutez-le depuis la racine : `docker compose -f labs/compose-data/docker-compose.yml up --build`.
3. Gardez un terminal dédié aux logs (`docker compose logs -f`) et détruisez les ressources de lab (`down -v`) quand vous avez fini.
4. Faites l’exercice **avant** de lire la correction. Les corrections donnent une solution possible, pas la seule solution correcte.

```bash
git clone <url-du-depot> && cd cours-docker-data-mlops
docker version
docker compose version
```

## Règles de sécurité pour les labs

Les mots de passe et clés dans les fichiers d’exemple sont **locaux et factices**. En production, utilisez un gestionnaire de secrets, des variables injectées par la plateforme et des images épinglées par digest. N’exposez jamais PostgreSQL, Docker socket ou une clé cloud sur Internet.

## Ressources complémentaires

- [Documentation Docker](https://docs.docker.com/) et [Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Docker image best practices](https://docs.docker.com/build/building/best-practices/)
- [OWASP Container Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [MLflow Documentation](https://mlflow.org/docs/latest/)

## Licence et conventions

Cours fourni pour l’apprentissage. Les commandes sont prévues pour un environnement de développement. Les fichiers utilisent LF, UTF-8 et des exemples de versions explicites lorsque cela apporte de la reproductibilité.

# Module 00 — Pourquoi Docker pour la data et le MLOps

## Objectifs

- Identifier les causes du « ça marche sur ma machine ».
- Distinguer processus, conteneur, image et machine virtuelle.
- Relier reproductibilité, portabilité et cycle de vie d’un modèle.

## Théorie

Un pipeline data assemble Python, bibliothèques natives, pilotes, bases, fichiers de configuration et variables d’environnement. Deux développeurs peuvent avoir le même code mais des versions de `numpy` ou de libc différentes. Docker décrit l’environnement sous forme de fichiers versionnés et exécute le processus dans un espace isolé.

Une **image** est un paquet immuable en couches (système minimal + dépendances + code). Un **conteneur** est une instance lancée depuis cette image, avec un processus principal, un système de fichiers en écriture éphémère et des limites de ressources possibles. Un registre stocke les images.

```text
Image (Dockerfile -> build)       Registre
        | pull/push                   |
        v                             v
Conteneur A : API              Conteneur B : worker
        \          réseau Docker       /
         +---- PostgreSQL / volume ---+
```

Un conteneur partage le noyau de l’hôte ; une VM embarque un OS invité. Le conteneur démarre plus vite et consomme moins, mais n’est pas une frontière de sécurité absolue. Docker ne remplace ni les tests, ni le versionnage de données, ni Kubernetes.

### Pourquoi c’est particulièrement utile en MLOps

Le modèle doit être servi avec le même tokenizer, les mêmes poids et la même version de runtime que pendant la validation. Une image peut embarquer le code de serving et référencer une version de modèle immuable dans un registre d’artefacts. On sépare ainsi **build** (reproductible) et **run** (configuration injectée).

## Exemple : rendre un diagnostic explicite

```bash
# Capturer les versions plutôt que dire seulement « Python récent »
python --version
pip freeze > requirements-lock.txt
docker version
```

Le fichier produit n’est pas encore une image reproductible : il faut aussi fixer l’OS de base, les bibliothèques système, le code et la configuration. Nous le ferons dès le module 03.

## Exercice

Listez cinq dépendances d’un pipeline de scoring (Python, modèle, base, secret, ressource CPU) et classez-les en : **à construire dans l’image**, **à monter/injecter au démarrage**, **à fournir par un service externe**.

### Correction possible

Code et dépendances Python vont dans l’image ; le modèle volumineux va dans un registry d’artefacts ou un volume contrôlé ; secrets dans un secret manager ; URL et seuils via variables d’environnement ; PostgreSQL comme service séparé avec sauvegarde et monitoring.

## Points clés

- Image immuable, conteneur remplaçable, données persistantes ailleurs.
- Reproductibilité = code + dépendances + configuration documentée.
- Un conteneur n’est pas une VM ni une garantie automatique de sécurité.

## Quiz rapide

1. Où stocker une base PostgreSQL ?  
   **Réponse :** dans un volume ou un service persistant, pas dans la couche éphémère du conteneur.
2. Une image en exécution est-elle modifiée par un `pip install` manuel ?  
   **Réponse :** seule la couche du conteneur change ; reconstruire l’image est la pratique reproductible.
3. Quel artefact doit être promu entre environnements ?  
   **Réponse :** la même image identifiée par tag contrôlé ou digest.

   # Module 01 — Installation et premiers pas

## Objectifs

Installer Docker selon son OS, lancer un conteneur, observer ses logs et nettoyer ses ressources.

## Installation

- **Linux** : installez Docker Engine et le plugin Compose depuis le dépôt officiel Docker de votre distribution ; ajoutez votre utilisateur au groupe `docker` seulement si votre politique de sécurité l’autorise, puis reconnectez-vous.
- **macOS** : installez Docker Desktop (Apple Silicon ou Intel adapté), démarrez-le et vérifiez les ressources allouées.
- **Windows** : installez Docker Desktop avec le backend WSL 2, activez l’intégration de la distribution Linux et ouvrez PowerShell ou un terminal WSL.

```bash
docker version
docker compose version
docker run --rm hello-world
```

`--rm` supprime le conteneur à l’arrêt. Le daemon télécharge l’image si elle manque localement.

## Commandes essentielles

```bash
# Démarrer un shell isolé puis le quitter
# (alpine est volontairement minimal pour apprendre)
docker run --name cli-demo -it alpine:3.20 sh

# Dans un autre terminal : lister les conteneurs actifs puis tous les conteneurs
docker ps
docker ps -a

# Observer et inspecter
docker logs cli-demo
docker inspect cli-demo

# Arrêter, supprimer ; l’image reste disponible
docker stop cli-demo
docker rm cli-demo
```

Pour un service en arrière-plan, utilisez `-d`, mappez explicitement le port avec `-p hôte:conteneur` et nommez la ressource. `docker exec -it nom sh` ouvre un nouveau processus dans un conteneur existant : ce n’est pas le processus applicatif principal.

```bash
docker run -d --name postgres-demo \
  -e POSTGRES_PASSWORD=local-only \
  -p 5432:5432 postgres:16-alpine
docker exec -it postgres-demo psql -U postgres -c 'SELECT version();'
docker logs --tail=50 postgres-demo
docker stop postgres-demo && docker rm postgres-demo
```

## Exercice

Lancez PostgreSQL sans port publié, trouvez son état avec `docker ps`, consultez les 20 dernières lignes de logs puis supprimez-le.

### Correction

```bash
docker run -d --name pg-lab -e POSTGRES_PASSWORD=local-only postgres:16-alpine
docker ps --filter name=pg-lab
docker logs --tail=20 pg-lab
docker stop pg-lab
docker rm pg-lab
```

## Points clés

- `run` crée et démarre ; `start` redémarre un conteneur existant.
- `ps -a`, `logs`, `inspect` sont les premiers outils de diagnostic.
- Ne publiez un port que si l’hôte doit réellement accéder au service.

## Quiz rapide

1. Que fait `--rm` ? **Supprime le conteneur après son arrêt.**
2. `docker exec` reconstruit-il une image ? **Non, il lance une commande dans un conteneur.**
3. Pourquoi `-p 5432:5432` ? **Pour rendre le port 5432 du conteneur accessible via le port 5432 de l’hôte.**


# Module 02 — Images et registres

## Objectifs

Télécharger, identifier, inspecter et publier une image sans confondre tag mutable et digest immuable.

## Anatomie

Une image est composée de couches réutilisables. `python:3.12-slim` est un tag lisible mais peut évoluer ; un digest `sha256:...` identifie le contenu exact. Docker Hub est un registre public ; une entreprise utilise souvent GHCR, ECR, GCR ou un registre privé.

```bash
docker pull python:3.12-slim
docker image ls
docker image history python:3.12-slim
docker image inspect python:3.12-slim
```

Pour une analyse de chaîne d’approvisionnement, vérifiez l’architecture et les métadonnées :

```bash
docker manifest inspect python:3.12-slim
# Taguer localement une image construite
# docker tag my-pipeline:dev ghcr.io/org/my-pipeline:0.1.0
# Publier après authentification (jamais avec un mot de passe dans l’historique shell)
# echo "$GHCR_TOKEN" | docker login ghcr.io -u USERNAME --password-stdin
# docker push ghcr.io/org/my-pipeline:0.1.0
```

Les tags conseillés : `1.4.2` pour une release, éventuellement `1.4`, et `git-<sha>` pour le lien CI. Évitez `latest` dans un déploiement critique.

## Exercice

Comparez la taille de `python:3.12` et `python:3.12-slim`, puis expliquez pourquoi une image plus petite peut manquer d’outils de compilation.

### Correction

```bash
docker pull python:3.12
docker pull python:3.12-slim
docker image ls python
```

`slim` retire de nombreux paquets et caches ; il réduit surface d’attaque et transfert, mais une dépendance native peut nécessiter gcc/headers pendant le build. Une approche propre est le multi-stage (module 07), pas l’installation de compilateurs en production.

## Points clés

- Un tag est une étiquette ; le digest est l’identité de contenu.
- Inspectez architecture, couches et provenance avant de promouvoir.
- Le registre est un composant de distribution et de gouvernance, pas seulement un disque distant.

## Quiz rapide

1. `pull` fait-il tourner l’image ? **Non, il la télécharge.**
2. Pourquoi éviter `latest` ? **Il est ambigu et peut changer sans changement de déploiement explicite.**
3. Où authentifier un pipeline ? **Avec un secret CI à durée et portée minimales.**


# Module 03 — Écrire un Dockerfile

## Objectifs

Comprendre les instructions essentielles, construire une image Python et réduire le contexte avec `.dockerignore`.

## Instructions et ordre

`FROM` choisit la base ; `WORKDIR` le répertoire courant ; `COPY` ajoute des fichiers ; `RUN` exécute pendant le build ; `ENV` définit une valeur par défaut ; `EXPOSE` documente un port (il ne le publie pas) ; `CMD` fournit la commande par défaut ; `ENTRYPOINT` fixe l’exécutable.

Chaque instruction peut produire une couche. Placez les fichiers de dépendances avant le code pour conserver le cache.

```dockerfile
# labs/python-pipeline/Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
WORKDIR /app

# Installer d'abord les dépendances : cette couche est réutilisable si le code change
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/
USER 10001
CMD ["python", "src/pipeline.py"]
```

`.dockerignore` :

```text
.git
.venv
__pycache__
*.pyc
models/
.env
notebooks/.ipynb_checkpoints
```

Construire et exécuter :

```bash
docker build -t data-pipeline:0.1 labs/python-pipeline
docker run --rm data-pipeline:0.1
```

`CMD ["python", ...]` est de forme exec : signaux et arrêt sont mieux propagés. `ENTRYPOINT` convient à une image outil dont l’exécutable ne doit pas être remplacé facilement ; on peut compléter avec un `CMD` d’arguments.

## Exercice

Ajoutez un argument `--limit` au script, faites-le passer au conteneur et vérifiez qu’un fichier `.env` n’est pas envoyé dans le contexte.

### Correction

```dockerfile
# Préférer la forme exec pour transmettre les signaux
ENTRYPOINT ["python", "src/pipeline.py"]
CMD ["--limit", "100"]
```

```bash
docker build --progress=plain -t data-pipeline:exercise labs/python-pipeline
docker run --rm data-pipeline:exercise --limit 10
```

Un `.dockerignore` réduit le contexte, mais ne remplace pas la règle « aucun secret dans le dépôt ».

## Points clés

- `EXPOSE` n’ouvre pas de port ; `-p` le publie.
- Ordre des couches = performance du build.
- Ne faites pas de `pip install` manuel dans un conteneur destiné à être reproduit.

## Quiz rapide

1. `RUN` s’exécute quand ? **Au build.**
2. Différence CMD/ENTRYPOINT ? **CMD est une valeur par défaut ; ENTRYPOINT fixe le programme.**
3. Pourquoi `--no-cache-dir` ? **Ne pas conserver les archives pip dans l’image finale.**

# Module 04 — Cycle de vie et stockage

## Objectifs

Distinguer couche éphémère, volume nommé et bind mount ; appliquer une stratégie de persistance et de sauvegarde.

À l’arrêt, le conteneur peut être redémarré, mais sa couche écrivable ne constitue pas une sauvegarde. Un **volume nommé** est géré par Docker et adapté aux données d’un service. Un **bind mount** relie un chemin de l’hôte : utile pour du code en développement, plus couplé et plus risqué en production.

```bash
docker volume create pgdata
docker run -d --name pg \
  -e POSTGRES_PASSWORD=local-only \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
docker volume inspect pgdata
```

Le mode lecture seule est une défense simple : `-v ./config:/app/config:ro`. Ne montez jamais `/var/run/docker.sock` à la légère : il donne généralement un contrôle privilégié du daemon.

## Sauvegarde PostgreSQL

```bash
# Export logique depuis le conteneur vers l’hôte
docker exec pg pg_dump -U postgres postgres > backup.sql
# Restaurer dans une base neuve (exemple de lab)
cat backup.sql | docker exec -i pg psql -U postgres postgres
```

En production, définissez RPO/RTO, chiffrez les sauvegardes, testez la restauration et surveillez l’espace disque. Un volume n’est pas une sauvegarde.

## Exercice

Créez un volume, écrivez un fichier dans Alpine, supprimez le conteneur, puis relancez avec le volume et relisez le fichier.

### Correction

```bash
docker volume create demo-data
docker run --rm -v demo-data:/data alpine:3.20 sh -c 'echo persistent > /data/state.txt'
docker run --rm -v demo-data:/data alpine:3.20 cat /data/state.txt
docker volume rm demo-data
```

## Points clés

- Remplacer un conteneur est normal ; perdre une base ne l’est pas.
- Choisissez explicitement ownership, permissions, sauvegarde et restauration.
- Un bind mount est pratique pour développer ; un volume est plus portable pour un service.

## Quiz rapide

1. Un volume survivra-t-il à `docker rm` ? **Oui, sauf suppression explicite ou `down -v`.**
2. Un volume garantit-il la durabilité matérielle ? **Non. Il faut sauvegarde et stockage adapté.**
3. Pourquoi `:ro` ? **Réduire les écritures et la possibilité de modification par le processus.**


# Module 05 — Réseaux Docker

## Objectifs

Créer un réseau, comprendre DNS inter-conteneurs et choisir entre bridge et host.

Sur un réseau utilisateur de type `bridge`, Docker fournit un DNS interne : les conteneurs se joignent par nom de service ou alias. Le conteneur utilise le **port interne** ; la publication vers l’hôte est indépendante.

```bash
docker network create data-net
docker run -d --name db --network data-net \
  -e POSTGRES_PASSWORD=local-only postgres:16-alpine
docker run --rm --network data-net postgres:16-alpine \
  psql -h db -U postgres -c 'SELECT 1;'
docker network inspect data-net
docker rm -f db; docker network rm data-net
```

- `bridge` : isolation et communication contrôlée ; choix par défaut pour une stack.
- `host` : partage du réseau de l’hôte, utile dans quelques cas de performance, mais moins isolé et indisponible de façon identique selon les OS.
- `none` : pas de réseau.

Dans Compose, utilisez `db:5432`, jamais `localhost:5432` depuis un autre service : `localhost` désigne le conteneur courant.

## Exercice

Pourquoi un script Python dans `ingestor` échoue-t-il avec `DB_HOST=localhost` ? Corrigez sa configuration.

### Correction

`localhost` pointe vers `ingestor`, où PostgreSQL ne tourne pas. Il faut `DB_HOST=db` (ou le nom Compose `postgres`) et le port interne 5432. Depuis l’hôte, on utiliserait le port publié.

## Points clés

- DNS de service > adresses IP codées en dur.
- Ne publiez pas la base si seul le réseau interne en a besoin.
- Le réseau fait partie du contrat d’architecture et doit être observé (`network inspect`).

## Quiz rapide

1. `localhost` depuis un conteneur désigne quoi ? **Le conteneur lui-même.**
2. Port à utiliser entre services PostgreSQL ? **5432, sans dépendre du port hôte.**
3. Bridge ou host pour isoler une base ? **Bridge.**

# Module 06 — Docker Compose : une stack data locale

## Objectifs

Lire un fichier Compose, orchestrer PostgreSQL, pgAdmin et une ingestion, utiliser healthchecks et profiles.

Compose décrit les services déclaratifs : image/build, environnement, volumes, réseaux, dépendances. `depends_on` ordonne le démarrage mais ne prouve pas qu’un service est prêt ; le `healthcheck` et `condition: service_healthy` ferment cette lacune. `profiles` permet d’activer un outil optionnel.

## Lab guidé

Le fichier [`labs/compose-data/docker-compose.yml`](../labs/compose-data/docker-compose.yml) contient :

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: warehouse, POSTGRES_USER: app, POSTGRES_PASSWORD: local-only }
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d warehouse"]
      interval: 5s
      timeout: 3s
      retries: 10
  ingestor:
    build: ./ingestor
    environment: { DB_HOST: postgres, DB_NAME: warehouse, DB_USER: app, DB_PASSWORD: local-only }
    depends_on:
      postgres: { condition: service_healthy }
  pgadmin:
    image: dpage/pgadmin4:8
    profiles: [debug]
    environment: { PGADMIN_DEFAULT_EMAIL: admin@example.local, PGADMIN_DEFAULT_PASSWORD: local-only }
    ports: ["8080:80"]
volumes: { pgdata: {} }
```

Depuis la racine :

```bash
docker compose -f labs/compose-data/docker-compose.yml up --build
# Dans un autre terminal
docker compose -f labs/compose-data/docker-compose.yml --profile debug up -d pgadmin
docker compose -f labs/compose-data/docker-compose.yml ps
docker compose -f labs/compose-data/docker-compose.yml logs -f ingestor
docker compose -f labs/compose-data/docker-compose.yml down -v
```

L’ingestor crée une table et insère des lignes idempotentes. En réel, séparez secrets, migrations, validation de schéma et observabilité.

## Exercice et correction

Ajoutez une healthcheck à l’ingestor et rendez l’insertion relançable. Une correction : utiliser `CREATE TABLE IF NOT EXISTS`, une clé naturelle avec `ON CONFLICT DO NOTHING`, puis :

```yaml
healthcheck:
  test: ["CMD-SHELL", "python -c 'import os; print("ok")'"]
  interval: 10s
  timeout: 3s
  retries: 3
```

Attention : une healthcheck « import Python » ne valide pas la qualité métier. Testez aussi une métrique ou une requête de contrôle.

## Points clés

- Compose est excellent pour dev, intégration et démonstration ; pas un scheduler de production complet.
- `depends_on` n’attend la santé que si la condition est configurée.
- `down -v` détruit les données du lab.

## Quiz rapide

1. Pourquoi un healthcheck ? **Distinguer processus lancé et service prêt.**
2. Comment activer pgAdmin ? **`--profile debug`.**
3. Nom DNS de PostgreSQL ? **`postgres`, nom du service.**

# Module 07 — Bonnes pratiques pour les images data

## Objectifs

Réduire taille, temps de build et surface d’attaque ; gérer pip, conda et uv ; exploiter le cache.

Principes : image slim ou distroless compatible avec le runtime, versions contraintes, wheels binaires, nettoyage des caches, contexte minimal, utilisateur non-root, build reproductible et scan. Pour une stack scientifique lourde, conda/micromamba peut simplifier les bibliothèques natives ; pip avec wheels est plus léger ; `uv` accélère résolution et installation mais doit être versionné et contrôlé.

### Multi-stage

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
RUN python -m venv /venv
COPY requirements.txt .
RUN /venv/bin/pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS runtime
ENV PATH="/venv/bin:$PATH" PYTHONUNBUFFERED=1
COPY --from=builder /venv /venv
COPY src /app/src
WORKDIR /app
RUN useradd --create-home --uid 10001 appuser
USER appuser
CMD ["python", "src/pipeline.py"]
```

Le builder peut contenir compilateur et headers ; l’image finale ne garde que l’environnement runtime. `docker buildx build --platform linux/amd64,linux/arm64` produit plusieurs architectures si toutes les dépendances le permettent.

### Cache et déterminisme

```dockerfile
# requirements.txt doit être copié avant le code
COPY requirements.txt .
RUN pip install --no-cache-dir --require-hashes -r requirements.txt
```

Les hashes empêchent une substitution silencieuse. Pinnez l’image de base par digest après validation. Utilisez BuildKit cache mounts avec prudence et ne mettez pas de secret dans une couche.

## Exercice

Mesurez une image qui copie tout le dépôt, puis une image avec `.dockerignore` et multi-stage. Notez les couches avec `docker history`.

### Correction

```bash
docker image ls data-pipeline
docker history data-pipeline:0.1
# comparer avec --no-cache pour distinguer cache et taille réelle
docker build --no-cache -t data-pipeline:clean labs/python-pipeline
```

Supprimer notebooks, datasets, caches et outils de compilation est généralement le gain principal. Ne sacrifiez pas une librairie runtime nécessaire uniquement pour atteindre un nombre de mégaoctets.

## Points clés / quiz

- Le plus gros gain vient souvent du contexte et du multi-stage.
- Lockfile/hashes et digest améliorent la reproductibilité.
- **Quiz :** où vont gcc et headers ? Dans le stage builder ; **que fait `--no-cache` ?** reconstruit sans couches réutilisées ; **distroless convient-il à un shell de debug ?** non, il faut un conteneur de debug séparé.

# Module 08 — Cas d’usage data engineering

## Objectifs

Containeriser un ETL, comprendre le rôle d’Airflow, connecter MinIO et choisir un scheduler adapté.

Un conteneur ETL doit être stateless : il lit une source, écrit une destination, expose des métriques et peut être relancé sans doublon. Le lab Python utilise une source CSV synthétique et PostgreSQL ; pour du stockage objet compatible S3, MinIO est utile en local. Airflow apporte DAG, retries, dépendances et UI, mais sa distribution Docker complète (webserver, scheduler, metadata DB, worker) demande des ressources et une configuration dédiée.

```yaml
# Fragment conceptuel d’un service MinIO local
minio:
  image: minio/minio:RELEASE.2024-06-13T22-53-53Z
  command: server /data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: localadmin
    MINIO_ROOT_PASSWORD: localpassword
  volumes: [minio-data:/data]
  ports: ["9000:9000", "9001:9001"]
```

Ne mettez pas une boucle infinie dans un conteneur simplement pour imiter cron : un orchestrateur externe (Airflow, Kubernetes CronJob, système CI) gère mieux retry, calendrier et historique. Pour un petit lab, un job one-shot lancé par `docker compose run --rm ingestor` est explicite.

### Connecter une source

Utilisez des variables `SOURCE_URL`, `DB_HOST` et un secret injecté. Validez schéma, encodage, timezone et volume avant transformation ; écrivez un manifeste de lot (identifiant, nombre de lignes, checksum, timestamp).

## Exercice et correction

Rendez l’ETL idempotent et rejouable. Correction : calculer `batch_id = sha256(source_bytes)`, stocker ce batch dans une table de contrôle avec contrainte unique, puis faire la transaction métier et le contrôle dans la même unité transactionnelle. Un retry doit être sûr.

```bash
docker compose -f labs/compose-data/docker-compose.yml run --rm ingestor
```

## Points clés / quiz

- Un ETL conteneurisé doit être observable et relançable.
- Airflow orchestre ; il ne remplace pas data quality ni lineage.
- **Quiz :** où conserver un gros CSV ? objet/volume adapté, pas dans l’image ; **pourquoi batch_id ?** idempotence ; **scheduler dans le conteneur ?** seulement pour un lab très simple.



# Module 09 — Cas d’usage MLOps

## Objectifs

Servir un modèle avec FastAPI, suivre un entraînement dans MLflow, séparer artefacts et image, utiliser Jupyter sans contaminer la production.

Le serving doit charger une version de modèle explicite au démarrage, valider les entrées, exposer `/health` et `/predict`, journaliser latence et version, et refuser les données sensibles dans les logs. Le modèle et ses artefacts vivent dans un registry (MLflow, stockage objet), pas forcément dans l’image.

```python
# Exemple minimal : labs/ml-serving/app/main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="score-api")
class Features(BaseModel):
    amount: float
    age: int

@app.get("/health")
def health():
    return {"status": "ok", "model_version": "2026-01-01"}

@app.post("/predict")
def predict(x: Features):
    # Remplacer cette logique par un artefact chargé et versionné
    score = 0.02 * x.amount + 0.1 * x.age
    return {"score": score, "model_version": "2026-01-01"}
```

MLflow Tracking Server centralise runs, paramètres, métriques et artefacts. En local, un Compose peut le relier à PostgreSQL (backend store) et MinIO (artifact store) ; en production, protégez l’endpoint, chiffrez le transport et contrôlez les permissions. Jupyter est un espace d’exploration isolé ; ne réutilisez pas automatiquement son image en serving.

```bash
docker build -t score-api:0.1 labs/ml-serving
docker run --rm -p 8000:8000 score-api:0.1
curl -X POST http://localhost:8000/predict -H 'content-type: application/json' -d '{"amount":100,"age":30}'
```

## Exercice

Proposez une promotion dev → staging → prod. Correction : entraîner et enregistrer l’artefact avec métriques et dataset fingerprint ; valider automatiquement ; approuver une version candidate ; déployer le même digest d’image avec `MODEL_URI` immuable ; surveiller dérive, latence et erreurs ; rollback vers la version précédente.

## Points clés / quiz

- Une image n’est pas un registry de modèles.
- Healthcheck de processus ≠ qualité du modèle.
- **Quiz :** où versionner poids ? registry d’artefacts ; **que renvoie `/health` ?** état et version ; **pourquoi même digest ?** éviter une reconstruction différente entre environnements.


# Module 10 — Docker et GPU

## Objectifs

Comprendre le runtime NVIDIA, choisir une image CUDA et diagnostiquer un entraînement GPU.

Sur Linux, installez le pilote NVIDIA puis le NVIDIA Container Toolkit selon la documentation officielle. Docker Desktop peut exposer le GPU selon l’OS et sa configuration ; vérifiez toujours la compatibilité pilote ↔ CUDA ↔ framework.

```bash
# Test de visibilité du GPU dans le conteneur
 docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Pour PyTorch, partez d’une image officielle compatible ou construisez avec un runtime CUDA validé. `--gpus all` autorise les GPU ; en production, limitez le nombre et documentez la mémoire attendue.

```bash
# Exemple d’entraînement du lab (script à adapter au framework)
docker run --rm --gpus 'device=0' \
  -v "$PWD/labs/gpu-training:/workspace" \
  -w /workspace pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime \
  python train.py
```

Le modèle, les checkpoints et datasets ne doivent pas être écrits dans la couche éphémère. Montez un volume ou utilisez un stockage objet. Pour la reproductibilité, enregistrez image digest, commit, versions CUDA/driver, seed et configuration.

## Exercice / correction

Si `nvidia-smi` fonctionne sur l’hôte mais échoue dans le conteneur, vérifiez : toolkit installé, daemon redémarré, option `--gpus`, version du pilote compatible et accès au device. Comparez `nvidia-smi` hôte/conteneur ; ne réinstallez pas un pilote dans l’image.

## Points clés / quiz

- Le pilote est fourni par l’hôte ; les bibliothèques user-space CUDA viennent généralement de l’image.
- GPU visible ne signifie pas entraînement efficace : mesurez utilisation, mémoire et débit.
- **Quiz :** option clé ? `--gpus` ; checkpoint ? volume/objet ; premier diagnostic ? `nvidia-smi`.

# Module 11 — Sécurité et production

## Objectifs

Réduire privilèges, scanner une image, gérer secrets et limites de ressources.

```dockerfile
# Dans l’image runtime : utilisateur non-root et filesystem applicatif limité
RUN useradd --create-home --uid 10001 appuser
USER 10001
```

Ne mettez jamais secret, token cloud ou clé SSH dans un Dockerfile, une variable `ENV` persistée ou une couche Git. En local, Compose secrets ou fichier monté en lecture seule ; en production, secret manager de la plateforme, rotation et audit. Évitez le socket Docker, `--privileged`, capabilities inutiles et images inconnues.

```bash
# Scanner localement (installer Trivy séparément)
trivy image --severity HIGH,CRITICAL --ignore-unfixed score-api:0.1
# Limiter un service :
docker run --read-only --cap-drop=ALL --pids-limit=200 \
  --memory=1g --cpus=1.0 score-api:0.1
```

Un scan n’est pas une preuve d’absence de risque : corrigez base et dépendances, analysez SBOM, signez et vérifiez la provenance (Cosign/Notary selon votre registre). Les images distroless réduisent outils et shell ; gardez une stratégie de debug externe. En production : logs structurés sans PII, TLS, réseau privé, healthchecks, quotas, restart policy, backups et alertes.

## Exercice / correction

Écrivez une checklist avant publication : utilisateur, digest base, lockfile, secrets absents, scan, SBOM, signature, ports minimaux, limites CPU/RAM, healthcheck, rollback. Refusez le build si une vulnérabilité critique exploitable est détectée, avec une exception documentée et expirante.

## Points clés / quiz

- Moindre privilège à chaque couche.
- Les secrets sont des données runtime, jamais du code source.
- **Quiz :** `--read-only` ? filesystem racine non inscriptible ; Trivy ? analyse vulnérabilités ; signature ? authenticité/intégrité, pas qualité métier.


# Module 12 — CI/CD et orchestration

## Objectifs

Automatiser tests/build/scan/push et comprendre le passage vers Kubernetes.

Un pipeline type : lint et tests → build avec BuildKit → scan/SBOM → signature → push vers GHCR/ECR → déploiement progressif. Publiez uniquement sur branche protégée ou tag signé ; utilisez OIDC plutôt qu’une clé cloud longue durée.

```yaml
# labs/ci/github-actions.yml
name: container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: labs/ml-serving
          push: false
          tags: ghcr.io/example/score-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Ajoutez authentification du registre, tests, Trivy et signature avant `push: true`. Un registre privé fournit RBAC, rétention, réplication et contrôle d’images.

### Kubernetes : les concepts

Kubernetes ne lance pas directement un Dockerfile : il orchestre des images OCI via un runtime compatible. Un **Pod** regroupe des conteneurs ; un **Deployment** gère replicas et rollout ; un **Service** fournit découverte réseau ; un **ConfigMap/Secret** injecte la configuration ; un **PVC** demande du stockage ; un **Job/CronJob** exécute une tâche batch.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: score-api }
spec:
  replicas: 2
  selector: { matchLabels: { app: score-api } }
  template:
    metadata: { labels: { app: score-api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/example/score-api:git-sha
          ports: [{ containerPort: 8000 }]
          resources: { requests: { cpu: "250m", memory: "256Mi" }, limits: { cpu: "1", memory: "1Gi" } }
          readinessProbe: { httpGet: { path: /health, port: 8000 } }
```

Pour MLOps, séparez online serving, batch inference, training jobs, GPU node pools et model registry. Stratégies : rolling update, blue/green, canary ; rendez le rollback possible et observez erreurs, latence, saturation, dérive.

## Exercice / correction

Transformez le service Compose en Deployment + Service + Secret. Correction : image publiée par digest, Secret pour credentials, readiness/liveness probes, requests/limits, Service interne et Ingress contrôlé ; PVC seulement pour stockage réellement persistant.

## Points clés / quiz

- CI produit un artefact vérifié ; CD le promeut.
- Kubernetes ajoute contrôle déclaratif, auto-réparation et scheduling, pas une magie de sécurité.
- **Quiz :** Job ou Deployment pour entraînement ponctuel ? Job ; Service ? DNS/point d’accès stable ; readiness ? autoriser le trafic seulement quand l’app est prête.


