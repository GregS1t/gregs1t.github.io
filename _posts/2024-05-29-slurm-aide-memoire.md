---
layout: post
title: "SLURM : aide-mémoire en questions"
date: 2024-05-29
description: "Un aide-mémoire pour soumettre et gérer des jobs SLURM — des premiers pas aux usages avancés, organisé par problèmes concrets."
tags: [slurm, hpc, cluster, gpu, calcul-intensif]
categories: [tutoriels]
giscus_comments: true
related_posts: false
toc:
  sidebar: left
---

J'utilise SLURM depuis assez récemment et j'avais stocké les commandes que j'utilise tout le temps dans un fichier texte. J'ai remis les choses en ordre pour en faire un petit aide-mémoire. Comme pour le billet que j'avais fait sur Git, je traite d'abord les **problèmes courants** du plus simple au plus avancé, ensuite j'aborde des **références thématiques** pour aller plus loin. 

---

## Problèmes courants

### Je démarre

#### Je veux lancer mon premier script sur le cluster

Créer un fichier `job.sh` :

```bash
#!/bin/bash
#SBATCH --job-name=mon-job
#SBATCH --output=logs/%j.out      # %j = identifiant du job
#SBATCH --error=logs/%j.err
#SBATCH --time=01:00:00           # Durée max : 1 heure
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G

python mon_script.py
```

```bash
mkdir -p logs
sbatch job.sh
```

---

#### Je ne sais pas si mon job tourne

```bash
squeue -u $USER                   # Jobs en cours ou en attente
squeue -u $USER -l                # Version longue (état, nœud, temps restant)
```

