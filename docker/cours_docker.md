# Docker, de débutant à avancé

## Parcours Data Engineer junior → MLOps Engineer

**Objectif.** Ce cours explique Docker par la pratique, depuis les premiers conteneurs jusqu’à une petite API de prédiction et une publication dans **Google Artifact Registry (GCP)**. Les exemples sont volontairement reproductibles sur Linux/macOS ; les notes Windows indiquent les adaptations utiles.

> **Ce que Docker ne fait pas :** Docker empaquette et exécute des applications de façon reproductible. Il ne remplace ni Kubernetes (orchestration, haute disponibilité, déploiement multi-nœuds), ni Terraform (provisionnement déclaratif de l’infrastructure), ni un système complet d’observabilité, de secrets ou de gouvernance. En production, Docker est souvent une brique parmi d’autres.

---

## 1. Le modèle mental

### Image, conteneur et registre

- **Image** : artefact immuable composé de couches en lecture seule. Elle contient un système minimal, les dépendances et l’application, mais pas le noyau Linux.
- **Conteneur** : processus isolé lancé à partir d’une image. Sa couche d’écriture est éphémère : supprimer le conteneur supprime cette couche.
- **Registre** : dépôt d’images, public ou privé. Docker Hub est un registre ; **Google Artifact Registry** est le registre GCP utilisé dans ce cours.
- **Tag** : alias lisible (`1.4.0`, `latest`). En production, préférer un tag de version et/ou le digest (`sha256:…`) plutôt que `latest` seul.
- **Docker Engine** : daemon (`dockerd`) et client CLI (`docker`). Docker Desktop regroupe l’Engine et une VM sur macOS/Windows.

Flux typique :

```text
Dockerfile + code → docker build → image → docker run → conteneur
                                      ↓
                              docker push → registre
```

### Isolation et limites

Un conteneur partage le noyau de l’hôte Linux. Il n’est donc pas une VM et l’isolation n’est pas une garantie absolue. Les namespaces isolent les processus, le réseau et certains points de montage ; les cgroups permettent notamment de limiter CPU et mémoire.

### Volumes et bind mounts

- **Volume nommé** : géré par Docker, adapté aux données persistantes d’un service.
- **Bind mount** : répertoire de l’hôte monté dans le conteneur, pratique pour le développement (`.:/app`) mais plus couplé à la machine.
- **tmpfs** : stockage en mémoire, utile pour des données temporaires et sensibles ; non persistant.

### Réseaux et ports

Les conteneurs d’un même réseau Docker se joignent par **nom de service** et port interne, par exemple `postgres:5432`. Le port `5432` n’a pas besoin d’être publié sur l’hôte pour être accessible depuis un autre conteneur. La syntaxe `-p 15432:5432` signifie `hôte:conteneur`.

---

## 2. Installation et vérification

### Linux (Ubuntu/Debian, approche simple)

Pour un poste de développement, Docker Desktop ou les paquets officiels Docker Engine conviennent. Après installation, vérifier :

```bash
docker --version
docker compose version
docker run --rm hello-world
docker info
```

Sur Linux, l’ajout de l’utilisateur au groupe Docker évite `sudo`, mais ce groupe donne des privilèges élevés :

```bash
sudo usermod -aG docker "$USER"
# Se déconnecter/reconnecter, puis :
docker run --rm hello-world
```

Si le daemon n’est pas démarré :

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
```

### macOS et Windows

- Installer **Docker Desktop** puis attendre que l’Engine soit démarré.
- Windows : activer WSL 2 si Docker Desktop le propose ; lancer les commandes dans PowerShell ou WSL.
- Les chemins de volumes sont plus simples depuis WSL ou un répertoire partagé par Docker Desktop.

```powershell
# PowerShell : mêmes commandes Docker
 docker --version
 docker compose version
 docker run --rm hello-world
