# Compte Rendu TP1 Docker

## Création Docker Base de données

## Docker PostgreSQL

```
FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
COPY ./docker-entrypoint-initdb.d /docker-entrypoint-initdb.d
```

## Commandes exécutées

```
docker build -t gregoryspn/mydatabase .
```

```
docker network create app-network
```

```
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

```
docker run -p 5432:5432 --name databaseTP1 -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd --network app-network -v /my/own/datadir:/var/lib/postgresql/data gregoryspn/mydatabase
```
- -p : port de notre conteneur
- -e : variables d'environnement 
-  -network : réseau partagé
- -v : volume utilisé pour la persistance des données 

*Why should we run the container with a flag `-e` to give the environment variables ?*

Pour garantir une meilleure sécurité, il est recommandé d'exécuter le conteneur avec l'option -e afin de fournir les variables d'environnement nécessaires plutôt que d'inclure les logins utilisés directement dans le Dockerfile.

*Why do we need a volume to be attached to our postgres container ?*

On utilise un volume pour la persistance des données dans le conteneur.

## Backend

```
FROM openjdk
COPY Main.java .
RUN javac Main.java
CMD ["java", "Main"]
```

```
docker build -t gregoryspn/mybackendapi .
docker run --name backendTP1 gregoryspn/mybackendapi
```

## Backend Simple

### Dockerfile

*Why do we need a multistage build? And explain each step of this dockerfile*.

```
# Build
## On récupère l'image de base qui contient le JDK
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
## On définit la varible d'environnement MYAPP_HOME
ENV MYAPP_HOME /opt/myapp
## On se déplace dans le répertoire /opt/myapp
WORKDIR $MYAPP_HOME
## On copie le pom.xml dans le répertoire courant
COPY ./simpleapi/simpleapi/pom.xml .
##On copie le répertoire src de notre machine vers notre conteneur
COPY ./simpleapi/simpleapi/src ./src
## On lance la compilation de notre Backend
RUN mvn package -DskipTests

# Run
## On récupère l'image qui contient le jre
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
## On copie a l'aide de "from" le code compilé du premier conteneur au deuxième
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
## On execute la commande java -jar myapp.jar pour lancer notre backend
ENTRYPOINT java -jar myapp.jar
```

On utilise un build multistage pour réduire la taille de l'image puisqu'on copie uniquement les fichiers nécessaires dans la dernière étape. Cela facilite aussi la gestion et permet de mieux comprendre les étapes du fichier Docker.

On modifie le ficher application.yml de notre API pour configurer l'accès à notre base de données.
```
datasource:  
  url: jdbc:postgresql://databaseTP1:5432/db  
  username: usr  
  password: pwd
```

On lance ensuite notre conteneur.
```
docker run --name backendsimple --network app-network -p 8080:8080 gregoryspn/backsimpleapi
```

## HTTP Server

### Docker file 

```
FROM httpd:2.4.58-alpine
COPY ./public-html/ /usr/local/apache2/htdocs/
```

```
docker build -t gregoryspn/httpservertp1 .
```

```
docker run --name httpserverTP1 -p 8081:80 gregoryspn/httpservertp1
```

On utilise docker exec pour afficher la configuration de notre reverse Proxy.
```
docker exec -it httpserverTP1 cat /usr/local/apache2/conf/httpd.conf
```

## Reverse Proxy
```
ServerName localhost
"<VirtualHost *:80>"\n\
"ProxyPreserveHost On"\n\
"ProxyPass / http://backendsimple:80/"\n\
"ProxyPassReverse / http://backendsimple:80/"\n\
"</VirtualHost>"\n\
"LoadModule proxy_module modules/mod_proxy.so"\n\
"LoadModule proxy_http_module modules/mod_proxy_http.so"\n\
```

Avec Apache on redirige l'appel à notre backend vers le port 80.

*Why do we need a reverse proxy ?*

On utilise un reverse proxy pour la sécurité, c'est une couche intermédiaire entre les utilisateurs et les serveurs d'origine.

## Docker Compose

```
version: '3.8'

services:
  database:
    build: ./Database
    ports:
      - "5432:5432"
    volumes:
      - ./Database/data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    networks:
      - app-network

  backend:
    build: ./Backend API/simpleapi/simpleapi/
    ports:
      - "8080:8080"
    networks:
      - app-network
    environment:
      - DB_URL=jdbc:postgresql://database:5432/db
      - DB_USER=usr
      - DB_PASSWORD=pwd
    depends_on:
      - database

  httpd:
    build: ./HTTP Server
    ports:
      - 80:80
    networks:
      - app-network
    depends_on:
      - backend
     
networks:
  app-network:
```

Le docker compose nous permet de lancer en une fois tout nos conteneurs : base de données, backend, proxy. Pour communiquer ensemble, ils sont tous sur le même réseau : app-network.
- -build : indique l'emplacement du Dockerfile  
- -port : ports utilisés par notre machine et le conteneur
- -environnement : définition des variables d'environnement
- -depends_on : on attend que le conteneur visé se finisse
## Publish 

On publie sur notre compte Docker Hub nos 3 images, cela permet de faire du versioning. 
```
docker tag tp1-database gregoryspn/tp1-database:1.0
docker push gregoryspn/tp1-database:1.0

docker tag tp1-backend gregoryspn/tp1-backend:1.0
docker push gregoryspn/tp1-backend:1.0

docker tag tp1-httpd gregoryspn/tp1-httpd:1.0
docker push gregoryspn/tp1-httpd:1.0
```

Les images sont désormais push et peuvent être utilisé par n'importe qui.
