# TP PART01 - Docker

## Database

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
- Si nous souhaitons persisté nos données (et donc ne pas perdre nos data) nous pouvons utilisé l'option `-v` : 
> docker run -p 8888:5432 -v /Users/tomecher/Documents/DevOps/TP01/PG/mypgdata:/var/lib/postgresql/data --name tp1 techer/tp01

## Java 

1. Création du Dockerfile : 
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

2. 


````
docker run -p 80:80 -d --name tp1web httpd
````