```

Si `docker compose` n’existe pas, installer une version récente de Docker Desktop ; l’ancien `docker-compose` est à éviter pour les nouveaux projets.

### Diagnostic initial

```bash
docker context ls
docker system df
docker ps
```

Ne jamais lancer `docker system prune -a --volumes` sans vérifier : cette commande peut supprimer des images, conteneurs et volumes inutilisés.

---

## 3. Commandes essentielles

### Images

```bash
docker pull python:3.12-slim

docker images

docker image ls

docker image inspect python:3.12-slim
docker history python:3.12-slim
docker rmi python:3.12-slim
```

### Conteneurs

```bash
docker run --name demo --rm python:3.12-slim python -c "print('bonjour')"
docker run -d --name web -p 8080:80 nginx:alpine
docker ps
docker ps -a
docker logs -f web
docker exec -it web sh
docker inspect web
docker stop web
docker rm web
```

`--rm` supprime le conteneur à sa sortie. `-d` le lance en arrière-plan. Un conteneur n’est pas une machine à administrer : le processus principal doit rester au premier plan.

### Construire et taguer

Depuis le répertoire qui contient un `Dockerfile` :

```bash
docker build -t demo-api:0.1.0 .
docker run --rm -p 8000:8000 demo-api:0.1.0
docker tag demo-api:0.1.0 localhost:5000/demo-api:0.1.0
```

Le contexte `.` est envoyé au builder : un `.dockerignore` est donc important.

### Nettoyage prudent

```bash
docker container prune       # conteneurs arrêtés
 docker image prune           # couches inutilisées
docker volume ls
docker network ls
```

Avant toute suppression, identifier ce qui est utilisé :

```bash
docker ps -a --size
docker system df -v
```

---

## 4. Premier Dockerfile, puis progression

### Étape 1 — statique avec Nginx

`Dockerfile` :

```dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

`index.html` :

```html
<!doctype html>
<html lang="fr"><meta charset="utf-8"><title>Docker</title>
<body><h1>Mon premier conteneur</h1></body></html>
```

```bash
docker build -t cours-web:1.0 .
docker run --rm --name cours-web -p 8080:80 cours-web:1.0
# Dans un autre terminal :
curl http://localhost:8080
```

`EXPOSE` documente le port ; il ne le publie pas. C’est `-p` qui publie le port.

### Étape 2 — application Python reproductible

Arborescence :

```text
hello-python/
├── Dockerfile
├── .dockerignore
├── requirements.txt
└── app.py
```

`app.py` :

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import os

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = f"Bonjour depuis {os.getenv('APP_NAME', 'Docker')}\n".encode()
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, fmt, *args):
        print(fmt % args, flush=True)

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

`requirements.txt` peut être vide pour cet exemple. `Dockerfile` :

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

`.dockerignore` :

```text
.git
.gitignore
__pycache__
*.py[cod]
.venv
.env
.pytest_cache
.coverage
```

```bash
docker build -t hello-python:1.0 .
docker run --rm --name hello-python -p 8000:8000 -e APP_NAME=junior-mle hello-python:1.0
curl http://localhost:8000
```

### `CMD` et `ENTRYPOINT`

`CMD` fournit la commande par défaut et peut être remplacé à la fin de `docker run`. `ENTRYPOINT` fixe le programme principal ; les arguments fournis ensuite lui sont ajoutés. Utiliser la forme JSON (`["python", "app.py"]`) pour une gestion correcte des signaux et éviter un shell inutile.

---

## 5. Bonnes pratiques et sécurité

1. **Image minimale et version figée** : `python:3.12-slim` plutôt qu’une image complète ; épingler une version, voire un digest après validation.
2. **Utilisateur non root** :

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser app.py .
USER appuser
CMD ["python", "app.py"]
```

3. **Contexte réduit** : `.dockerignore`, pas de `.git`, données brutes, notebooks ni secrets.
4. **Pas de secret dans une image** : ni `ENV`, ni `ARG`, ni `COPY .` contenant `.env`. Les couches peuvent rester dans l’historique même après suppression.
5. **Scanner** : utiliser un outil validé par l’équipe (par exemple Docker Scout ou Trivy) et traiter les vulnérabilités selon leur exploitabilité, pas seulement leur nombre.
6. **Build reproductible** : dépendances versionnées (`requirements.txt` ou lockfile), tags immuables, build automatisé.
7. **Système de fichiers en lecture seule** lorsque possible :

```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop=ALL --security-opt=no-new-privileges \
  --rm hello-python:1.0
