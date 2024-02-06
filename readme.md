# TP1

## Part 01

### Database

1-1 Document your database container essentials: commands and Dockerfile.

``` Dockerfile
#l'image de base
FROM postgres:14.1-alpine

#on déclarre des varibales d'environement
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

#On copy les fichier présent dans entrypoint vers le dossier /docker-entrypoint-initdb.d du contener
COPY entrypoint/* /docker-entrypoint-initdb.d
```
Pour build mon image postgress :
``` sh
sudo docker build -t max/post . 
```
Pour démarrer cette image:
``` sh
sudo docker run -d --network app-network -v /tmp/data:/var/lib/postgresql/data --name post max/post
```
Pour démarrer adminer:
``` sh
sudo docker run -d --network app-network --name adminer -p 8080:8080 adminer
```

Pour ce connecter à adminer à l'adress http://localhost:8080
```
System	: PostgreSQL
Server	: 172.18.0.2  //trouver grace à docker inspect post changer ensuite par le nom du container
Username : usr
Password : pwd 
Database : db
```

### BackendAPI

``` Dockerfile
FROM openjdk:11

WORKDIR /app
COPY Main.class /app

CMD ["java", "Main"]
```
```
javac Main.java
sudo docker build -t java .
sudo docker run -it --rm --name java java
```



1-2 Why do we need a multistage build? And explain each step of this dockerfile.

il est intérresant d'avoir un multistep car cela permet d'éviter d'embarqué le compilateur (JDK) dans notre container de run qui n'a besoin que de JRE
```
sudo docker build -t simpleapi .

sudo docker run -d -p 8090:8080 --rm --name simpleapi simpleapi


sudo docker build -t back .

sudo docker run -d -p 8091:8080 --rm --network app-network --name back back
```

### Front
``` Dockerfile
FROM httpd:2.4
COPY ./html/ /usr/local/apache2/htdocs/
COPY ./https.conf /usr/local/apache2/conf/httpd.conf
```
```
docker build -t http .
docker run -dit --network app-network --name http -p 8080:80 http
```


```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://back:8080/
ProxyPassReverse / http://back:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
```
docker cp ./httpd.conf http:/usr/local/apache2/conf/httpd.conf 

docker restart http
```

Why do we need a reverse proxy?

un reverse proxy est utile afin de centralisé les accès à différent serveur web
Cela permet donc d'exposer à internet selement les services autorisé


Why is docker-compose so important?

Docker compose permet de gerer plusieurs container avec une seule commande avec toute leurs configuration nécéssaire

1-3 Document docker-compose most important commands.
```sh
docker-compose up -d --force-recreate // démarre la stack
docker-compose down -v // arrete la stack et supprimer tous les volumes et les networks
```
 1-4 Document your docker-compose file.

 ```yml
version: '3.8'  #version de docker compose à utiliser

services:
    backend:  #pour le container backend
        container_name: back # nom du container
        build: BackendAPI/simple-api-student  #chemin du dockerfile à build
        networks: #list des networks à utiliser
          - my-network
        depends_on: #list des container nécéssaire avant de démarrer
          - database
        env_file: #on définit le fichier des variables d'environements
          - database.env

    database:
        container_name: database
        build: database
        networks:
          - my-network
        env_file:
          - database.env
        volumes: # on ajoute le volume pour la persistante 
          - /tmp/data:/var/lib/postgresql/data 

    httpd:
        container_name: front
        build: Front
        ports:  # on expose le port 80 vers l'hote
          - 80:80
        networks:
          - my-network
        depends_on:
          - database
          - backend

networks:
    my-network: #on déclare le network utilisé
```

docker-compose.override.yml
```yml
version: '3.8'

services:
    adminer:
      container_name: adminer
      image: adminer
      ports:
        - 8093:8080
      networks:
          - my-network
```

1-5 Document your publication commands and published images in dockerhub.

```
docker login # pour ce connecter a docker hub

docker tag part_01-database maximedubois1cpe/madatabase:1.0 # permet de versionner mon image et la préparer à l'envoie

docker push maximedubois1cpe/madatabase:1.0 # on push l'image sur docker hub
```

Why do we put our images into an online repo?

Pour que ces images soit accessible n'importe ou dans le monde


# TP2

2-1 What are testcontainers?
Ce sont des bibliothèque java qui permet l'exécution des tests dans des docker

```yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify
```
