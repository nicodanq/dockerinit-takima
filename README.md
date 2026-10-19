# TP01 – Discover Docker

Application 3-tiers conteneurisée : **httpd** (reverse proxy) → **backend** (API Spring Boot) → **database** (PostgreSQL).

## Structure

```
tp01/
├── compose.yaml          # orchestration des 3 services
├── .env                  # identifiants de la base (lus par compose)
├── postgres/             # image base de données + scripts d'init SQL
├── simpleapi/            # API Spring Boot (build multistage)
├── httpd/                # Apache en reverse proxy + httpd.conf
├── java-app/             # Hello World Java (premier essai)
└── volume/               # données Postgres persistées (ignoré par git)
```

## Ce qu'on a fait

1. **Base de données** : image `postgres:17.2-alpine`, scripts `01-CreateScheme.sql` et `02-InsertData.sql` copiés dans `/docker-entrypoint-initdb.d`, données persistées dans un volume, visualisation avec Adminer.
2. **Hello World Java** : compilation et exécution de `Main.java` dans un conteneur `eclipse-temurin`.
3. **Backend** : API Spring Boot `simple-api` construite par un Dockerfile multistage (Maven + JDK pour le build, JRE seul pour l'exécution). La connexion à la base est paramétrée par variables d'environnement dans `application.yml`.
4. **HTTP server** : `httpd:2.4` avec un `httpd.conf` récupéré depuis le conteneur puis modifié pour faire reverse proxy vers le backend.
5. **Docker Compose** : les 3 services orchestrés avec 2 réseaux, des healthchecks, un ordre de démarrage garanti et les identifiants dans un `.env`.

Résultat : l'API est accessible sur `http://localhost/departments/IRC/students`.

## Problèmes rencontrés

| Problème | Cause | Solution |
|---|---|---|
| Options ignorées (`docker run ... nicolas/java -p 3333:3333`) | Tout ce qui suit le nom de l'image est passé en argument au programme, pas à Docker | Mettre `-p`, `-v`, `--name`... **avant** le nom de l'image |
| Apache ne démarre pas | Dans `httpd.conf`, les directives étaient coupées sur deux lignes (`ProxyPreserveHost` / `On`) | Une directive par ligne : `ProxyPreserveHost On` |
| API inaccessible sur `localhost:80` | Port publié en `3001:80` | `-p 80:80` / `"80:80"` |
| `Proxy Error: DNS lookup failure for: backend` | Aucun conteneur nommé `backend` sur le réseau. Sur un réseau Docker, le **nom du conteneur sert de nom d'hôte** | Nommer le conteneur de l'API `backend` (même nom que dans `ProxyPass`) et le mettre sur le même réseau |
| `UnknownHostException: postgres` au démarrage de l'API | La base n'était pas lancée sous le nom attendu par `POSTGRES_HOST` | Lancer dans l'ordre postgres → backend → httpd. Avec compose : `depends_on` + `condition: service_healthy` |
| 502 après avoir recréé `backend` | Apache garde en cache l'ancienne IP du backend | `docker restart httpd` |
| httpd du compose ne démarre pas | Les conteneurs lancés à la main tournaient encore et occupaient le port 80 | `docker rm -f httpd backend postgres` avant `docker compose up` |
| Identifiants modifiés non pris en compte | Postgres n'initialise la base (utilisateur, scripts SQL) que si le volume est vide | `docker compose down -v` pour repartir d'un volume vide |
| `permission denied ... docker.sock` | Utilisateur hors du groupe `docker` | `sudo` (ou `usermod -aG docker $USER`) |

## Réponses aux questions

### 1-1 Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?

- Un `ENV` est **gravé dans l'image** : n'importe qui ayant accès à l'image (registry, `docker inspect`, `docker history`) peut lire le mot de passe. Il se retrouve aussi dans git avec le Dockerfile.
- Avec `-e`, la **même image** sert dans tous les environnements (dev, test, prod) avec des identifiants différents, sans rebuild.
- On peut ensuite aller plus loin avec `--env-file`, un `.env` hors git, ou des secrets (Docker/Kubernetes, Vault).

### 1-2 Pourquoi attacher un volume au conteneur Postgres ?

Le système de fichiers d'un conteneur est **éphémère** : un `docker rm` supprime toutes les données. Le volume stocke `/var/lib/postgresql/data` **en dehors du conteneur**, sur l'hôte. Les données survivent donc à la suppression, à la recréation ou à la mise à jour de l'image. Effet de bord : les scripts de `/docker-entrypoint-initdb.d` ne s'exécutent que si ce volume est vide.

### 1-3 Conteneur base de données : Dockerfile et commandes

`postgres/Dockerfile` :
```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd

COPY docker-entrypoint-initdb.d/ /docker-entrypoint-initdb.d/
```

Commandes :
```bash
docker network create app-network
docker build -t nicolas/postgres ./postgres
docker run -d --name postgres --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v "$(pwd)/volume/data":/var/lib/postgresql/data \
  nicolas/postgres

# interface web pour vérifier les tables et les données (http://localhost:8090)
docker run -d --name adminer --network app-network -p 8090:8080 adminer
```
Les `-e` surchargent les `ENV` du Dockerfile.

### 1-4 Pourquoi un build multistage ? Explication du Dockerfile

Le build a besoin d'un **JDK + Maven** (lourds), l'exécution seulement d'un **JRE** et du `.jar`. Le multistage sépare les deux : l'image finale ne contient que le JRE et le jar. Elle est donc plus légère, avec moins de surface d'attaque, sans code source ni outils de build. Et on n'a besoin ni de Java ni de Maven sur sa machine pour builder.

```dockerfile
# --- Stage 1 : build ---
FROM eclipse-temurin:21-jdk-alpine AS myapp-build   # image avec JDK, nommée "myapp-build"
ENV MYAPP_HOME=/opt/myapp                           # variable pour le dossier de travail
WORKDIR $MYAPP_HOME                                 # se place dans /opt/myapp
RUN apk add --no-cache maven                        # installe Maven
COPY pom.xml .                                      # copie la définition du projet
COPY src ./src                                      # copie le code source
RUN mvn package -DskipTests                         # compile et produit target/*.jar

# --- Stage 2 : run ---
FROM eclipse-temurin:21-jre-alpine                  # nouvelle image, JRE seulement
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar  # récupère uniquement le jar du stage 1
ENTRYPOINT ["java", "-jar", "myapp.jar"]           # lance l'API au démarrage du conteneur
```

Amélioration possible : copier `pom.xml` puis lancer `mvn dependency:go-offline` **avant** `COPY src`. Docker met alors en cache la couche des dépendances, et elles ne sont retéléchargées que si le `pom.xml` change.

### 1-5 Pourquoi un reverse proxy ?

- **Point d'entrée unique** : seul le port 80 est exposé, le backend et la base restent invisibles depuis l'extérieur.
- On peut **changer, déplacer ou multiplier** les backends sans que le client change d'URL (load balancing).
- Il centralise les fonctions transverses : **HTTPS/TLS**, compression, cache, en-têtes de sécurité, logs d'accès.
- Il peut servir un **front-end statique** et l'API sur le même domaine, ce qui évite les problèmes de CORS.

### 1-6 Pourquoi docker-compose est-il si important ?

Lancer 3 conteneurs à la main demande de recréer réseaux, volumes, variables, noms et ordre de démarrage à chaque fois, avec beaucoup d'erreurs possibles (cf. problèmes ci-dessus). Compose décrit toute l'application dans **un seul fichier déclaratif et versionné**. Avec une commande, on démarre, arrête ou reconstruit tout. C'est reproductible sur n'importe quelle machine et ça documente l'architecture.

### 1-7 Commandes docker-compose principales

| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | Build les images si besoin et démarre tous les services en arrière-plan |
| `docker compose down` | Arrête et supprime conteneurs et réseaux |
| `docker compose down -v` | Idem + supprime les volumes (perte des données) |
| `docker compose ps` | État des services (dont `healthy` / `unhealthy`) |
| `docker compose logs -f backend` | Suit les logs d'un service |
| `docker compose build` | Reconstruit les images |
| `docker compose restart httpd` | Redémarre un service |
| `docker compose exec database psql -U usr -d db` | Exécute une commande dans un service |
| `docker compose config` | Valide le fichier et affiche la config finale (avec les variables du `.env`) |

### 1-8 Le fichier docker-compose

Voir [`compose.yaml`](compose.yaml). Choix faits :

- **3 services** construits depuis leurs dossiers (`build: ./postgres`, `./simpleapi`, `./httpd`).
- **Ségrégation réseau** :
  - `front-network` : httpd ↔ backend
  - `back-network` : backend ↔ database

  httpd ne peut pas joindre la base, seul le backend fait le lien.
- **Seul httpd publie un port** (`80:80`). Le backend et la base ne sont pas exposés sur l'hôte.
- **Variables d'environnement** : identifiants dans `.env` à la racine, injectés via `${POSTGRES_USER}` etc. Le backend reçoit `POSTGRES_HOST: database` (nom du service).
- **Volume nommé** `db-postgres` pour persister les données.
- **Healthchecks** :
  - database : `pg_isready`
  - backend : `wget` sur `/actuator/health`, avec un `start_period` de 30 s le temps que Spring démarre
  - httpd : test du port 80 en bash (pas de curl/wget dans l'image)
- **Ordre de démarrage** : `depends_on` avec `condition: service_healthy`, donc database → backend → httpd. Chaque service attend que le précédent soit réellement prêt, pas seulement démarré.

### 1-9 Publication sur Docker Hub

```bash
docker login

docker tag tp01-database nicodnq25/tp01-database:1.0
docker tag tp01-backend  nicodnq25/tp01-backend:1.0
docker tag tp01-httpd    nicodnq25/tp01-httpd:1.0

docker push nicodnq25/tp01-database:1.0
docker push nicodnq25/tp01-backend:1.0
docker push nicodnq25/tp01-httpd:1.0
```

Images publiées :
- [nicodnq25/tp01-database](https://hub.docker.com/r/nicodnq25/tp01-database)
- [nicodnq25/tp01-backend](https://hub.docker.com/r/nicodnq25/tp01-backend)
- [nicodnq25/tp01-httpd](https://hub.docker.com/r/nicodnq25/tp01-httpd)

Attention : `docker login` sans `sudo` enregistre les identifiants pour l'utilisateur courant. `sudo docker push` utilise ceux de root, donc soit tout faire avec sudo, soit ajouter son utilisateur au groupe `docker`.

### 1-10 Pourquoi mettre ses images dans un registry en ligne ?

- **Partager** les images avec l'équipe, la CI et les serveurs, sans rebuild sur chaque machine.
- **Déployer exactement le même artefact** partout (dev, test, prod), ce qui évite le « ça marche chez moi ».
- **Versionner** avec des tags (`1.0`, `1.1`...) et pouvoir revenir à une version précédente.
- Base d'un pipeline **CI/CD** : la CI build et push, les serveurs pull.