```

Adaptez ces options si l’application doit écrire ailleurs ou requiert une capacité explicite. Ne pas ajouter `--privileged` par facilité.

### Variables et secrets

Une configuration non sensible peut être injectée :

```bash
docker run --rm -e APP_NAME=dev hello-python:1.0
```

Un fichier `.env` local ne doit pas être commité :

```text
APP_NAME=dev
# API_KEY=valeur-locale-uniquement
```

Compose charge souvent `.env` pour l’interpolation. Cela ne rend pas le secret sûr si le fichier est envoyé dans le contexte ou copié dans l’image. En CI/CD ou en production, utiliser un gestionnaire de secrets (Secret Manager GCP, mécanisme de secrets de l’orchestrateur, ou secret store de la plateforme) et injecter au dernier moment. Aucun secret réel n’est utilisé dans ce cours.

---

## 6. Persistance : PostgreSQL pour la data

Un conteneur PostgreSQL est pratique pour le développement, pas une stratégie de sauvegarde de production à lui seul.

```bash
docker volume create pgdata

docker run -d --name postgres-dev \
  -e POSTGRES_USER=de_user \
  -e POSTGRES_PASSWORD='dev-only-change-me' \
  -e POSTGRES_DB=warehouse \
  -p 15432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Le mot de passe ci-dessus est uniquement un exemple local.
docker exec -it postgres-dev psql -U de_user -d warehouse
```

Tester :

```sql
CREATE TABLE events (id serial PRIMARY KEY, event_type text NOT NULL, created_at timestamptz DEFAULT now());
INSERT INTO events(event_type) VALUES ('prediction_requested');
SELECT * FROM events;
```

Le volume survit à `docker rm postgres-dev`. Pour repartir de zéro en local :

```bash
docker rm -f postgres-dev
docker volume rm pgdata
```

En production, préférer une base managée (par exemple Cloud SQL for PostgreSQL sur GCP) ou une architecture opérée avec sauvegardes, réplication, chiffrement et restauration testée.

---

## 7. Réseau Docker

```bash
docker network create data-net
docker run -d --name db --network data-net \
  -e POSTGRES_PASSWORD=dev-only \
  -e POSTGRES_USER=app -e POSTGRES_DB=appdb postgres:16

docker run --rm --network data-net postgres:16 \
  pg_isready -h db -U app -d appdb
```

Dans `data-net`, le nom `db` résout vers le conteneur. Depuis l’hôte, le port n’est pas publié ici : il n’est donc pas joignable par `localhost:5432`.

Règles pratiques : créer un réseau par application, ne publier que les ports nécessaires et distinguer réseau interne et réseau exposé. Compose crée automatiquement un réseau de projet.

---

## 8. Docker Compose : application + PostgreSQL

### Projet cohérent

```text
compose-demo/
├── compose.yaml
├── .env.example
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
└── db/init.sql
```

`api/requirements.txt` :

```text
fastapi==0.115.6
uvicorn[standard]==0.34.0
psycopg[binary]==3.2.3
```

`api/main.py` :

```python
import os
from fastapi import FastAPI
import psycopg

app = FastAPI(title="demo-data-api")

def connection():
    return psycopg.connect(
        host=os.getenv("DB_HOST", "db"), port=5432,
        dbname=os.getenv("POSTGRES_DB", "warehouse"),
        user=os.getenv("POSTGRES_USER", "de_user"),
        password=os.environ["POSTGRES_PASSWORD"],
    )

@app.get("/health")
def health():
    with connection() as conn:
        conn.execute("SELECT 1")
    return {"status": "ok"}

@app.get("/events/count")
def count():
    with connection() as conn:
        row = conn.execute("SELECT count(*) FROM events").fetchone()
    return {"count": row[0]}
