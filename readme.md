### TP1

## Part 01

# Database

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
System	: PostgreSQL
Server	: 172.18.0.2  //trouver grace à docker inspect post
Username : usr
Password : pwd 
Database : db


# BackendAPI

``` Dockerfile
FROM openjdk:11

WORKDIR /app
COPY Main.class /app

CMD ["java", "Main"]
```
javac Main.java
sudo docker build -t java .
sudo docker run -it --rm --name java java



1-2 Why do we need a multistage build? And explain each step of this dockerfile.

il est intérresant d'avoir un multistep car cela permet d'éviter d'embarqué le compilateur (JDK) dans notre container de run qui n'a besoin que de JRE

sudo docker build -t simpleapi .

sudo docker run -d -p 8090:8080 --rm --name simpleapi simpleapi


sudo docker build -t back .

sudo docker run -d -p 8091:8080 --rm --network app-network --name back back