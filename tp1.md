# DEVOPS - TP 01 - DOCKER
## DATABASE

### Basics

Création d'une image basique postgre à partir du dockerfile suivant :
```
FROM  postgres:11.6-alpine  
ENV  POSTGRES_DB=db \  
POSTGRES_USER=usr \  
POSTGRES_PASSWORD=pwd
```
Pour build, j'ai placé le fichier Dockerfile dans un dossier *postgres* et j'ai lancé le build avec 
```
docker build . -t eloi/postgres
```

J'ai oublié de donner un nom à mon image, du coup j'ai utilisé
```
docker images 
```

Et je l'ai renommé à partir de son ID en faisant :
```
docker tag 7a110d9e8a8e eloi/postgres:latest
```

Pour démarrer un container utilisant cette image, j'ai utilisé
```
docker run -p 127.0.0.1:5432:5432/tcp --name postgres eloi/postgres
```
Afin de tester le container postgres j'ai utilisé adminer comme ceci :
```
docker run -p 8080:8080 adminer
```

Pour relier par un réseau adminer et postgres j'ai créé un Network et j'ai relancé les docker en les reliant à ce réseau :
```
docker network create app-network
docker run -p 127.0.0.1:5432:5432/tcp --network app-network --name postgres eloi/postgres
docker run -p 8080:8080 --network app-network --name adminer adminer
```

Docker réalise via les network de la résolution d'adresse, de ce fait dans Adminer, dans la partie host, pour accéder au container "postgres" depuis adminer j'ai utilisé le "postgres" qui est le nom du container.

### Init database

Pour initialiser la database j'ai utilisé des scripts SQL que j'ai placé à l'emplacement de mon fichier Dockerfile et j'ai créé le dossier *docker-entrypoint-initdb.d*. Suite à cela j'ai rajouté les lignes suivantes dans le Dockerfile :
```
COPY CreateScheme.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/
```

### Persist Data

Pour éviter que les données soient supprimées à chaque fois que l'on supprime le container, on ajoute un stockage interne à la machine hôte au container via le flag -v. 
```
docker run -p 127.0.0.1:5432:5432/tcp --network app-network -v /data:/var/lib/postgresql/data --name postgres eloi/postgres
```

## Backend API

### Basics

Voici le Dockerfile permettant d'éxecuter un fichier Java sous un OpenJDK :
```
FROM	openjdk:16-alpine3.13
ADD Main.java Main.java
RUN javac Main.java
CMD java Main
```

Pour l'éxecuter on utilise donc :
```
docker run --name backend eloi/backend
```

### Multistage build

Pour le multistage build, j'ai extrait le dossier généré via spring et j'ai placé à sa racine (au même niveau que SRC) le Dockerfile suivant :
```
# Build 
FROM maven:3.6.3-jdk-11 AS myapp-build 
ENV MYAPP_HOME /opt/myapp 
WORKDIR $MYAPP_HOME 
COPY pom.xml . 
COPY src ./src 
RUN mvn package -DskipTests 

# Run 
FROM openjdk:11-jre 
ENV MYAPP_HOME /opt/myapp 
WORKDIR $MYAPP_HOME 
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar 

ENTRYPOINT java -jar myapp.jar
```

Le multi-stage build permet d'utiliser plusieurs FROM dans le Dockerfile et de n'utiliser que ce qui est nécessaire au fonctionnement du projet et ainsi éviter d'englober toutes les dépendances dans son Container.

La ligne :
```
COPY --from=myapp-build $MYAPP_HOME/target/*.jar
```
permet de récupérer les fichier générés lors du build afin de réaliser le Run. avec l'open JDK en version 11.

### Backend API
Tout d'abord, au lancement du container, j'ai utilisé la commande suivante pour utiliser le network :
```
docker run --name backend-api -p 8081:8080 --network app-network eloi/backend-api-2
```

Pour relier l'application à la base de données, au sein de application.yml j'ai configuré comme ceci la partie liée à la BDD :
```
  datasource:
    url: jdbc:postgresql://postgres:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver

```

Avec postgresql correspondant au driver et postgres correspondant au nom du container, mapping réalisé par le network.

## HTTP Server

### Basics
Pour créer un serveur Web Apache écoutant sur le port 80 j'ai utilisé le dockerfile suivant :
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
```

Je l'ai build et exécuté comme ceci :
```
docker build . -t eloi/http_basic
docker run --name http_basic -p 80:80 eloi/http_basic
```

Pour retrouver les logs du serveur j'utilise :
```
docker logs http_basic
```

### Configuration

Tout d'abord j'ai récupéré le fichier de config HTTPD du container via docker cp
```
docker cp http_basic:/usr/local/apache2/conf/httpd.conf .
```

Je l'ai ajouté au dockerfile pour écraser l'ancien lors du build :
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/
```

On active le mod_proxy_http dans le fichier httpd.conf, on ajoute les lignes suivantes :

```
ServerName localhost
<VirtualHost *:80>
        ProxyPreserveHost On
        ProxyPass / http://backend-api:8080/
        ProxyPassReverse / http://backend-api:8080/
</VirtualHost>
```

J'ai lancé le container en le reliant au réseau app-network
```
docker run --name http_basic -p 80:80 --network app-network eloi/http_basic
```


## Link application

### Docker compose

Voici le fichier docker-compose.yaml :
```
version: '3.7'

services:
  backend-api:
    build:
      ./backend-api-2/simple-api-main/simple-api/ #chemin vers dockerfile
    networks:
      - my-network
    depends_on:
      - postgres  # demarre après postgres
  postgres:
    build:
      ./postgres/
    networks:
      - my-network
    volumes:
      - db-volume:/var/lib/postgresql/data

  httpd:
    build:
      ./http_basic/ #chemin vers dockerfile
    ports:
      - 80:80
    networks:
      - my-network
    depends_on:
      - backend-api # demarre après backend-api
networks:
  my-network:
volumes:
  db-volume: {}
```

J'ai placé le fichier docker-compose.yml à la racine du TP (racine contenant tous les dossiers contenant les Dockerfiles des 3 images).

Pour le lancer j'ai utilisé :
```
docker-compose up -d #start
---------------
docker-compose restart backend-api # restart d'un seul service
---------------
docker-compose stop #stop
---------------
docker-compose rm #remove
```

## Publish

Pour le publish j'ai fait les commandes dans cet ordre
```
docker login
docker tag eloi/postgres toufub/postgres:1.0 #pour lui donner un nom
docker push toufub/postgres:1.0 # pour le push
```