```

`api/Dockerfile` :

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
USER appuser
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`db/init.sql` :

```sql
CREATE TABLE IF NOT EXISTS events (
  id bigserial PRIMARY KEY,
  event_type text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

`.env.example` :

```text
POSTGRES_USER=de_user
POSTGRES_PASSWORD=dev-only-change-me
POSTGRES_DB=warehouse
```

`compose.yaml` :

```yaml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/001-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build: ./api
    environment:
      DB_HOST: db
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pgdata:
```

Lancer :

```bash
cp .env.example .env
# Modifier .env localement ; ne pas le commiter.
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs -f api
curl http://localhost:8000/health
curl http://localhost:8000/events/count
docker compose down
# Pour supprimer aussi la base locale : docker compose down -v
```

`depends_on` avec healthcheck ordonne le démarrage, mais l’application doit quand même tolérer les reconnexions et les pannes transitoires. Les scripts dans `/docker-entrypoint-initdb.d` ne s’exécutent qu’à l’initialisation d’un volume PostgreSQL vide ; les migrations applicatives sont préférables pour un vrai projet.

### Exemple pipeline Python

Un pipeline batch ne doit pas rester bloqué dans un serveur web. `pipeline.py` peut être lancé avec un profil Compose ponctuel :

```python
import csv
from pathlib import Path

src = Path("/data/input.csv")
out = Path("/data/output.csv")
with src.open(newline="") as f:
    rows = list(csv.DictReader(f))
clean = [r for r in rows if r.get("event_type")]
with out.open("w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=clean[0].keys() if clean else ["event_type"])
    writer.writeheader()
    writer.writerows(clean)
print(f"rows_in={len(rows)} rows_out={len(clean)}", flush=True)
```

Commande :

```bash
docker run --rm -v "$PWD/data:/data" pipeline-image:1.0 python pipeline.py
```

Airflow peut orchestrer ce pipeline, mais il est plus lourd qu’un `docker compose up` : il faut au minimum scheduler, webserver, metadata DB et souvent un broker/worker selon l’exécuteur. Utilisez l’image et le `docker-compose.yaml` officiels compatibles avec la version choisie, consultez la documentation Airflow, et ne copiez pas un exemple de démonstration en production sans persistance, secrets, migrations, sauvegardes et dimensionnement.

---

## 9. Logs, debug et healthcheck

### Logs structurés et diagnostic

Un conteneur doit écrire ses logs applicatifs sur stdout/stderr :

```bash
docker compose logs --tail=100 api
docker logs --since=10m api

docker exec -it api sh
docker inspect --format '{{json .State}}' api
docker stats
docker top api
```

Codes utiles : `docker ps -a` affiche le code de sortie ; `137` indique souvent un kill lié à la mémoire, `1` une erreur applicative générique. Vérifier aussi les événements :

```bash
docker events --since 10m
```

### Healthcheck

Un healthcheck doit vérifier le service utile, pas seulement la présence du processus. Pour une API FastAPI :

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)" || exit 1
```

Dans Compose :

```yaml
healthcheck:
  test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
  interval: 30s
  timeout: 5s
  retries: 3
```

Un healthcheck défaillant ne redémarre pas automatiquement un conteneur dans tous les contextes ; il produit un état dont la plateforme peut se servir. Ajouter une politique `restart` avec discernement.

---

## 10. Optimisation et multi-stage builds

Chaque instruction peut créer une couche. Regrouper les opérations, supprimer les caches et tirer parti du cache de build :

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS runtime
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser src/ ./src/
USER appuser
CMD ["python", "-m", "src.main"]
```

Pour une application compilée, séparer builder et runtime :

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Pour Python avec dépendances natives, le principe est identique mais il faut transférer proprement les wheels ou l’environnement vers l’étape runtime. Ne pas sacrifier la lisibilité à quelques mégaoctets : mesurer avec `docker image ls` et `docker history`.

BuildKit/cache (si le builder le supporte) :

```bash
docker buildx build --tag demo-api:1.0 --load .
```

---

## 11. Limites de ressources

Sans limites, un conteneur peut épuiser la machine. En local :

