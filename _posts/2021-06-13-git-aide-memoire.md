---
layout: post
title: "Git : aide-mémoire en questions"
date: 2021-06-13
description: "Un aide-mémoire pratique pour maîtriser Git — des premiers pas aux usages avancés, organisé par problèmes concrets."
tags: [git, développement, workflow, outils]
categories: [tutoriels]
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---
Il y a quelques années, j'ai donné des formations Git/Gitlab/GitHub a des collègues. J'ai décidé de reprendre le contenu des slides qui avait un peu vieilli et de le formuler sous une autre forme.
J'ai organisé le billet en deux parties. D'abord les **problèmes courants** — formulés comme tu les rencontres, du plus simple au plus avancé. Ensuite les **références thématiques** pour aller plus loin sur chaque sujet. Je ne suis clairement pas exhaustif mais j'espère que tu trouveras ton bonheur !
Ca me sert aussi d'aide-mémoire perso pour éviter d'avoir à rechercher tout le temps les mêmes commandes. Donc ce contenu est régulièrement enrichi de mes trouvailles.

---

## Problèmes courants

### Je démarre

#### J'ai tout mon travail sur mon disque dur et je veux le sauvegarder

```bash
cd mon-projet/
git init                                        # Initialise un dépôt local
git add .                                       # Ajoute tous les fichiers
git commit -m "Initial commit"                  # Première sauvegarde
```

Pour le pousser sur un serveur distant (GitHub, GitLab...) :

```bash
git remote add origin git@github.com:user/repo.git
git push -u origin main
```

À partir de là, chaque `git commit` + `git push` est une sauvegarde horodatée et versionnée.

---

#### Je veux récupérer le code d'un collègue

```bash
git clone git@github.com:user/repo.git          # Avec SSH (recommandé)
git clone https://github.com/user/repo.git      # Avec HTTPS
```

---

#### Je ne sais pas où j'en suis dans mes modifications

```bash
git status                      # Fichiers modifiés, stagés, non suivis
git diff                        # Détail des modifications non stagées
git diff --staged               # Détail des modifications en staging
git log --oneline -10           # Les 10 derniers commits
```

---

### Je travaille seul

#### Git ouvre vim et je ne sais pas en sortir

Taper `:q!` pour quitter sans sauvegarder, ou `:wq` pour sauvegarder et quitter.

Pour ne plus avoir ce problème :

```bash
git config --global core.editor "nano"
git config --global core.editor "code --wait"   # VS Code
```

---

#### J'ai oublié un fichier dans mon dernier commit

```bash
git add fichier_oublie.py
git commit --amend --no-edit        # Ajoute le fichier sans changer le message
```

**ATTENTION** : Ne pas amender un commit déjà poussé sur un dépôt partagé.

---

#### Je veux corriger le message de mon dernier commit

```bash
git commit --amend -m "Nouveau message correct"
```

---

#### J'ai commité sur `main` au lieu de ma branche

```bash
git branch ma-feature               # Crée la branche avec le commit
git reset HEAD~1 --mixed            # Annule le commit sur main (modifs conservées)
git switch ma-feature               # Bascule sur la bonne branche
```

---

#### Mon historique est illisible (10 commits "fix", "wip", "test")

Réécrire les derniers commits avant de pousser :

```bash
git rebase -i HEAD~10
```

Dans l'éditeur : `squash` pour fusionner, `reword` pour renommer, `drop` pour supprimer.

---

#### Je veux annuler mes modifications locales et repartir du dernier commit

```bash
git restore fichier.py              # Annule les modifs d'un fichier (irréversible)
git restore .                       # Annule toutes les modifs locales (irréversible)
git clean -fd                       # Supprime aussi les fichiers non suivis
```

---

#### J'ai fait un `reset --hard` et perdu mon travail

Le reflog conserve l'historique de toutes les actions sur `HEAD` pendant 90 jours.

```bash
git reflog                          # Retrouver le hash du commit perdu
git checkout abc1234                # Se repositionner dessus
git branch recuperation abc1234     # En faire une branche
```

---

#### Je veux mettre mon travail de côté sans committer

```bash
git stash                           # Empile les modifs en cours
git stash pop                       # Réapplique le dernier stash
git stash list                      # Liste tous les stashs
```

---

### Je travaille en équipe

#### Mon push est rejeté

```bash
# Message typique : "rejected [...] non-fast-forward"
git pull --rebase                   # Récupère les modifs distantes et rejoue les tiennes
git push
```

---

#### Je veux récupérer les modifications de mes collègues sans tout casser

```bash
git fetch origin                    # Récupère sans toucher au code local
git log origin/main --oneline       # Inspecte ce qui est arrivé
git merge origin/main               # Intègre explicitement
```

