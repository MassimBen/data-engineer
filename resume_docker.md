# Commande docker 

[Docker documentation officiel](https://docs.docker.com/reference)
## Base command docker 
```
## Afficher de l'aide
docker help
docker <sous-commande> --help

## Afficher des informations sur l'installation de Docker
docker --version
docker version
docker info

## Executer une image Docker
docker run hello-world

## Lister des images Docker
docker image ls
# ou
docker images

## Supprimer une image Docker
docker images rmi <IMAGE_ID ou IMAGE_NAME>  # si c'est le nom de l'image qui est spécifié alors il prendra par défaut le tag latest
    -f ou --force : forcer la suppression

## Supprimer tous les images Docker
docker rmi -f $(docker images -q)

## Rechercher une image depuis le Docker hub Registry
docker search ubuntu
    --filter "is-official=true" : Afficher que les images officielles

## Télécharger une image depuis le Docker hub Registry
docker pull <IMAGE_NAME>  # prendra par défaut le tag latest
docker pull ubuntu:16.04 # prendra le tag 16.04
```

## Other docker command
```
## Exécuter une image Docker
docker run <CONTAINER_ID ou CONTAINER_NAME>
    -t ou --tty : Allouer un pseudo TTY
    --interactive ou -i : Garder un STDIN ouvert
    --detach ou -d : Exécuter le conteneur en arrière-plan
    --name : Attribuer un nom au conteneur
    --expose: Exposer un port ou une plage de ports
    -p ou --publish : Mapper un port  "<PORT_CIBLE:PORT_SOURCE>"
    --rm : Supprimer automatiquement le conteneur quand on le quitte

## Lister des conteneurs en état running Docker
docker container ls
# ou
docker ps
    -a ou --all : Afficher tous les conteneurs peut-importe leur état

## Supprimer un conteneur Docker
docker rm <CONTAINER_ID ou CONTAINER_NAME>
    -f ou --force : forcer la suppression

## Supprimer tous les conteneurs Docker
docker rm -f $(docker ps -aq)

## Exécuter une commande dans un conteneur Docker
docker exec <CONTAINER_ID ou CONTAINER_NAME> <COMMAND_NAME>
    -t ou --tty : Allouer un pseudo TTY
    -i ou --interactive : Garder un STDIN ouvert
    -d ou --detach : lancer la commande en arrière plan

## sorties/erreurs d'un conteneur
docker logs <CONTAINER_ID ou CONTAINER_NAME>
    -f : suivre en permanence les logs du conteneur
    -t : afficher la date et l'heure de la réception de la ligne de log
    --tail <NOMBRE DE LIGNE> = nombre de lignes à afficher à partir de la fin (par défaut "all")
## Transformer un conteneur en image
docker commit <CONTAINER_NAME ou CONTAINER_ID> <NEW IMAGENAME>
    -a ou --author <string> : Nom de l'auteur (ex "John Hannibal Smith <hannibal@a-team.com>")
    -m ou --message <string> : Message du commit
```
## Build image

Create a dockerfile for example and use docker build to create a image

```
docker build -t mass/docker-learn/image_name:version
```
-t : it's tag you give to docker image otherwise it's a name

mass : it's username

docker-learn : it's the repository to put the image

image_name : name of image you create

version : the version of image


After build image you can push to your docker hub

```
docker push mass/docker-learn/image_name:version
```
Container is isolate envirenement can run specique task you can run something like frontend of you page. You use multiple containers to run multiple think in web or something else. it's like VM but don't need space in your machine or something, anyone can run it.
Image it's like a package you create and you can use it any where and everyone can use when it's create and put un register docker hub or you can send it.

Container it's like a receip you use to put thing in a see what happen, in thise case container contain image and run it to produce something.

## explanation of docker compose model

Computing components of an application are defined as services. A service is an abstract concept implemented on platforms by running the same container image, and configuration, one or more times.

Services communicate with each other through networks. In the Compose Specification, a network is a platform capability abstraction to establish an IP route between containers within services connected together.

Services store and share persistent data into volumes. The Specification describes such a persistent data as a high-level filesystem mount with global options.

Some services require configuration data that is dependent on the runtime or platform. For this, the Specification defines a dedicated configs concept. From inside the container, configs behave like volumes—they’re mounted as files. However, configs are defined differently at the platform level.

A secret is a specific flavor of configuration data for sensitive data that should not be exposed without security considerations. Secrets are made available to services as files mounted into their containers, but the platform-specific resources to provide sensitive data are specific enough to deserve a distinct concept and definition within the Compose Specification.

Note
With volumes, configs and secrets you can have a simple declaration at the top-level and then add more platform-specific information at the service level.

A project is an individual deployment of an application specification on a platform. A project's name, set with the top-level name attribute, is used to group resources together and isolate them from other applications or other installation of the same Compose-specified application with distinct parameters. If you are creating resources on a platform, you must prefix resource names by project and set the label com.docker.compose.project.

Compose offers a way for you to set a custom project name and override this name, so that the same compose.yaml file can be deployed twice on the same infrastructure, without changes, by just passing a distinct name.

The Compose file
The default path for a Compose file is compose.yaml (preferred) or compose.yml that is placed in the working directory. Compose also supports docker-compose.yaml and docker-compose.yml for backwards compatibility of earlier versions. If both files exist, Compose prefers the canonical compose.yaml.

You can use fragments and extensions to keep your Compose file efficient and easy to maintain.

Multiple Compose files can be merged together to define the application model. The combination of YAML files is implemented by appending or overriding YAML elements based on the Compose file order you set. Simple attributes and maps get overridden by the highest order Compose file, lists get merged by appending. Relative paths are resolved based on the first Compose file's parent folder, whenever complementary files being merged are hosted in other folders. As some Compose file elements can both be expressed as single strings or complex objects, merges apply to the expanded form. For more information, see Working with multiple Compose files.

If you want to reuse other Compose files, or factor out parts of your application model into separate Compose files, you can also use include. This is useful if your Compose application is dependent on another application which is managed by a different team, or needs to be shared with others.