```bash
docker run --rm --memory=512m --cpus=1.0 --pids-limit=200 hello-python:1.0
```

Compose :

```yaml
services:
  api:
    # Les champs ci-dessous sont compris par les implémentations Compose modernes.
    mem_limit: 512m
    cpus: 1.0
    pids_limit: 200
```

Observer `docker stats`. Les limites doivent être basées sur des mesures : charge normale, pic, temps de réponse et comportement du garbage collector. Kubernetes possède ses propres `requests`/`limits` ; ne pas supposer que les réglages Compose se transposent automatiquement.

---

## 12. API FastAPI de prédiction : mini-service MLOps

### Arborescence

```text
ml-api/
├── Dockerfile
├── requirements.txt
├── app/main.py
├── app/model.py
└── tests/test_api.py
```

`requirements.txt` :

```text
fastapi==0.115.6
uvicorn[standard]==0.34.0
scikit-learn==1.6.0
joblib==1.4.2
pytest==8.3.4
httpx==0.28.1
```

`app/model.py` — modèle déterministe jouet, à remplacer par un artefact entraîné et versionné :

```python
from pathlib import Path
import joblib
from sklearn.linear_model import LogisticRegression

MODEL_PATH = Path("/app/model.joblib")

def load_or_train():
    if MODEL_PATH.exists():
        return joblib.load(MODEL_PATH)
    x = [[0.0], [1.0], [2.0], [3.0]]
    y = [0, 0, 1, 1]
    model = LogisticRegression().fit(x, y)
    MODEL_PATH.parent.mkdir(parents=True, exist_ok=True)
    joblib.dump(model, MODEL_PATH)
    return model
```

`app/main.py` :

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field
from .model import load_or_train

app = FastAPI(title="prediction-api", version="1.0.0")
model = load_or_train()

class Request(BaseModel):
    feature: float = Field(..., ge=-100, le=100)

@app.get("/health")
def health():
    return {"status": "ok", "model_loaded": model is not None}

@app.post("/predict")
def predict(request: Request):
    prediction = int(model.predict([[request.feature]])[0])
    probability = float(model.predict_proba([[request.feature]])[0][prediction])
    return {"prediction": prediction, "probability": probability, "model_version": "demo-1"}
```

`tests/test_api.py` :

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["model_loaded"] is True

def test_predict_contract():
    response = client.post("/predict", json={"feature": 2.5})
    assert response.status_code == 200
    assert 0 <= response.json()["probability"] <= 1
```

`Dockerfile` :

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
COPY tests ./tests
RUN python -c "from app.model import load_or_train; load_or_train()" && chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3)" || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
cd ml-api
python -m pytest tests
# Build et test manuel
docker build -t prediction-api:1.0 .
docker run --rm --name prediction-api -p 8000:8000 prediction-api:1.0
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict -H 'Content-Type: application/json' -d '{"feature":2.5}'
```

Pour un vrai service, ne pas entraîner au démarrage : construire ou récupérer un artefact validé, contrôler sa signature/version, effectuer des tests de compatibilité et gérer les erreurs de chargement. Le modèle ne doit pas provenir d’une URL non vérifiée ; la désérialisation de formats Python peut exécuter du code si l’artefact est hostile.

### Compose du service ML

```yaml
services:
  prediction-api:
    build: .
    ports:
      - "8001:8000"
    environment:
      MODEL_VERSION: demo-1
    read_only: true
    tmpfs:
      - /tmp
    cap_drop: ["ALL"]
    security_opt:
      - no-new-privileges:true
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 20s
      timeout: 5s
      retries: 3
```

Le port hôte `8001` évite le conflit avec l’exemple Compose data qui utilise `8000`; le port interne reste `8000`.

### Observabilité de base

Commencer par : logs JSON sur stdout avec `request_id`, latence, code HTTP et version du modèle ; endpoints `/health` et éventuellement `/ready` ; métriques de requêtes, erreurs et latence ; traces distribuées si le système le justifie. Les logs peuvent contenir des données personnelles : minimiser, masquer et définir une rétention.

Un simple format de log en Python :

```python
import json, logging, time, uuid
log = logging.getLogger("prediction-api")