Préférer `fetch` + inspection + `merge` explicite plutôt que `git pull` aveugle.

---

#### Deux collègues ont modifié le même fichier

```bash
git status                          # Voir les fichiers en conflit
# Éditer les fichiers : résoudre les marqueurs <<<<, ====, >>>>
git add fichier_resolu.py
git merge --continue
# Pour tout abandonner :
git merge --abort
```

Avec un outil visuel :

```bash
git mergetool                       # Ouvre l'outil configuré (meld, VS Code...)
```

---

#### Je veux synchroniser mon fork avec le dépôt original

```bash
git remote add upstream git@github.com:original/repo.git
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

---

#### Mon `push --force` a écrasé le travail d'un collègue

Toujours utiliser `--force-with-lease` à la place : il vérifie que personne n'a poussé entre-temps.

```bash
git push --force-with-lease
```

---
<!-->
#### Mes fins de ligne créent des conflits avec mes collègues (Linux/Windows)

Créer un `.gitattributes` à la racine du projet :

```
* text=auto
*.py text eol=lf
*.sh text eol=lf
*.bat text eol=crlf
*.fits binary
*.png binary
```

```bash
git add --renormalize .             # Appliquer les règles aux fichiers existants
git commit -m "Normalize line endings"
```
-->
---

#### J'ai poussé un fichier sensible (`.env`, clé API)

**Étape 1 — Révoquer la clé immédiatement.** Le remote en a une copie.

**Étape 2 — Supprimer le fichier de tout l'historique :**

```bash
pip install git-filter-repo
git filter-repo --path .env --invert-paths
git push --force-with-lease
```

**Étape 3 — Ajouter le fichier à `.gitignore`.**

---

### Mon dépôt est dans un sale état

#### Je cherche qui a modifié cette ligne

```bash
git blame fichier.py
git blame -L 10,25 fichier.py       # Lignes 10 à 25 seulement
```

---

#### Je ne sais pas quel commit a introduit ce bug

`git bisect` fait une recherche dichotomique dans l'historique.

```bash
git bisect start
git bisect bad                      # Le commit actuel est buggé
git bisect good v1.2.0              # Ce tag fonctionnait
# Git bascule sur un commit médian → tester, puis :
git bisect good                     # ou : git bisect bad
# Répéter jusqu'à identification. Puis :
git bisect reset
```

Avec un script de test automatisé :

```bash
git bisect run python tests/test_regression.py
```

---

#### Mon merge est un désastre, je veux tout annuler

Pendant le merge (avant le commit de merge) :

```bash
git merge --abort
```

Après le commit de merge :

```bash
git revert -m 1 HEAD                # Crée un commit qui annule le merge
```

---

#### Mon dépôt est devenu énorme

Identifier les gros objets dans l'historique :

```bash
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print $3, $4}' \
  | sort -rn \
  | head -20
```

```bash
git gc --aggressive --prune=now     # Nettoyage et compression

# Pour les gros fichiers binaires : Git LFS
git lfs install
git lfs track "*.fits"
git add .gitattributes
```

---

#### Je veux repartir d'une version propre sans perdre l'historique

```bash
git revert abc1234                  # Annule un commit précis (crée un nouveau commit)
git revert HEAD~3..HEAD             # Annule les 3 derniers commits
```

Pour revenir brutalement à un état passé (destructif, à n'utiliser que seul) :

```bash
git reset --hard abc1234
git push --force-with-lease
```

---

## Configuration

### Débutant

#### Configurer son identité

```bash
git config --global user.name "Prénom Nom"
git config --global user.email "vous@example.com"
```

---

#### Comprendre les niveaux de configuration

Git applique la configuration dans cet ordre (du plus prioritaire au moins prioritaire) :

| Niveau | Fichier | Portée |
|---|---|---|
| `--local` | `.git/config` | Le dépôt courant uniquement |
| `--global` | `~/.gitconfig` | Tous les dépôts de l'utilisateur |
| `--system` | `/etc/gitconfig` | Tous les utilisateurs de la machine |

```bash
git config --local user.email "pro@labo.fr"     # Surcharge pour ce dépôt
git config --list --show-origin                 # Voir toutes les valeurs actives
```

---

#### Ignorer des fichiers avec `.gitignore`

```
# Fichiers Python compilés
__pycache__/
*.pyc

# Environnements virtuels
.venv/

# Fichiers de données volumineux
*.fits
*.hdf5

