---
layout: post
title: "Docker : aide-mémoire en questions"
date: 2026-04-30
description: "Un aide-mémoire pour utiliser Docker au quotidien — des premiers pas aux usages intermédiaires, organisé par problèmes concrets."
tags: [docker, conteneurs, devops, environnement, reproductibilité]
categories: [tutoriels]
giscus_comments: false
published: false
related_posts: false
toc:
  sidebar: left
---

Après avoir suivi une formation Docker, j'ai décidé de faire comme pour Git et SLURM : consolider les commandes essentielles dans un aide-mémoire organisé par situations concrètes, du plus simple au plus courant. Je n'ai pas cherché à être exhaustif — juste à couvrir ce qu'on rencontre vraiment quand on démarre avec Docker.

---

## Problèmes courants

### Je démarre

#### Je veux vérifier que Docker est bien installé

```bash
docker --version                    # Version du client Docker
docker info                         # Informations sur le démon Docker
docker run hello-world              # Test complet : télécharge et lance un conteneur de test
```

---

#### Je veux lancer un conteneur pour tester

```bash
docker run ubuntu:22.04 echo "Bonjour depuis un conteneur"
```

Avec un terminal interactif :

```bash
docker run -it ubuntu:22.04 bash    # -i : interactif, -t : terminal
```

`-it` ensemble est le raccourci habituel pour entrer dans un conteneur en ligne de commande.

---

#### Je ne sais pas quels conteneurs tournent

```bash
docker ps                           # Conteneurs en cours d'exécution
docker ps -a                        # Tous les conteneurs (y compris arrêtés)
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

---

#### Je veux voir les images disponibles sur ma machine

```bash
docker images                       # Liste toutes les images locales
docker images -a                    # Inclut les couches intermédiaires
```

---

### Mon conteneur ne se comporte pas comme prévu

#### Mon conteneur s'arrête immédiatement

Un conteneur s'arrête dès que son processus principal se termine. Pour l'inspecter :

```bash
docker run -it mon-image bash       # Lancer en interactif pour déboguer
docker logs <CONTAINER_ID>          # Lire la sortie du conteneur arrêté
docker inspect <CONTAINER_ID>       # Détails complets (état, configuration, réseau)
```

---

#### Je veux voir les logs d'un conteneur

```bash
docker logs <CONTAINER_ID>          # Sortie complète
docker logs -f <CONTAINER_ID>       # Suivre en temps réel (comme tail -f)
docker logs --tail=50 <CONTAINER_ID> # 50 dernières lignes
```

---

#### Je veux entrer dans un conteneur qui tourne déjà

```bash
docker exec -it <CONTAINER_ID> bash
# Si bash n'est pas disponible :
docker exec -it <CONTAINER_ID> sh
```

---

#### Mon conteneur utilise trop de ressources

```bash
docker stats                        # Utilisation CPU, mémoire, réseau en temps réel
docker stats <CONTAINER_ID>         # Un seul conteneur
```

---

### Je veux gérer mes conteneurs et images

#### Je veux arrêter ou supprimer un conteneur

```bash
docker stop <CONTAINER_ID>          # Arrêt propre (signal SIGTERM)
docker kill <CONTAINER_ID>          # Arrêt forcé (signal SIGKILL)
docker rm <CONTAINER_ID>            # Supprimer un conteneur arrêté
docker rm -f <CONTAINER_ID>         # Forcer la suppression (même s'il tourne)
```

Lancer un conteneur qui se supprime automatiquement à l'arrêt :

```bash
docker run --rm ubuntu:22.04 echo "Éphémère"
```

---

#### Je veux supprimer des images

```bash
docker rmi <IMAGE_ID>               # Supprimer une image
docker rmi mon-image:1.0            # Par nom et tag
```

---

#### Mon disque est plein à cause de Docker

```bash
docker system df                    # Voir l'espace occupé par Docker
docker system prune                 # Supprimer tout ce qui n'est plus utilisé
docker system prune -a              # Inclut aussi les images non utilisées
docker image prune                  # Images sans tag ("dangling") uniquement
docker volume prune                 # Volumes non attachés
```

---

### Je veux partager des données et des ports

#### Je veux accéder à un service qui tourne dans un conteneur

Publier le port du conteneur vers la machine hôte :

```bash
docker run -p 8080:80 nginx         # Port hôte 8080 → port conteneur 80
docker run -p 5000:5000 mon-api
```

Vérifier les ports exposés :

```bash
docker port <CONTAINER_ID>
```

---

#### Je veux que le conteneur accède à mes fichiers locaux

Monter un répertoire local dans le conteneur :

```bash
docker run -v /chemin/local:/chemin/dans/conteneur mon-image
# Exemple : monter le répertoire courant
docker run -v $(pwd):/app mon-image
```

Pour un accès en lecture seule :

```bash
docker run -v $(pwd)/data:/app/data:ro mon-image
```

---

#### Je veux passer des variables d'environnement au conteneur

```bash
docker run -e MA_VARIABLE=valeur mon-image
docker run --env-file .env mon-image    # Depuis un fichier .env
```

---

## Construire ses propres images

### Je démarre

#### Comment est structuré un Dockerfile ?

Un `Dockerfile` décrit, couche par couche, comment construire une image.

```dockerfile
FROM python:3.11-slim               # Image de base