def audit_log(event, **fields):
    log.info(json.dumps({"event": event, "request_id": str(uuid.uuid4()), **fields}))
```

Pour un vrai déploiement, brancher OpenTelemetry/Prometheus selon la plateforme et vérifier les SLO. Docker fournit surtout les primitives de processus, réseau et logs ; il ne constitue pas à lui seul une plateforme d’observabilité.

---

## 13. Publication dans Google Artifact Registry (GCP)

Pré-requis : projet GCP, API Artifact Registry activée, Docker installé et droits IAM adaptés. Remplacer les valeurs d’exemple ; elles ne sont pas des secrets.

```bash
export PROJECT_ID="mon-projet-gcp"
export REGION="europe-west1"
export REPOSITORY="containers"
export IMAGE="prediction-api"
export TAG="1.0.0"

gcloud config set project "$PROJECT_ID"
gcloud services enable artifactregistry.googleapis.com
gcloud artifacts repositories create "$REPOSITORY" \
  --repository-format=docker \
  --location="$REGION" \
  --description="Images Docker data et ML"

gcloud auth configure-docker "${REGION}-docker.pkg.dev"

docker build -t "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${IMAGE}:${TAG}" .
docker push "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${IMAGE}:${TAG}"
gcloud artifacts docker images list \
  "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}"
```

Le nom complet suit :

```text
LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG
```

En CI, préférer une identité de workload fédérée (Workload Identity Federation) à une clé JSON longue durée. Accorder le moindre privilège, séparer les projets et repositories, activer la journalisation et les politiques de rétention. Ne pas mettre de clé de service dans une image, un commit ou une commande enregistrée. Selon l’environnement, Cloud Build, GitHub Actions avec fédération, Cloud Run ou GKE peuvent consommer l’image ; le choix d’exécution est distinct du registre.

### CI/CD conceptuel

1. Pull request : lint, tests unitaires, tests de contrat API et scan de dépendances.
2. Build : construire l’image, produire un tag lié au commit (`git rev-parse --short HEAD`), scanner l’image.
3. Publication : pousser vers Artifact Registry uniquement depuis une branche protégée.
4. Déploiement : promouvoir la même image immuable vers staging puis production ; ne pas reconstruire un artefact différent entre environnements.
5. Post-déploiement : smoke test `/health`, métriques, rollback vers le tag précédent.

Docker ne fournit pas ces contrôles à lui seul. Les permissions IAM, l’approbation, la gestion des environnements et la cible (Cloud Run, GKE, VM…) doivent être conçues séparément. Terraform peut déclarer le repository, les comptes de service et les services GCP ; Dockerfile/Compose décrivent le packaging et le développement local, pas toute l’infrastructure.

---

## 14. Exercices corrigés

### Exercice 1 — port et processus

**Énoncé :** lancer `nginx:alpine` sur le port hôte `8088`, vérifier puis arrêter.

**Correction :**

```bash
docker run -d --name ex-nginx -p 8088:80 nginx:alpine
curl -I http://localhost:8088
docker logs ex-nginx
docker rm -f ex-nginx
```

### Exercice 2 — volume PostgreSQL

**Énoncé :** créer une table, supprimer le conteneur, recréer le conteneur avec le même volume et vérifier la table.

**Correction :** utiliser `docker volume create ex-pgdata`, monter `/var/lib/postgresql/data`, puis `docker exec … psql`. La donnée persiste grâce au volume, pas grâce au conteneur.

### Exercice 3 — service non joignable

**Énoncé :** une API utilise `DB_HOST=localhost` dans Compose.

**Correction :** remplacer par le nom de service, par exemple `DB_HOST=db`. `localhost` désigne le conteneur API lui-même. Vérifier avec `docker compose exec api getent hosts db` si l’outil existe, puis `pg_isready -h db` depuis une image PostgreSQL.

### Exercice 4 — image trop grosse et fuite de secret

**Énoncé :** le Dockerfile contient `COPY . .`, installe sans `--no-cache-dir` et copie `.env`.

**Correction :** ajouter `.dockerignore`, copier d’abord le fichier de dépendances, utiliser une image slim, `pip install --no-cache-dir`, et injecter le secret à l’exécution via une solution dédiée. Scanner puis reconstruire depuis un contexte nettoyé.

### Exercice 5 — healthcheck

**Énoncé :** écrire un contrôle qui distingue un processus vivant d’une API prête.

**Correction :** ajouter `/health` et faire vérifier cette URL par `HEALTHCHECK` ou Compose. Si l’API dépend d’une base, `/ready` peut vérifier la connexion ; garder `/live` simple pour distinguer panne de processus et dépendance indisponible.

---

## 15. Mini-projets progressifs

### Mini-projet A — pipeline batch

Créer une image non-root qui lit `data/input.csv` via un bind mount, nettoie les lignes invalides et écrit `data/output.csv`. Ajouter un test local, un code de sortie non nul en cas d’entrée absente, puis un volume ou un stockage objet pour un environnement partagé. Critères : build reproductible, image sans secret, logs de nombre de lignes, test de reprise.

### Mini-projet B — stack data locale

Étendre le Compose PostgreSQL avec un worker Python qui consomme des événements et écrit une table agrégée. Ajouter un healthcheck DB, une migration idempotente, des limites CPU/mémoire et un réseau séparé pour ne publier que l’API. Critères : `docker compose up`, test d’intégration, `down -v` documenté et sauvegarde de test.

### Mini-projet C — prediction promotionnée

Construire l’API ML, versionner un artefact de modèle, publier l’image taguée par commit dans Artifact Registry et exécuter un smoke test depuis une image propre. Ajouter métriques de latence et taux d’erreur, puis décrire une promotion staging→production et un rollback. Critères : tests, scan, utilisateur non-root, healthcheck, absence de clé GCP dans le dépôt.

---

## 16. Aide-mémoire

```bash
# Cycle image
docker build -t name:tag .
docker image inspect name:tag
docker run --rm -p HOST:CONTAINER name:tag

