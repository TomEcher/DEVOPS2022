# TP DevOps 2022

## Création de nos conteneurs

### Database

 1. Création de l'image :
- Etape 1 : Création du fichier Dockerfile (à partir de l'extrait fournis)
- Etape 2 : Build une nouvelle image : 
&ensp;&ensp;Pour créer une image nous allons utiliser la commande `docker build` avec l'option : 
&ensp;&ensp;&ensp;&ensp; `-t` afin de renseigner un tag
```
docker build -t techer/tp01 .

[+] Building 21.5s (5/5) FINISHED
```
2. Lancement de l'image
- Après la lecture de la documentation, nous pouvons identifier le port utilisé par PostgreSQL (5432), cette information nous sera utile pour le lancement...
&ensp;&ensp;Pour lancer notre conteneur nous allons utiliser la commande `docker run` avec les options : 
&ensp;&ensp;&ensp;&ensp; `-p` afin de réaliser un mappage des ports
&ensp;&ensp;&ensp;&ensp; `-d` afin de détacher l'exécution du docker
&ensp;&ensp;&ensp;&ensp; `--name` afin de donner un nom au conteneur
```
docker run -p 8888:5432 -d --name tp1 techer/tp01
```

> Afin d'éviter de rentrer les paramètres de port pour chaques lancement nous aurions pu ajouter au Dockerfile : EXPORT 5432 (si nous souhaitons ouvrir le port 5432)

- Après avoir lancé le docker nous pouvons tenter la connexion avec un client SQL comme DBeaver par exemple avec les paramètres suivants :
>@IP : localhost
Port : 8888
database : db
username : usr
password : pwd

&ensp;&ensp; Si nous souhaitons améliorer la sécurité et donc éviter de retrouver les identifiants de connexion dans le Dockerfile, il suffit de ne pas mettre les variables d'environnement dans celui-ci mais plutôt lors de l'exécution de la commande RUN avec l'argument `-e`
3. Ajouter l'exécution des scripts SQL :
```
FROM  postgres:11.6-alpine
COPY  SQLScript/*  /docker-entrypoint-initdb.d
ENV  POSTGRES_DB=db  \
     POSTGRES_USER=usr  \
     POSTGRES_PASSWORD=pwd
```
- Si nous souhaitons persisté nos données (et donc ne pas perdre nos datas) nous pouvons utilisé l'option `-v` : 
> docker run -p 8888:5432 -v /Users/tomecher/Documents/DevOps/TP01/PG/mypgdata:/var/lib/postgresql/data --name tp1 techer/tp01

### Java 

1. **Création du Dockerfile :** 
- FROM : image avec Java (openJDK11)
- COPY : Cela nous permettera de copier notre fichier Java sur le conteneur
- RUN : Cela nous permettera de compiler notre java
- CMD : Execute notre Java et nous renvoie le retour
````
FROM  openjdk:11
COPY  Main.java  /usr/src/app/
RUN  javac  /usr/src/app/Main.java
CMD  ["java",  "/usr/src/app/Main"]
````

2.  **Multistage build :**

Le mutlistage build permet de décomposer la fonction de compilation et d'exécution en deux conteneurs différents.
Cela permet qu'un conteneurs assure une seule et unique fonction. Nous nous retrouvons donc avec deux petits conteneurs et non un gros conteneurs plus lourd.
```
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY simple-api/pom.xml .
COPY simple-api/src ./src
RUN mvn package -DskipTests

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```
### Http server

**Création de notre Dockerfile pour notre serveur web**

Voici les actions réaliser par ce Dockerfile :

- On ce base sur l'image httpd
- On copie notre fichier html dans le répertoire de notre conteneur prévu à cet effet 
```
FROM  httpd
COPY  index.html  /usr/local/apache2/htdocs/
```
**Fichier HTML pour notre serveur web**

```
<html>
	<h1> Bienvenue sur mon site ! </h1>
	<h2> Toto de la Clergery </h2>
</html>
```
> Pour lancer ce conteneur il suffit de réaliser la commande suivante :
```
docker run -p 80:80 -d --name tp1web httpd
```
4. **Reverse Proxy**
Nous allons mettre en place un reverse proxy, qui nous permettra de joindre notre API (créer précédemment) ainsi que notre serveur web que nous venons de créer.
Grâce à cela on expose uniquement un conteneur, qui en fonction du chemin redirigera sur le bon conteneur.
	- /api : API du conteneur backend
	- /tdlc : Serveur web avec une page index.html 

**Dockerfile de notre conteneur : reverseproxy**

Voici les actions réaliser par ce Dockerfile :

- On ce base sur l'image apache-proxy de diouxx 
- On copie notre fichier de configuration dans le répertoire de notre conteneur prévu à cet effet 
```
FROM  diouxx/apache-proxy
COPY  site1.conf  /opt/proxy-conf/
```
> Il serait préférable d'utiliser une image officiel, comme discuté nous ne sommes pas à l'abri que le propriétaire de cette image souhaite ajouter des applications malveillante à l'intérieur
> > Pour les prochaines fois, penser à utiliser les **images officielles**

**Fichier de configuration pour notre reverse proxy**
```
<VirtualHost *:80>
	ServerName tom.echer.takima.cloud

	ProxyPass "/api"  "http://api:8080/"
	ProxyPassReverse "/api"  "http://api:8080/"

	ProxyPass "/tdlc"  "http://web/"
	ProxyPassReverse "/tdlc"  "http://web/"
</VirtualHost>
```
### Création du docker compose :

Nous allons créer un docker compose, ce qui permet de lancer tous les conteneurs avec les paramètres choisis très rapidement. Nous pouvons aussi les builds très facilement.
Ce fichier est composé de deux parties majeures :

- Service : Celle ci va contenir tous les conteneurs
	- api : celui-ci correspond au backend. Avec les paramètres suivants :
		- networks : "my-network" (utiliser pour interconnecter nos dockers)
		- depends_on : "db" (permet de stipuler que ce conteneur dépend du conteneur db)
	- db : celui-ci correspond à la base de donnée. Avec les paramètres suivants :
		- networks : "my-network" (utiliser pour interconnecter nos dockers)
	- web : celui-ci correspond à notre serveur web. Avec les paramètres suivants :
		- networks : "my-network" (utiliser pour interconnecté nos dockers)
	- httpd : celui-ci correspond au reverse proxy. Avec les paramètres suivants :
		- networks : "my-network" (utiliser pour interconnecté nos dockers)
		- depends_on : "db" "web" (permet de stipuler que ce conteneur dépend du conteneur api et web
		- ports : "80:80" ("permet de dire que le port 80 de l'hôte correspond au port 80 de ce conteneur)
		
```
version: '3.7'
services:
	api:
		build: "./backend"
		networks:
			- "my-network"
		depends_on:
			- "db"

	db:
		build: "./PG"
		networks:
			- "my-network"

	web:
		build: "./web"
		networks:
			- "my-network"
			
	httpd:
		build: "./reverseProxy"
		ports:
			- "80:80"
		networks:
			- "my-network"
		depends_on:
			- "api"
			- "web"

networks:
	my-network:
		name: network01
```

### Publication de nos images :
1. Connexion à docker : 
```
docker login
```
2. Création du tag :
```
docker tag db echert/my-database:1.0
```
4. Push de notre image :
```
docker push echert/my-database
```
> Nous pouvons poster nos images pour plusieurs raisons : 
> - Facilité le partage entre plusieurs personnes. (Utile si nous souhaitons fournir à d'autre utilisateur une image qui serait susceptible de l'utiliser)
> - Réaliser la mise en production via des outils en ligne comme github actions.
## CI / CD : Utilisation de GitHub Actions

### Configuration de GitHub Actions :
#### Build et test avec Maven

Explication du fichier de configuration relatif à GitHub Actions :
Nous pouvons découper ce fichier en deux parties majeures :
- on: celui-ci permet de décider à quel moment nous souhaitons réaliser une actions, dans notre cas on indique que nous souhaitons déclencher **lors d'un push sur la branches main**.
- jobs : permet d'entrer les différents paramètres d'un 'job' :
	- runs-on: On indique sur quel type d'élément nous souhaitons réaliser les différentes étapes
	- steps : On indique les différentes étapes
	Dans notre cas nous souhaitons java dans notre environnement, nous devons donc indiquer que nous souhaitons l'installer. Et par la suite nous allons build et tester notre code avec Maven. Nous allons avoir à la fin un résultat.
```
name: CI DevOps 2022 CPE
on:
	push:
		branches:
			- main
		pull_request:

jobs:
	test-backend:
		runs-on: ubuntu-18.04
		steps:
			- uses: actions/checkout@v2.3.3
			- uses: actions/setup-java@v2
			  with:
				distribution: 'temurin'
				java-version: '11'
				cache: 'maven'

			- name: Build and test with Maven
			  run: mvn clean verify --file TP01/backend/simple-api/pom.xml
			  
```
#### Ajout de la publications des images sur dockerhub :

Maintenant nous souhaitons retrouver nos images sur DockerHub à chaques fois que nous réalisons des pushs. Nous devons donc maintenants rajouter une nouvelles tâches qui réalisera ce rôle.
Voici les éléments, expliqué brièvement: 
1. Connexion à dockerhub. Afin de sécurisé, nous décidons d'utiliser les variables secrète de github
> Pour utiliser ces variables il suffit d'utiliser de mettre ce code dans le code `${{secrets.DOCKERHUB_USERNAME}}`
2. Nous pouvons par la suite commencer le build et la publication de nos images.
```
	build-and-push-docker-image:
		needs: test-backend
		runs-on: ubuntu-latest
		steps:
			- name: Login to DockerHub
			  run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PASSWORD }}
			- name: Checkout code
			  uses: actions/checkout@v2
			- name: Build image and push backend
			  uses: docker/build-push-action@v2
			  with:
				push: ${{ github.ref == 'refs/heads/main' }}
				context: ./TP01/backend
				tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:backend

			- name: Build image and push db
			  uses: docker/build-push-action@v2
			  with:
				push: ${{ github.ref == 'refs/heads/main' }}
				context: ./TP01/PG
				tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:database

			- name: Build image and push reverse proxy
			  uses: docker/build-push-action@v2
			  with:
				push: ${{ github.ref == 'refs/heads/main' }}
				context: ./TP01/reverseProxy
				tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:rproxyv3

			- name: Build image and push apache
			  uses: docker/build-push-action@v2
			  with:
				push: ${{ github.ref == 'refs/heads/main' }}
				context: ./TP01/web
				tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:web
```
#### Rajout de Sonar :

Pour ce faire il est nécessaire de créer un compte sur SonarCloud et par la suite, un organisation. Cela nous permettra de récupérer un SONAR_TOKEN que nous utiliserons dans notre code :

> Modification du fichier de configuration de GitHub Actions :
```
- name: Build and test with Maven
  env:
	GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any

	SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: mvn -B verify sonar:sonar -Dsonar.projectKey=TomEcher_DEVOPS2022 -Dsonar.organization=tomecher -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONAR_TOKEN }} --file TP01/backend/simple-api/pom.xml
```

## Utilisation d'Ansible pour le déploiement :

### Fichier de l'inventaire :
A l'interieur de celui-ci nous allons indiquer l'intégralité de nos machines sur lesquels nous voulons déployer, dans notre cas, cela n'a pas énormement de sens car nous en avons qu'une seule...
Dans ce fichier, nous allons indiqué les paramètres suivants : 
- Login de connexion (ansible user)
- Cle privée (ansible_ssh_private_key_file)
- Host : (hosts)
```
all:
	vars:
		ansible_user: centos
		ansible_ssh_private_key_file: /Users/tomecher/Documents/DevOps/id_rsa
	children:
		prod:
			hosts: centos@tom.echer.takima.cloud
```

> Grâce à ce fichier, nous pouvons réaliser des commandes très rapidement sans être obliger de rentrer les informations à chaque fois... (exemple de commande : ansible all -i inventories/setup.yml -m ping)

### Playbooks :

Pour la création d'un premier Playbook, j'ai utiliser le même que fournis.

### Création de rôle :
```
ansible-galaxy init roles/docker
```
Cette commande nous permet de créer un rôle docker, cependant celle-ci nous créer une multitude de fichier/repertoire. Dans notre cas nous avons uniquement besoin du chemin suivant :
`/roles/docker/tasks/main.yml` 

On retrouvera dans celui l'intégralité de la configuration de nos conteneur, par exemple :
```
---
# tasks file for roles/docker
- name: Clean packages
  command:
  cmd: dnf clean -y packages
  
- name: Install device-mapper-persistent-data
  dnf:
  name: device-mapper-persistent-data
  state: latest
  
- name: Install lvm2
  dnf:
  name: lvm2
  state: latest
  
- name: add repo docker
  command:
  cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  dnf:
  name: docker-ce
  state: present

- name: install python3
  dnf:
  name: python3

- name: Pip install
  pip:
	  name: docker

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```


### Deployer l'application  :

Pour déployer correctement l'application il suffit de créer chaque rôles et des les appeler dans notre playbook (cf. git)

### Front :

Pour commencer nous devons récupérer les fichiers sur le github. Celui-ci contient les fichiers suivant: 
- configuration pour le docker-compose
- fichier du programme/de configuration

concernant le conteur relatif au front, tous est déjà configuré.

Pour respecter le diagramme souhaité avec l'accessibilité (via les ports etc...) nous sommes obliger de modifier la configuration de notre reverse proxy.
Nous allons créer deux fichiers de configuration suivant :

**CONF 1 :**
```
Listen 8080

<VirtualHost *:8080>
	ServerName tom.echer.takima.cloud
	ProxyPass "/" "http://api:8080/"
	ProxyPassReverse "/" "http://api:8080/"
</VirtualHost>
```
**CONF 2 :**
```
<VirtualHost *:80>
	ServerName tom.echer.takima.cloud
	ProxyPass "/" "http://front/"
	ProxyPassReverse "/" "http://front/"
</VirtualHost>
```

Maintenant, en terme de configuration de nos conteneurs nous sommes OK. Cependant, maintenant nous devons modifier notre workflow afin de publier aussi l'image du conteneur représentant le front :

**Main.yml : (rajout)**
```
- name: Build image and push front
  uses: docker/build-push-action@v2
  with:
	push: ${{ github.ref == 'refs/heads/main' }}
	context: ./TP01/devops-front
	tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:front4
```

Maintenant, nous allons configurer Ansible :

La première partie est de modifier la configuration pour le role du reverse proxy :

```
---
# tasks file for roles/proxy
- name: Run HTTPD
  docker_container:
  name: rproxy
  image: echert/tp-devops-cpe:rproxyv6
  ports:
	- "80:80"
	- "8080:8080"
  networks:
	- name: "network01"
```

Par la suite nous allons ajouter le nouveau role relatif au front :

```
---
# tasks file for roles/app
- name: Run Front
  docker_container:
  name: front
  image: echert/tp-devops-cpe:front4
  networks:
	- name: "network01"
```

Pour finir, nous devons modifier le playbook :

```
- hosts: all
  gather_facts: false
  become: yes
  roles:
	- docker
	- network
	- database
	- app
	- web
	- front
	- proxy
```