États courants : `PD` (en attente), `R` (en cours), `CG` (en fin d'exécution).

---

#### Je veux voir ce qui est disponible sur le cluster

```bash
sinfo                             # Partitions et état des nœuds
sinfo -o "%P %a %l %D %t %N"     # Vue compacte
```

---

### Mon job ne démarre pas

#### Mon job est bloqué en `PD` (pending) depuis longtemps

```bash
squeue -u $USER --start           # Estimation du démarrage
scontrol show job <JOBID>         # Raison du blocage (champ "Reason")
```

Raisons fréquentes :

| Raison | Cause |
|---|---|
| `Resources` | Ressources demandées pas encore disponibles |
| `Priority` | D'autres jobs ont priorité |
| `QOSMaxCpuPerUserLimit` | Quota CPU utilisateur atteint |
| `ReqNodeNotAvail` | Nœud demandé hors service |

---

#### Mon job est refusé à la soumission

Vérifier les limites de la partition :

```bash
scontrol show partition <nom>     # Limites de temps, CPU, mémoire
sacctmgr show user $USER          # Quotas associés au compte
```

---

#### Je veux lancer un job interactif pour tester

```bash
srun --ntasks=1 --cpus-per-task=4 --mem=8G --time=01:00:00 --pty bash
```

Sur Jean Zay, spécifier la partition :

```bash
srun --partition=gpu_p1 --gres=gpu:1 --time=00:30:00 --pty bash
```

---

### Mon job a planté

#### Mon job s'est terminé mais je ne sais pas pourquoi

```bash
sacct -j <JOBID> --format=JobID,State,ExitCode,Elapsed,MaxRSS
```

États d'échec courants :

| État | Cause probable |
|---|---|
| `FAILED` | Erreur dans le script (voir `.err`) |
| `TIMEOUT` | Temps limite dépassé |
| `OUT_OF_MEMORY` | Mémoire insuffisante |
| `CANCELLED` | Annulé manuellement ou par l'admin |

```bash
cat logs/<JOBID>.err              # Lire la sortie d'erreur
```

---

#### Mon job a été tué pour dépassement mémoire

Augmenter `--mem` dans le script. Pour estimer la mémoire réellement utilisée :

```bash
sacct -j <JOBID> --format=MaxRSS  # Pic mémoire du job terminé
```

`MaxRSS` est en Ko. Ajouter une marge de 20-30 % pour la soumission suivante.

---

#### Mon job a été tué par `TIMEOUT`

Augmenter `--time` ou découper le calcul en étapes avec des checkpoints.

Pour connaître la durée maximale autorisée sur une partition :

```bash
scontrol show partition <nom> | grep MaxTime
```

---

#### Je veux annuler un job

```bash
scancel <JOBID>                   # Annuler un job précis
scancel -u $USER                  # Annuler tous ses jobs
scancel --name=mon-job            # Annuler par nom
```

---

### Je veux optimiser mes soumissions

#### Je veux lancer le même script avec des paramètres différents (sweep)

Utiliser un **array job** :

```bash
#!/bin/bash
#SBATCH --job-name=sweep
#SBATCH --array=0-9               # 10 jobs : indices 0 à 9
#SBATCH --output=logs/%A_%a.out   # %A = job array ID, %a = index
#SBATCH --time=00:30:00
#SBATCH --mem=8G

python train.py --seed $SLURM_ARRAY_TASK_ID
```

Avec une liste de valeurs :

```bash
#SBATCH --array=1,5,10,50,100
```

Limiter le nombre de jobs simultanés :

```bash
#SBATCH --array=0-99%10           # 100 jobs, 10 en parallèle max
```

---

#### Je veux utiliser un GPU

```bash
#!/bin/bash
#SBATCH --partition=gpu           # Nom de la partition GPU (varie selon le cluster)
#SBATCH --gres=gpu:1              # 1 GPU
#SBATCH --gres=gpu:a100:2         # 2 GPU de type A100 (si disponible)
#SBATCH --cpus-per-task=8         # CPUs associés (en général 8-10 par GPU)
#SBATCH --mem=40G
```

Vérifier que le GPU est bien alloué depuis le job :

```bash
nvidia-smi
echo $CUDA_VISIBLE_DEVICES
```

---

#### Je veux utiliser plusieurs nœuds (MPI)

```bash
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=1

srun python -m torch.distributed.launch --nproc_per_node=32 train_dist.py
```

---

#### Mon environnement conda n'est pas chargé dans le job

Les jobs SLURM ne sourcent pas `.bashrc` par défaut. Initialiser conda explicitement :

```bash
#!/bin/bash
#SBATCH ...

source ~/.bashrc                          # ou :
source /path/to/miniconda3/etc/profile.d/conda.sh
conda activate mon-env

python mon_script.py
```

---

#### Je veux charger des modules logiciels

```bash
module avail                      # Lister les modules disponibles
module load python/3.11 cuda/12.1
module list                       # Modules chargés
module purge                      # Décharger tout
```

Dans le script SLURM :

```bash
#!/bin/bash
#SBATCH ...

module purge
module load python/3.11 cuda/12.1
source ~/.venv/bin/activate
python mon_script.py
```

---

## Écrire un script SLURM

### Débutant

#### Les directives `#SBATCH` essentielles

```bash
#SBATCH --job-name=nom            # Nom affiché dans squeue
#SBATCH --output=logs/%j.out      # Sortie standard (%j = JOBID)
#SBATCH --error=logs/%j.err       # Sortie d'erreur
#SBATCH --time=HH:MM:SS           # Durée maximale
#SBATCH --ntasks=1                # Nombre de tâches (processus MPI)
#SBATCH --cpus-per-task=4         # CPUs par tâche (threads OpenMP)
#SBATCH --mem=16G                 # Mémoire totale par nœud
```

---

#### Rediriger les sorties dans un seul fichier

```bash
#SBATCH --output=logs/%j.log      # stdout et stderr dans le même fichier
```

Ou depuis la ligne de commande :

```bash
sbatch --output=logs/test.log job.sh
```

---

#### Recevoir un email à la fin du job

```bash
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=vous@labo.fr
```

Valeurs possibles : `BEGIN`, `END`, `FAIL`, `ALL`.

---

### Intermédiaire

#### Spécifier une partition et un compte

Sur les clusters nationaux, les heures sont décomptées par projet (compte) :

```bash
#SBATCH --partition=gpu_p2
#SBATCH --account=abc1234         # Code projet DARI/GENCI
```

---

#### Utiliser des variables d'environnement SLURM dans le script

```bash
echo "Job ID      : $SLURM_JOB_ID"
echo "Nœud        : $SLURM_JOB_NODELIST"
echo "CPUs alloués : $SLURM_CPUS_PER_TASK"
echo "Array index : $SLURM_ARRAY_TASK_ID"
```

---

#### Copier les données vers le scratch local avant le calcul

Sur les grands clusters, les I/O sur le stockage partagé sont coûteuses. Utiliser le scratch local du nœud :

```bash
SCRATCH=$SLURM_TMPDIR             # Variable fournie par SLURM sur certains clusters
cp -r $HOME/data/dataset/ $SCRATCH/
python train.py --data $SCRATCH/dataset/
```

Sur Jean Zay : `$SCRATCH` et `$STORE` sont les espaces recommandés pour les données.

---

### Avancé
<!--
#### Enchaîner des jobs avec des dépendances

```bash
JOB1=$(sbatch --parsable preprocess.sh)
JOB2=$(sbatch --parsable --dependency=afterok:$JOB1 train.sh)
sbatch --dependency=afterok:$JOB2 postprocess.sh
```

Types de dépendances utiles :

| Type | Déclenchement |
|---|---|
| `afterok` | Si le job précédent a réussi (exit 0) |
| `afternotok` | Si le job précédent a échoué |
| `afterany` | Dans tous les cas |
| `after` | Dès que le job précédent a démarré |

---
-->

#### Reprendre un calcul interrompu (checkpointing)

Sauvegarder l'état régulièrement dans le script Python :

```python
# Sauvegarder toutes les N epochs
if epoch % save_every == 0:
    torch.save({"epoch": epoch, "model": model.state_dict()}, "checkpoint.pt")
```

Dans le script SLURM, reprendre si un checkpoint existe :

```bash
if [ -f checkpoint.pt ]; then
    python train.py --resume checkpoint.pt
else
    python train.py
fi
```

---

## Monitoring & Débogage

### Débutant

#### Suivre l'avancement d'un job en temps réel

```bash
tail -f logs/<JOBID>.out          # Suivre la sortie en direct
watch -n 5 squeue -u $USER        # Rafraîchir squeue toutes les 5 secondes
```

---

#### Voir les ressources consommées par un job terminé

```bash
sacct -j <JOBID> --format=JobID,JobName,State,Elapsed,CPUTime,MaxRSS,ExitCode
```

---

### Intermédiaire

#### Voir l'utilisation en temps réel d'un job qui tourne

```bash
srun --jobid=<JOBID> --pty bash   # Se connecter au nœud du job
# Puis :
top -u $USER
nvidia-smi                        # Si GPU
```

Ou directement :

```bash
ssh $(squeue -j <JOBID> -o "%N" --noheader)   # Se connecter au nœud
```

---

#### Estimer l'efficacité d'un job

```bash
seff <JOBID>                      # Résumé CPU efficiency + mémoire (si disponible)
```

Un job efficace utilise > 80 % des CPUs alloués et < 90 % de la mémoire demandée.

---

#### Consulter l'historique de ses jobs

```bash
sacct -u $USER --starttime=2025-01-01 --format=JobID,JobName,State,Elapsed,CPUTime
sacct -u $USER --starttime=now-7days  # Dernière semaine
```

---

### Avancé

#### Connaître sa consommation d'heures de calcul

```bash
sreport cluster AccountUtilizationByUser start=2025-01-01 end=now
# Sur Jean Zay :
idrquota                          # Outil spécifique IDRIS
```

---

#### Diagnostiquer un nœud défaillant

```bash
scontrol show node <nom-noeud>    # État détaillé d'un nœud
sinfo -n <nom-noeud>              # Rapide
```

---

## Configuration et environnement

### Débutant

#### Configurer son environnement par défaut pour SLURM

Créer un fichier `~/.slurm/defaults` (si supporté par le cluster) ou factoriser les options communes dans un script source :

```bash
# config_slurm.sh — à sourcer en début de script
BASE_ARGS="--account=abc1234 --partition=gpu_p2 --mail-user=vous@labo.fr --mail-type=FAIL"
```
<!--
---

#### Organiser ses scripts et logs

Structure recommandée :

```
projet/
├── scripts/
│   ├── train.sh
│   └── preprocess.sh
├── logs/
│   └── .gitkeep
├── src/
└── data/
```

```bash
#SBATCH --output=logs/%x_%j.out   # %x = nom du job, %j = JOBID
```
-->
---

### Intermédiaire

#### Utiliser un environnement reproductible avec conda

```bash
# Créer l'environnement une fois sur le nœud de login
conda env create -f environment.yml
conda activate mon-env
pip install -e .

# Dans le script SLURM
source ~/.bashrc
conda activate mon-env
```

Pour éviter les problèmes de PATH, toujours utiliser le chemin absolu de l'interpréteur :

```bash
PYTHON=$(which python)
$PYTHON mon_script.py
```

<!--
### Avancé

#### Utiliser des conteneurs Singularity/Apptainer

Disponible sur la plupart des clusters HPC (Docker n'est pas autorisé).

```bash
# Construire depuis une image Docker (sur sa machine)
singularity pull mon_image.sif docker://pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime

# Soumettre avec Singularity
singularity exec --nv mon_image.sif python train.py
# --nv : passe les GPU au conteneur
```

Dans le script SLURM :

```bash
#SBATCH --gres=gpu:1

singularity exec --nv $HOME/images/mon_image.sif python train.py
```
-->
---

## Références

- Documentation SLURM : [slurm.schedmd.com/documentation.html](https://slurm.schedmd.com/documentation.html)
- Documentation Jean Zay (IDRIS) : [www.idris.fr/jean-zay](http://www.idris.fr/jean-zay/)
- `man sbatch`, `man srun`, `man sacct` — pages de manuel intégrées