# Secrets
.env
```

```bash
git check-ignore -v fichier.fits    # Vérifier quelle règle s'applique
```

---

### Intermédiaire

#### Configurer des alias

```bash
git config --global alias.st "status"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.undo "reset HEAD~1 --mixed"
```

---

#### Exclure des fichiers localement sans toucher `.gitignore`

Éditer `.git/info/exclude` — même syntaxe que `.gitignore`, non versionné, non partagé.

---

#### Normaliser les fins de ligne avec `.gitattributes`

Voir la section *Problèmes courants → Mes fins de ligne créent des conflits*.

---

### Avancé
<!--
#### Configurer des identités différentes selon les projets (`includeIf`)

Dans `~/.gitconfig` :

```ini
[includeIf "gitdir:~/travail/"]
    path = ~/.gitconfig-travail

[includeIf "gitdir:~/perso/"]
    path = ~/.gitconfig-perso
```

Chaque fichier cible contient son propre `[user]`.

---


#### Configurer des hooks partagés dans l'équipe

Les hooks dans `.git/hooks/` ne sont pas versionnés. Pour les partager :

```bash
mkdir .githooks/
git config core.hooksPath .githooks/
```

Exemple de hook `pre-commit` qui lance les tests :

```bash
#!/bin/bash
python -m pytest tests/ -q || exit 1
```

```bash
chmod +x .githooks/pre-commit
```

---

#### Signer ses commits avec GPG

```bash
gpg --list-secret-keys --keyid-format LONG
git config --global user.signingkey <ID_CLÉ>
git config --global commit.gpgsign true
git log --show-signature
```

---
-->

## Workflow quotidien

### Débutant

#### Le cycle de base : modifier → stager → committer

```bash
git add fichier.py              # Stager un fichier
git add -p                      # Stager interactivement par blocs
git commit -m "Description courte et précise"
```

Règle pratique pour les messages : compléter *"Ce commit…"*.

---

#### Créer une branche et basculer dessus

```bash
git switch -c ma-feature        # Créer + basculer
git switch main                 # Basculer sur une branche existante
git branch                      # Lister les branches locales
git branch -d ma-feature        # Supprimer une branche fusionnée
```

---

### Intermédiaire

#### Fusionner une branche

```bash
git switch main
git merge ma-feature            # Fast-forward si possible
git merge --no-ff ma-feature    # Force un commit de merge (conserve la trace)
```

---

#### `merge` vs `rebase` : quand utiliser quoi ?

| | `merge` | `rebase` |
|---|---|---|
| Historique | Préservé, non linéaire | Réécrit, linéaire |
| Usage | Intégration de branches | Nettoyage avant merge |
| Règle | Toujours OK | **Jamais sur une branche publique partagée** |

```bash
git switch ma-feature
git rebase main                 # Rejoue ma-feature par-dessus main
```

---

#### Voir l'historique efficacement

```bash
git log --oneline --graph --all --decorate
git log --author="Greg" --since="2 weeks ago"
git log --grep="fix" --oneline
git log -- src/model.py                         # Historique d'un fichier
git log --follow -- ancien_nom.py               # Suit les renommages
```

---

#### Comparer des branches ou des commits

```bash
git diff main..ma-feature
git log main..ma-feature --oneline              # Commits dans ma-feature pas encore dans main
git diff HEAD~3 HEAD -- fichier.py
```

---

### Avancé

#### Réécrire plusieurs commits (`rebase` interactif)

```bash
git rebase -i HEAD~5
```

Dans l'éditeur : `pick`, `squash`, `reword`, `drop`, `edit`.

---

#### Travailler sur plusieurs branches en parallèle (`worktree`)

```bash
git worktree add ../projet-hotfix hotfix/bug-42
git worktree list
git worktree remove ../projet-hotfix
```

---

#### Appliquer un commit précis d'une autre branche (`cherry-pick`)

```bash
git cherry-pick abc1234
git cherry-pick abc1234..def5678    # Plage de commits
```

---

## Collaboration / Remotes

### Débutant

#### Ajouter et gérer un remote

```bash
git remote add origin git@github.com:user/repo.git
git remote -v
git remote set-url origin git@github.com:user/nouveau-repo.git
```

---

#### Pousser une branche

```bash
git push -u origin ma-feature       # Première fois (crée le tracking)
git push                            # Fois suivantes
git push origin --delete ma-feature # Supprimer la branche distante
```

---

### Intermédiaire

#### `fetch` vs `pull` : quelle différence ?

- `git fetch` : récupère les modifications distantes sans les intégrer. Sûr à tout moment.
- `git pull` : `fetch` + `merge` (ou `rebase`). Modifie la branche courante.

```bash
git fetch origin
git log origin/main --oneline       # Inspecter avant d'intégrer
git merge origin/main
```

---

#### Marquer une version avec un tag

```bash
git tag -a v1.0.0 -m "Version 1.0.0"
git push origin v1.0.0
git push origin --tags
git tag -d v1.0.0                   # Supprimer localement
git push origin --delete v1.0.0     # Supprimer sur le remote
```

---

### Avancé

#### Inclure un autre dépôt (`submodule`)

```bash
git submodule add https://github.com/lib/lib.git libs/lib
git submodule update --init --recursive
git submodule update --remote
```

---

#### Partager un dépôt sans remote (`bundle`)

```bash
git bundle create archive.bundle --all
git clone archive.bundle nouveau-depot
git bundle verify archive.bundle
```

---

#### Échanger des patches entre dépôts

```bash
git format-patch HEAD~3             # Génère 3 fichiers .patch
git am *.patch                      # Applique dans un autre dépôt
```

---

#### Existe-t-il une convention pour écrire de bons messages de commit ?

Oui. La convention la plus adoptée s'appelle **Conventional Commits** ([conventionalcommits.org](https://www.conventionalcommits.org)).

**Format :**

```
<type>(<scope>): <description courte>

