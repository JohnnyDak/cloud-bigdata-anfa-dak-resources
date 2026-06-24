# Rendu - Séance 2

**Nom et prénom :** DAKOU Koudjo  
**Identifiant GitHub :** JohnnyDak  
**Date de soumission :** 24/06/2026

---

## Résumé de la séance

J’ai écrit un Dockerfile pour conteneuriser un script PySpark analysant le référentiel d’Anfa, construit l’image `anfa-analyse:v1`, appliqué les bonnes pratiques (`.dockerignore`, ordre des instructions pour optimiser le cache), puis orchestré une stack à 3 services (MinIO, Jupyter, et mon image custom) avec Docker Compose. Enfin, j’ai exploré les données du bucket `anfa-raw` depuis un notebook Jupyter via `boto3` et `pandas`.

---

## Étapes principales

1. **Écriture du Dockerfile et construction de l’image `anfa-analyse:v1`** (taille observée : **1,17 Go**).
2. **Mise en place du `.dockerignore`** et observation du cache de Docker.
3. **Écriture du `docker-compose.yml`** orchestrant MinIO, Jupyter, et l’image custom.
4. **Création du notebook `exploration_minio.ipynb`** qui lit les données depuis MinIO via `boto3` et `pandas`.

---

## Captures d’écran

### docker compose ps
![docker compose ps](captures/docker-ps.png)

### Notebook Jupyter
![Notebook Jupyter](captures/jupyter-pandas.png)

---

## Bonus multi-stage

- **Taille de `anfa-analyse:v1` :** 1,17 Go
- **Taille de `anfa-analyse:v2-multistage` :** 1,17 Go
- **Gain :** Pas de gain visible
![Image de comparaison](captures/comparer-multi-stage.png)

---

## Exercices d’application

### Exercice 1 – QCM conceptuel

**1.1** – **C. Un conteneur partage le noyau de la machine hôte.**  
*Justification :* Contrairement à une VM qui a son propre noyau, un conteneur utilise le noyau de l’hôte via les namespaces et cgroups.

**1.2** – **B. L’image est un modèle figé en lecture seule ; le conteneur est une instance en cours d’exécution.**  
*Justification :* L’image est un template statique (couches de fichiers), le conteneur est l’exécution de cette image avec sa propre couche en lecture‑écriture.

**1.3** – **B. Les namespaces.**  
*Justification :* Docker utilise les namespaces (PID, NET, UTS, etc.) pour isoler les processus d’un conteneur.

**1.4** – **A. Les cgroups.**  
*Justification :* Les cgroups (control groups) permettent de limiter les ressources CPU, mémoire, etc.

**1.5** – **B. Dans une machine virtuelle Linux invisible gérée par Docker Desktop.**  
*Justification :* Sous macOS, Docker Desktop exécute un hyperviseur léger (HyperKit) avec une VM Linux.

**1.6** – **B. La société d’origine qui a créé et open-sourcé Docker en 2013.**  
*Justification :* Docker (alors dotCloud) a open‑sourcé le projet en 2013, popularisant la conteneurisation.

**1.7** – **C. Docker a apporté un format d’image portable, une CLI simple et un registre public, en s’appuyant sur les mêmes primitives que LXC.**  
*Justification :* LXC existait déjà, Docker a simplifié l’expérience et ajouté l’écosystème (Docker Hub, images, etc.).

**1.8** – **B. Open Container Initiative — une norme ouverte pour les images et le runtime.**  
*Justification :* L’OCI standardise les formats d’images et les runtimes pour garantir l’interopérabilité.

---

### Exercice 2 – Lecture et analyse d’un Dockerfile

