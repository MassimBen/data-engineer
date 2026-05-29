

# Docker and Postegresql note


**Introduction to docker** : basic command and way to use it (containerization the code), create image and use container for code or to try a code without install dependancy in local system
 Below there are some example of command line of docker
 
```
# see version of docker
docker --version
# exemple of running image of python
docker run -it python:3.13
#example of running image with entrypoint bash
docker run -it entrypoint=bash
 # list of container
docker ps
# delete of container
docker rm id_container or use docker rm $(docker ps -aq)


```

**Discover uv ( virtual environnement)** : it's able as to run a code in virtual environement, use it with docker to containerize and a pipeline of ingestion dans test a code without use local system ressource.

```
pip install uv
```

**Create dockerfile**: it's usefull to write and use docker build to create a containers start directly with the file and do all thing write.
example
```
# base Docker image that we will build on
FROM python:3.13.11-slim

# set up our image by installing prerequisites; pandas in this case
RUN pip install pandas pyarrow

# set up the working directory inside the container
WORKDIR /app
# copy the script to the container. 1st name is source file, 2nd is destination
COPY pipeline.py pipeline.py

# define what to do first when the container runs
# in this example, we will just run the script
ENTRYPOINT ["python", "pipeline.py"]

```
Other example of docker file with more instruction : docker with uv

```
# Start with slim Python 3.13 image
FROM python:3.13.10-slim

# Copy uv binary from official uv image (multi-stage build pattern)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

# Set working directory
WORKDIR /app

# Add virtual environment to PATH so we can use installed packages
ENV PATH="/app/.venv/bin:$PATH"

# Copy dependency files first (better layer caching)
COPY "pyproject.toml" "uv.lock" ".python-version" ./
# Install dependencies from lock file (ensures reproducible builds)
RUN uv sync --locked

# Copy application code
COPY pipeline.py pipeline.py

# Set entry point
ENTRYPOINT ["uv", "run", "python", "pipeline.py"]
```
**Postregre to docker**: create container for postgre in docker with environnement and parameters( best to do create a folder and go in create le docker inside), install pgcli with uv and connect to postgre in docker to write sql query et create database.

**data ingestion** : In this step use jupiter notbook (uv add --dev jupyter notebook) to load the data and prepare for ingestion in postegre, create script of ingestion use sqlalchemy for ingestion data in postegre, the data is so big use chunck to partition the ingestion.  After connect to postegre to see porgression of ingestion

**ingestion script** : use clic to better ingestion with parameter in engine et define default paramater can use in terminal or modify in terminal for secur and goog pratice

**pgadmin** : use pgdamin interface web because it's usefull for complex query than pgcli in the terminal, for run that need to connect to pgdatabase in the same network in docker, to do that, we will create docker network and define in the same docker network. After use the link to go to web site of pgdmin enter identifiant and password, create server and explore the databse. When you link pgdatabase with pgdamin the ingestion in pgdatabase goes in pgadmin directly.

**dockerinzing ingestion and docker compose** : After convert a scripty to .py you dockrize (use docker file) the script with all element need, build it and run it with parameter need for postgre define un the script (with click), define the network where the pgdatabase and pgdmin are to do the ingestion and use the interface web of pgadmin to query the data ingest. To define u=in the same network in unique way use docker-compose file to store all element for both and excute it after that run the docker for ingestion.

```
services:
  pgdatabase:
    image: postgres:18
    environment:
      POSTGRES_USER: "root"
      POSTGRES_PASSWORD: "root"
      POSTGRES_DB: "ny_taxi"
    volumes:
      - "ny_taxi_postgres_data:/var/lib/postgresql"
    ports:
      - "5432:5432"

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: "admin@admin.com"
      PGADMIN_DEFAULT_PASSWORD: "root"
    volumes:
      - "pgadmin_data:/var/lib/pgadmin"
    ports:
      - "8085:80"



volumes:
  ny_taxi_postgres_data:
  pgadmin_data:
```