[corps optionnel]

[footer optionnel]
```

**Les types standards :**

| Type | Usage |
|---|---|
| `feat` | Nouvelle fonctionnalité |
| `fix` | Correction de bug |
| `docs` | Documentation uniquement |
| `style` | Formatage, pas de changement logique |
| `refactor` | Ni feat ni fix — restructuration |
| `test` | Ajout ou correction de tests |
| `chore` | Maintenance, dépendances, CI |
| `perf` | Amélioration de performance |

**Exemples :**

```
feat(model): add KL regularization to VAE encoder
fix(nms): move reshape inside conditional block
docs(api): update OTFClient usage examples
chore(deps): bump torch from 2.1 to 2.3
```

**Les règles de base** (Tim Pope, 2008 — toujours la référence) :

- Titre : **50 caractères max**, pas de point final
- Si corps : **ligne vide** entre le titre et le corps
- Corps : **72 caractères par ligne**, explique le *pourquoi* pas le *quoi*
- Utiliser l'**impératif** : *Add feature* et non *Added feature*

**Pourquoi c'est utile :** Conventional Commits permet de générer automatiquement un changelog et de piloter le versioning sémantique (SemVer) :

```bash
# Avec semantic-release ou standard-version
feat  → bump version mineure  (1.0.0 → 1.1.0)
fix   → bump version patch    (1.0.0 → 1.0.1)
feat! → bump version majeure  (1.0.0 → 2.0.0)  # breaking change
```

Pour un projet solo ou une petite équipe, la convention complète est souvent surdimensionnée. Le minimum utile reste le préfixe de type :

```
feat: ajouter le support des tuiles 512x512
fix: corriger le reshape hors du bloc conditionnel
refactor: extraire OTFClient dans un module séparé
docs: ajouter la spec LaTeX du protocole
```

Même sans outillage automatique, ça rend `git log --oneline` immédiatement lisible.




## Débogage & Inspection

### Débutant

#### Afficher le contenu d'un commit

```bash
git show abc1234
git show HEAD
git show HEAD~2
```

---

#### Lister les fichiers suivis

```bash
git ls-files
git ls-files --others --exclude-standard    # Fichiers non suivis
```

---

### Intermédiaire

#### Chercher dans le code et l'historique

```bash
git grep "nom_fonction"                         # Dans le code actuel
git log -S "nom_fonction" --oneline             # Commits qui ont modifié cette chaîne
git log -G "regex" --oneline                    # Idem avec une regex
```

---

#### Statistiques de contribution

```bash
git shortlog -sn                        # Nombre de commits par auteur
git shortlog -sn --since="1 year"
```

---

### Avancé

#### Retrouver un commit perdu (`reflog`)

```bash
git reflog                              # Historique de toutes les actions HEAD
git checkout abc1234
git branch recuperation abc1234
```

---

#### Trouver le commit qui a introduit un bug (`bisect`)

Voir la section *Problèmes courants → Je ne sais pas quel commit a introduit ce bug*.

---

#### Vérifier et optimiser le dépôt

```bash
git fsck --full                         # Vérifier l'intégrité
git count-objects -vH                   # Taille du dépôt
git gc --aggressive                     # Nettoyage et compression
```

---

#### Inspecter les objets internes de Git

```bash
git cat-file -t abc1234     # Type : blob, tree, commit, tag
git cat-file -p abc1234     # Contenu de l'objet
git ls-tree HEAD            # Arbre du commit courant
```

---

## Références

- Documentation officielle Git : [git-scm.com/doc](https://git-scm.com/doc)
- *Pro Git* (Chacon & Straub, 2e éd.) — gratuit en ligne : [git-scm.com/book/fr/v2](https://git-scm.com/book/fr/v2)
- `man git-<commande>` — pages de manuel intégrées