**Dockerfile fourni :**
```dockerfile
FROM python:3.11
WORKDIR /application
COPY . /application
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "main.py"]


2.1 – Explication de chaque instruction :

Instruction	Explication
FROM python:3.11	Utilise l’image officielle Python 3.11 comme base.
WORKDIR /application	Définit le répertoire de travail à /application dans le conteneur.
COPY . /application	Copie tout le contenu du contexte de build dans /application.
RUN pip install -r requirements.txt	Installe les dépendances Python listées dans requirements.txt.
EXPOSE 5000	Déclare que le conteneur écoute sur le port 5000 (information, pas de redirection).
CMD ["python", "main.py"]	Commande par défaut exécutée lorsque le conteneur démarre.
2.2 – Différence entre EXPOSE 5000 et –p 5000:5000 :

EXPOSE est une documentation : elle indique le port utilisé par l’application, mais ne publie rien sur l’hôte.

–p 5000:5000 (ou -p) publie le port 5000 du conteneur sur le port 5000 de l’hôte, rendant le service accessible depuis l’extérieur.

2.3 – Deux problèmes dans ce Dockerfile et leurs corrections :

Ordre des instructions inefficace pour le cache

Problème : COPY . /application avant RUN pip install invalide le cache à chaque modification du code, forçant une réinstallation de toutes les dépendances.

Correction : Copier d’abord requirements.txt, installer les dépendances, puis copier le code.

dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
Image de base non optimisée (Python 3.11 ~ 900 Mo)

Problème : L’image python:3.11 est volumineuse.

Correction : Utiliser python:3.11-slim ou python:3.11-alpine pour réduire la taille.

2.4 – Version corrigée du Dockerfile :

dockerfile
# Image de base plus légère
FROM python:3.11-slim

# Créer un utilisateur non-root
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copier requirements.txt d'abord (cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Puis le code
COPY . .

# Changer d'utilisateur pour l'exécution
USER appuser

EXPOSE 5000
CMD ["python", "main.py"]
Exercice 3 – Cas pratiques
3.1 – Le build qui échoue

Dockerfile erroné :

dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install -r requirements.txt   # ← erreur
COPY . .
CMD ["python", "main.py"]
a. Cause précise de l’erreur :
Le fichier requirements.txt n’existe pas encore dans le conteneur au moment de l’exécution de RUN pip install, car il n’a pas été copié avant. Docker cherche le fichier dans le répertoire /app qui est vide.

b. Correction :
Copier requirements.txt avant d’exécuter pip install :

dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
c. Pourquoi cette erreur illustre une mauvaise compréhension de Docker :
L’étudiant a supposé que RUN pip install pouvait accéder aux fichiers du contexte de build (hôte) sans les avoir copiés au préalable. Docker exécute chaque instruction dans une couche isolée ; seuls les fichiers copiés avec COPY ou ADD sont disponibles.

3.2 – Le conteneur qui ne voit pas l’autre

docker-compose.yml fourni :

yaml
services:
  api:
    build: ./api
    environment:
      DATABASE_URL: "postgresql://user:password@localhost:5432/anfa"
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: anfa
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
a. Erreur dans DATABASE_URL :
Le conteneur api ne peut pas utiliser localhost pour joindre la base de données, car chaque conteneur a son propre localhost. Dans un réseau Docker Compose, les services sont accessibles par leur nom (ici db). L’URL correcte est :
postgresql://user:password@db:5432/anfa

Exercice 4 – Optimisation d’image
Dockerfile donné :

dockerfile
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3 python3-pip
RUN apt-get install -y curl wget git build-essential
COPY . /app
WORKDIR /app
RUN pip3 install -r requirements.txt
CMD ["python3", "downloader.py"]
Problèmes identifiés (au moins 4) :

Image de base lourde (ubuntu:22.04 ~ 80 Mo) → préférer python:3.11-slim (plus léger et déjà avec Python).

Absence de nettoyage du cache APT → les fichiers temporaires alourdissent l’image. Ajouter rm -rf /var/lib/apt/lists/* après les installations.

Multiples RUN → chaque RUN crée une couche. Fusionner en un seul RUN pour réduire le nombre de couches.

Installation de paquets inutiles (curl, wget, git, build-essential) alors que seule l’application Python est nécessaire. Supprimer les paquets non requis.

COPY . /app avant pip install → invalide le cache à chaque modification du code. Copier requirements.txt en premier.

Version corrigée :

dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python3", "downloader.py"]
Exercice 5 – Mini-cas d’architecture
a. Services à conteneuriser dans docker-compose.yml :

Service	Rôle
ftp-downloader	Script Python qui lit les données GPS depuis un FTP, les nettoie et les écrit dans MinIO.
minio	Stockage objet S3 compatible pour les données agrégées et les résultats.
jupyter	Notebook Jupyter pour l’exploration des données et la création de graphiques.
b. restart policy pour le script FTP :
Je choisirais on-failure.
Justification : Le script est un batch qui s’exécute puis s’arrête. En cas d’échec (erreur réseau, FTP indisponible), on veut qu’il redémarre pour tenter à nouveau. on-failure permet cela sans redémarrer en boucle en cas de succès.

c. Passer la date au script sans modifier le code Python :

Mécanisme 1 : Variable d’environnement (DATE_PROCESS=2026-06-24) dans docker-compose.yml.

Mécanisme 2 : Passer la date en argument à la commande du conteneur (ex. CMD ["python", "downloader.py", "2026-06-24"]).

Recommandation : Variable d’environnement, car elle permet de changer la date sans reconstruire l’image (en modifiant simplement le docker-compose.yml).

d. Réponse à l’équipe (pourquoi un conteneur séparé plutôt que dans Jupyter) :

Le script de nettoyage est un job batch qui doit s’exécuter régulièrement de manière autonome, sans interface utilisateur.

Le séparer permet d’orchestrer son exécution (redémarrage, logs, monitoring) de façon indépendante, sans risque d’interférer avec l’environnement interactif du notebook.

De plus, cela respecte le principe de responsabilité unique et facilite les mises à jour du pipeline sans perturber l’analyse exploratoire.

e. Squelette de docker-compose.yml (15‑25 lignes) :

yaml
services:
  ftp-downloader:
    build: ./downloader
    environment:
      - DATE_PROCESS=${DATE:-2026-06-24}
      - MINIO_ENDPOINT=minio:9000
    volumes:
      - ./data:/data
    restart: on-failure
    depends_on:
      - minio

  minio:
    image: minio/minio:latest
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3

  jupyter:
    image: jupyter/scipy-notebook:latest
    ports:
      - "8888:8888"
    environment:
      JUPYTER_TOKEN: anfa-token
    volumes:
      - ./notebooks:/home/jovyan/work
    depends_on:
      minio:
        condition: service_healthy

volumes:
  minio-data:
Difficultés rencontrées
Au départ, le bucket anfa-raw était vide car j’avais exécuté docker compose down -v (suppression du volume). J’ai recréé le bucket via boto3 depuis le notebook et rechargé les fichiers CSV via l’interface web MinIO.

Sur Windows, le fichier Dockerfile.multistage a été créé avec une faute de frappe (ockerfile.multistage), ce qui a bloqué la construction multi-stage. J’ai corrigé le nom avec ren.