# Cycle conteneur
docker ps -a
docker logs -f CONTAINER
docker exec -it CONTAINER sh
docker inspect CONTAINER
docker stop CONTAINER && docker rm CONTAINER

# Compose
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs -f SERVICE
docker compose exec SERVICE sh
docker compose down

# Réseau et stockage
docker network ls
docker volume ls
docker system df

# Sécurité/diagnostic
docker stats
docker history IMAGE
docker events --since 10m

# Artifact Registry GCP
 gcloud auth configure-docker REGION-docker.pkg.dev
docker tag IMAGE:TAG REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG
docker push REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG
```

### Checklist avant partage d’une image

- [ ] Version de base et dépendances maîtrisées.
- [ ] `.dockerignore` présent ; aucun secret dans le contexte.
- [ ] Processus au premier plan, forme JSON de `CMD`/`ENTRYPOINT`.
- [ ] Utilisateur non root, capacités inutiles supprimées.
- [ ] Healthcheck pertinent et logs sur stdout/stderr.
- [ ] Limites de ressources testées.
- [ ] Tests et scan exécutés ; image identifiée par un tag immuable ou digest.
- [ ] Volumes, migrations, sauvegardes et restauration documentés.
- [ ] Registre, IAM GCP et CI/CD séparés des secrets.
- [ ] Limites de Docker local clairement distinguées de celles de Kubernetes/Cloud Run/GKE.

---

## Conclusion

Commencer par comprendre le cycle **Dockerfile → image → conteneur**, puis maîtriser ports, réseaux, volumes, logs et Compose. Pour évoluer vers MLOps, ajouter la reproductibilité des dépendances, les tests, le scan, les artefacts de modèles, les healthchecks, les métriques et une promotion CI/CD vers Artifact Registry. Enfin, choisir explicitement la plateforme d’exécution et l’outil d’infrastructure : Docker facilite le packaging, mais la fiabilité d’un système data/ML dépend aussi de l’orchestration, du cloud, de la sécurité, de la donnée et des opérations.
