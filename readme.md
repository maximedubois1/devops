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

2-2 Document your Github Actions configurations.
```yml
name: CI devops 2023
on:
  push:
    branches: #list des branches qui déclanches un event
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04 #sur ubuntu 22.04
    env: #on déclare des varaibles d'environnement
      working-directory: ./TP/Part_01/BackendAPI/simple-api-student
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0  #on checkout sur la bonne branch et le bon commit

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17 #mise en place de JDK17 avec sa distribution et sa version
        uses: actions/setup-java@v3
        with:
          distribution: 'adopt'
          java-version: '17'

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify  #et la commande à exécuter pour les test
        working-directory: ${{env.working-directory}} #on met à jour le repertoire d'exécution
```

Document your quality gate configuration.
```
run: mvn -B verify sonar:sonar -Dsonar.projectKey=maximedubois1_simple-api -Dsonar.organization=maximedubois1 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
working-directory: ${{env.working-directory}}
```
J'ai modifier les différentes information lié à mon projet et mon orga sonar et j'ai modifier le chemin du fichier pom.xml, et ajouter le tocker à github


main.yml
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
    env:
      working-directory: ./TP/Part_01/BackendAPI/simple-api-student
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'adopt'
          java-version: '17'

     #finally build your app with the latest command
        run: |
          mvn clean verify
          mvn -B verify sonar:sonar -Dsonar.projectKey=maximedubois1_simple-api -Dsonar.organization=maximedubois1 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
        working-directory: ${{env.working-directory}}

```

build-push.yml
```yml
name: Build push
on:
  workflow_run:
    workflows: ["CI devops 2023"]
    types:
        - completed                
jobs:
  build-and-push-docker-image:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-22.04
    env:
      backdir: ./TP/Part_01/BackendAPI/simple-api-student
      databasedir: ./TP/Part_01/database
      frontdir: ./TP/Part_01/Front
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name : Login to DockerHub
        run: docker login -u ${{ secrets.DOCKER_HUB_USERNAME }} -p ${{ secrets.DOCKER_HUB_TOKEN }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ${{env.backdir}}
          tags:  ${{secrets.DOCKER_HUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ${{env.databasedir}}
          tags:  ${{secrets.DOCKER_HUB_USERNAME}}/database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ${{env.frontdir}}
          tags:  ${{secrets.DOCKER_HUB_USERNAME}}/front:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

# TP3 Ansible

## TD

```
maxime.dubois.1@tpc18:~/Documents/devops$ ansible all -m ping -i ./ansible/hosts -u centos --private-key=~/Documents/id_rsa
maxime.dubois.1.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

## TP

3-1 Document your inventory and base commands

```yml
all: # on créer le group all
  vars: #on définit des varibales
    ansible_user: centos #nom d'utilisateur utilisé par ansible
    ansible_ssh_private_key_file: ~/Documents/id_rsa # clé privé utilisé pour ce connecter
  children:
    prod: # on créer le group prod
      hosts: maxime.dubois.1.takima.cloud #on déclare les hots de ce groupe
```

3-2 Document your playbook
```yml
# on choisi les cibles
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:

  - name: Install device-mapper-persistent-data
    yum: #avec le module yum
      name: device-mapper-persistent-data #on install ce programme
      state: latest #a la dernière version

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command: #Avec le module commande
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo #on execute cette commande

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip: #Avec le module pip
      name: docker
      executable: pip3
    vars: #on ajoute des variables
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker
```


role back
```yml
# tasks file for roles/app-back
- name: Create BACK container
  community.docker.docker_container:
    name: back
    image: maximedubois1cpe/tp-devops-simple-api:latest
    networks:
      - name: app-network
    env_file: /tmp/.env  #je récupere le fichier .env copier par le role env

```

role Database
```yml
---
# tasks file for roles/app-database
- name: Create DATABASE container
  community.docker.docker_container:
    name: database
    image: maximedubois1cpe/database:latest
    volumes:
      - data:/var/lib/postgresql/data
    networks:
      - name: app-network
    env_file: /tmp/.env

```

role network
```yml

---
# tasks file for roles/network
- name: Create Network
  community.docker.docker_network:
    name: app-network

```

role env
```yml
---
# tasks file for roles/env
- name: Copier le fichier .env vers la machine cible
  ansible.builtin.copy:
    src: ../../.env
    dest: /tmp/.env


```

role front
```yml
---
# tasks file for roles/app-front
- name: Creatde FRONT container
  community.docker.docker_container:
    name: httpd
    image: maximedubois1cpe/front:latest
    networks:
      - name: app-network
    ports: #on expose le port 80
      - "80:80"

```

httpd.conf
```
Listen 8080
[...]
#pour le front :
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://www:80/
ProxyPassReverse / http://www:80/
</VirtualHost>

#pour le back
<VirtualHost *:8080>
ProxyPreserveHost On
ProxyPass / http://back:8080/
ProxyPassReverse / http://back:8080/
</VirtualHost>
```

```yml
---
# tasks file for roles/proxy
- name: Create Proxy container
  community.docker.docker_container:
    name: httpd
    image: maximedubois1cpe/proxy:latest
    networks:
      - name: app-network
    ports:
      - 80:80
      - 8080:8080
    recreate: true
    pull: true
```