# data-engineer
### Data engineer zoomcamp
1. Module

**Introduction to docker** : basic command and way to use it (containerization the code), create image and use container for code or to try a code without install dependancy in local system

**Discover uv ( virtual environnement)** : it's able as to run a code in virtual environement, use it with docker to containerize and a pipeline of ingestion dans test a code without use local system ressource.

**Create dockerfile**: it's usefull to write and use docker build to create a containers start directly with the file and do all thing write.

**Postregre to docker**: create container for postgre in docker with environnement and parameters( best to do create a folder and go in create le docker inside), install pgcli with uv and connect to postgre in docker to write sql query et create database.

**data ingestion** : In this step use jupiter notbook (uv add --dev jupyter notebook) to load the data and prepare for ingestion in postegre, create script of ingestion use sqlalchemy for ingestion data in postegre, the data is so big use chunck to partition the ingestion.  After connect to postegre to see porgression of ingestion

**ingestion script** : use clic to better ingestion with parameter in engine et define default paramater can use in terminal or modify in terminal for secur and goog pratice

**pgadmin** : use pgdamin interface web because it's usefull for complex query than pgcli in the terminal, for run that need to connect to pgdatabase in the same network in docker, to do that, we will create docker network and define in the same docker network. After use the link to go to web site of pgdmin enter identifiant and password, create server and explore the databse. When you link pgdatabase with pgdamin the ingestion in pgdatabase goes in pgadmin directly.

**dockerinzing ingestion and docker compose** : After convert a scripty to .py you dockrize (use docker file) the script with all element need, build it and run it with parameter need for postgre define un the script (with click), define the network where the pgdatabase and pgdmin are to do the ingestion and use the interface web of pgadmin to query the data ingest. To define u=in the same network in unique way use docker-compose file to store all element for both and excute it after that run the docker for ingestion.