WORKDIR /app                        # Répertoire de travail dans le conteneur

COPY requirements.txt .             # Copier les dépendances d'abord (cache Docker)
RUN pip install --no-cache-dir -r requirements.txt

COPY . .                            # Copier le reste du code

CMD ["python", "mon_script.py"]     # Commande lancée au démarrage du conteneur
```

---

#### Je veux construire mon image

```bash
docker build -t mon-image:1.0 .     # . = répertoire contenant le Dockerfile
docker build -t mon-image .         # Sans tag explicite → "latest" par défaut
docker build -f autre.Dockerfile -t mon-image .
```

---

#### Quelle est la différence entre `CMD` et `ENTRYPOINT` ?

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| Rôle | Commande par défaut | Commande fixe, non remplaçable |
| Surchargeable | Oui (`docker run mon-image autre-commande`) | Non (sauf `--entrypoint`) |
| Usage typique | Scripts Python, serveurs | Outils CLI encapsulés |

Combinaison courante :

```dockerfile
ENTRYPOINT ["python"]
CMD ["mon_script.py"]               # Argument par défaut, remplaçable
```

---

#### Mon image est trop volumineuse

Quelques bonnes pratiques pour réduire la taille :

- Utiliser une image de base légère (`-slim`, `-alpine`)
- Regrouper les commandes `RUN` pour limiter le nombre de couches
- Supprimer les caches dans le même `RUN`
- Utiliser un fichier `.dockerignore`

```dockerfile
# À préférer
RUN apt-get update && apt-get install -y gcc \
    && rm -rf /var/lib/apt/lists/*
```

Exemple de `.dockerignore` :

```
__pycache__/
*.pyc
.git/
.env
data/
logs/
```

---

#### Je veux pousser mon image sur Docker Hub

```bash
docker login                                        # S'authentifier
docker tag mon-image:1.0 monuser/mon-image:1.0      # Tagger avec le compte Docker Hub
docker push monuser/mon-image:1.0                   # Pousser
docker pull monuser/mon-image:1.0                   # Récupérer depuis n'importe où
```

---

### Intermédiaire

#### Mon image se reconstruit entièrement à chaque modification

Docker met en cache chaque couche. Si une couche change, tout ce qui suit est reconstruit.
Ordre recommandé : ce qui change rarement → ce qui change souvent.

```dockerfile
# Bon ordre : dépendances avant le code source
COPY requirements.txt .
RUN pip install -r requirements.txt   # Mis en cache tant que requirements.txt ne change pas
COPY . .                              # Copié en dernier : change souvent
```

---

#### Je veux inspecter le contenu d'une image couche par couche

```bash
docker history mon-image            # Couches et tailles
docker inspect mon-image            # Configuration complète en JSON
```

---

## Docker Compose

### Je démarre

#### À quoi sert Docker Compose ?

Compose permet de définir et de lancer plusieurs conteneurs ensemble (une application + sa base de données, par exemple) via un fichier `compose.yml`.

```yaml
services:
  app:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/app
    environment:
      - DEBUG=true
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: motdepasse
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

#### Les commandes Compose essentielles

```bash
docker compose up                   # Démarre tous les services
docker compose up -d                # En arrière-plan (detached)
docker compose down                 # Arrête et supprime les conteneurs
docker compose down -v              # Inclut la suppression des volumes
docker compose logs -f              # Logs de tous les services
docker compose logs -f app          # Logs d'un service précis
docker compose ps                   # État des services
docker compose build                # Reconstruit les images
docker compose exec app bash        # Ouvrir un terminal dans un service
```

---

#### Je veux relancer uniquement un service

```bash
docker compose restart app
docker compose stop app
docker compose start app
```

---

## Monitoring & Débogage

### Débutant

#### Connaître l'état détaillé d'un conteneur

```bash
docker inspect <CONTAINER_ID>
# Extraire une information précise avec --format :
docker inspect --format='{{.State.Status}}' <CONTAINER_ID>
docker inspect --format='{{.NetworkSettings.IPAddress}}' <CONTAINER_ID>
```

---

#### Voir les événements Docker en temps réel

```bash
docker events                       # Flux d'événements (démarrages, arrêts, etc.)
docker events --filter type=container
```

---

### Intermédiaire

#### Copier des fichiers entre le conteneur et la machine hôte

```bash
docker cp <CONTAINER_ID>:/app/logs/output.log ./output.log    # Conteneur → hôte
docker cp ./config.yml <CONTAINER_ID>:/app/config.yml         # Hôte → conteneur
```

---

#### Sauvegarder et restaurer une image sans registry

```bash
docker save -o mon-image.tar mon-image:1.0      # Exporter
docker load -i mon-image.tar                    # Importer sur une autre machine
```

---

## Références

- Documentation officielle Docker : [docs.docker.com](https://docs.docker.com)
- Référence Dockerfile : [docs.docker.com/reference/dockerfile](https://docs.docker.com/reference/dockerfile/)
- Documentation Docker Compose : [docs.docker.com/compose](https://docs.docker.com/compose/)
- Docker Hub (registry public) : [hub.docker.com](https://hub.docker.com)
