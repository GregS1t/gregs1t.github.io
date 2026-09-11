---
layout: post
title: "Conda : un aide-mémoire en questions"
date: 2026-09-11
description: "J'ai encore oublié cette commande Conda...Un aide-mémoire pratique."
tags: [conda, python, environnements, outils]
published: true
lang: fr
categories: [tutorials]
giscus_comments: false
related_posts: false
toc:
  sidebar: left
---

Comme pour les aides-mémoire de Git et SLURM, celui-ci existe surtout pour arrêter de chercher les mêmes trois commandes tous les quelques mois. Les environnements Conda sont simples sur le principe, mais l'option exacte dont on a besoin (`--from-history`, `-n` ou `-p`, `--prune`...) ne reste jamais vraiment en mémoire.

Le billet suit la même structure : des **problèmes courants**, formulés comme on les rencontre réellement, puis quelques **références thématiques** pour aller plus loin.
Ce n'est pas un article figé par nature. Il s'enrichit au fil de mes besoins, il se corrige au fil de mes découvertes d'erreur. 
---

## Problèmes courants

### Premiers pas

#### Je veux créer un nouvel environnement

```bash
conda create -n mon-env python=3.11                  # Environnement nommé
conda create -p ./envs/mon-env python=3.11            # Environnement dans un chemin précis
conda create -n mon-env python=3.11 numpy pandas      # Avec des paquets dès la création
```

---

#### Je veux activer / désactiver un environnement

```bash
conda activate mon-env
conda deactivate
```

---

#### Je ne me souviens plus des environnements que j'ai, ni de ce qu'il y a dedans

```bash
conda env list                       # Tous les environnements de la machine
conda list                           # Paquets de l'environnement actif
conda list -n mon-env                # Paquets d'un environnement précis
```

---

### Sauvegarder et recréer des environnements

#### Je veux sauvegarder mon environnement pour pouvoir le recréer plus tard

```bash
conda env export > environment.yml                  # Export complet, avec les builds exacts
conda env export --from-history > environment.yml   # Seulement les paquets installés explicitement
```

**ATTENTION** : `--from-history` est plus portable entre systèmes d'exploitation (Linux, macOS, Windows), mais ne conserve pas les versions exactes résolues des dépendances. Utilise l'export complet si reproduire l'état précis compte pour toi.

---

#### Je veux recréer un environnement à partir d'un fichier `.yml`

```bash
conda env create -f environment.yml                  # Utilise le nom écrit dans le fichier
conda env create -f environment.yml -n nouveau-nom    # Force un nom différent
```

---

#### Je veux sauvegarder ça quelque part en ligne

Un `environment.yml` est un petit fichier texte, donc un dépôt Git (privé si besoin) ou une synchronisation vers Google Drive / Dropbox fonctionne très bien. L'intérêt des exports `--from-history` est justement d'être assez légers et stables pour être versionnés.

```bash
git add environment.yml
git commit -m "chore: mise à jour de l'environnement conda"
git push
```

---

#### Je veux mettre à jour un environnement existant à partir d'un `.yml` modifié

```bash
conda env update -f environment.yml --prune          # --prune retire les paquets qui ne sont plus listés
```

---

### Usage quotidien

#### Je veux installer ou retirer des paquets

```bash
conda install numpy pandas
conda install -n mon-env numpy                        # Dans un environnement précis
conda install -c conda-forge astropy                   # Depuis un canal précis
conda remove numpy
```

---

#### Je veux mettre à jour des paquets

```bash
conda update numpy                    # Un seul paquet
conda update --all                    # Tout l'environnement actif
conda update conda                    # Conda lui-même
```

---

#### J'ai mélangé conda et pip et je ne suis pas sûr que ce soit propre

Installe d'abord avec conda dès qu'un paquet y est disponible, puis termine avec pip pour ce qui manque. Faire l'inverse tend à casser la résolution des dépendances.

```bash
conda install numpy pandas
pip install un-paquet-absent-de-conda
```

Si les paquets installés via pip doivent apparaître dans l'export :

```bash
conda env export > environment.yml       # Les paquets pip sont listés dans une section "pip:"
```

---

#### Je veux dupliquer un environnement

```bash
conda create --name nouvel-env --clone ancien-env
```

---

#### Je veux supprimer un environnement

```bash
conda env remove -n mon-env
```

---

### Configuration

#### Je veux gérer les canaux (channels)

```bash
conda config --add channels conda-forge
conda config --set channel_priority strict
conda config --show channels
```

---

#### Je veux savoir où mes environnements sont réellement stockés

```bash
conda info --envs
conda config --show envs_dirs
```

---

### Dépannage

#### Conda est très lent à résoudre les dépendances

Passe au solveur `libmamba`, bien plus rapide et devenu la valeur par défaut depuis Conda 23.10, mais il vaut la peine de vérifier si une installation plus ancienne utilise encore le solveur classique.

```bash
conda install -n base conda-libmamba-solver
conda config --set solver libmamba
```

---

#### Mon disque est plein de vieux caches de paquets

```bash
conda clean --all                     # Supprime les paquets en cache, les tarballs, les fichiers inutilisés
conda clean --dry-run                 # Prévisualise ce qui serait supprimé
```

---

#### Je veux vérifier les incohérences d'un environnement

```bash
conda list --revisions                # Historique des changements de l'environnement
conda install --revision 2            # Revenir à une révision précédente
```

---

## Références

- Documentation officielle Conda : [docs.conda.io](https://docs.conda.io)
- Aide-mémoire Conda officiel (PDF) : [docs.conda.io/projects/conda/en/latest/user-guide/cheatsheet.html](https://docs.conda.io/projects/conda/en/latest/user-guide/cheatsheet.html)
- `conda <commande> --help` : aide intégrée pour n'importe quelle commande
