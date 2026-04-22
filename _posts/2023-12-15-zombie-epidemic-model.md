---
layout: post
title: "Quand les maths rencontrent les zombies !"
date: 2023-12-15
description: >
  Un week-end pluvieux, une série à regarder, et une question qui s'impose : et si on modélisait ça ?
tags: [EDO, simulation, zombies, last of us, epidemiology]
categories: simulation
series_order: 1
related_posts: true
toc:
  sidebar: left
math: true
---

C'est en regardant *The Last of Us* que l'idée m'est venue. La série met en scène une pandémie fongique (au chordiceps) qui transforme progressivement les humains en créatures agressives. En parallèle, le souvenir du COVID-19 était encore frais — cette période étrange où chacun suivait des courbes, des $R_0$, des taux de reproduction. On était tous devenus épidémiologistes amateurs.

Et là, entre deux épisodes, la question s'est posée naturellement : *est-ce qu'on peut modéliser une épidémie de zombies de la même façon qu'une vraie épidémie ?*

La réponse est oui. Et quelqu'un l'a fait avant moi.

## L'article qui a tout déclenché

En 2009, Munz, Hudea, Imad et Smith? ont publié *"When Zombies Attack! : Mathematical Modelling of an Outbreak of Zombie Infection"* dans *Infectious Disease Modelling Research Progress*.
L'article est sérieux, peer-reviewed, et utilise des outils épidémiologiques standard. C'est cette base que j'ai reprise pour coder les modèles en Python.

## Les modèles SIR, et pourquoi les zombies compliquent tout

En épidémiologie classique, on découpe la population en compartiments. Le modèle de base — le **SIR** — distingue les **Susceptibles** (S), les **Infectés** (I) et les **Retirés** (R, c'est-à-dire guéris ou décédés). Les équations qui gouvernent l'évolution de ces trois
populations forment un système d'équations différentielles ordinaires (EDO) couplées :

$$\frac{dS}{dt} = -\beta SI, \quad \frac{dI}{dt} = \beta SI - \gamma I, \quad \frac{dR}{dt} = \gamma I$$

Le terme $\beta SI$ capture le fait que les rencontres entre susceptibles et infectés sont proportionnelles aux deux populations. Le paramètre $\beta$ est le taux de transmission, $\gamma$ le taux de guérison.

### Ce que les zombies changent au modèle SIR

Dans une épidémie classique, les individus du compartiment **R** sont définitivement hors-jeu : guéris avec immunité, ou morts. Le flux est à sens unique — on n'en revient pas.

Les zombies brisent cette hypothèse de façon radicale : **les morts rejoignent les infectés**.
Le compartiment R devient une réserve de zombies potentiels, alimentée en permanence par la mortalité naturelle ($\delta S$) et les humains tués par des zombies ($\alpha SZ$). Un taux de résurrection $\zeta$ fait "fuir" une fraction de R vers Z à chaque instant.

C'est ce seul ajout — un flux $\zeta R \to Z$ — qui transforme un SIR en SZR, et qui change fondamentalement la dynamique : dans le SIR, l'épidémie s'éteint naturellement quand les susceptibles se raréfient. Dans le SZR, le réservoir R alimente continuellement Z, ce qui explique le résultat analytique de Munz et al. : **la coexistence stable humains-zombies est mathématiquement impossible**. L'un des deux camps disparaît nécessairement.

Le modèle SZR s'écrit :

$$\frac{dS}{dt} = \Pi - \beta SZ - \delta S$$

$$\frac{dZ}{dt} = \beta SZ + \zeta R - \alpha SZ$$

$$\frac{dR}{dt} = \delta S + \alpha SZ - \zeta R$$

où $\Pi$ est le taux de natalité, $\delta$ la mortalité naturelle, $\alpha$ le taux de destruction des zombies par les humains.

## Vers un modèle plus réaliste : le SIZRQ

Munz et al. proposent ensuite plusieurs raffinements. Le plus intéressant est le modèle **SIZRQ**, qui ajoute deux compartiments :

- **I** (Infectés) : les personnes mordues mais pas encore zombifiées — la période d'incubation   observée dans la plupart des œuvres de fiction, *The Last of Us* inclus.
- **Q** (Quarantaine) : les infectés et zombies mis à l'écart de la population générale.

Le système devient :

$$\frac{dS}{dt} = \Pi - \beta SZ - \delta S$$

$$\frac{dI}{dt} = \beta SZ - \rho I - \delta I - \kappa I$$

$$\frac{dZ}{dt} = \rho I + \zeta R - \alpha SZ - \sigma Z$$

$$\frac{dR}{dt} = \delta S + \delta I + \alpha SZ - \zeta R + \gamma Q$$

$$\frac{dQ}{dt} = \kappa I + \sigma Z - \gamma Q$$

Ici $\rho$ est le taux de zombification, $\kappa$ et $\sigma$ les taux de mise en quarantaine des infectés et des zombies, $\gamma$ le taux d'évasion de quarantaine.

### Ce que disent les paramètres

Les paramètres ne sont pas de simples lettres grecques — ils ont chacun une interprétation concrète.

**$\beta = 0.0095$** est le taux de transmission : avec $S = 500$ humains et $Z = 2$ zombies,
le terme $\beta SZ \approx 9.5$ humains mordus par jour — l'épidémie démarre vite.

**$\alpha = 0.005$** est le taux d'élimination des zombies par les humains. Plus faible que $\beta$ : les humains sont moins efficaces à tuer des zombies qu'à se faire mordre.

**$\rho = 0.025$** est la vitesse de zombification : un mordu devient zombie en $1/\rho \approx 40$ jours en moyenne. C'est le paramètre qui donne toute sa tension narrative à *The Last of Us*.

**$\zeta = 0.0001$** est le taux de résurrection spontanée : très faible, mais non nul — même sans morsure, les morts finissent par revenir.

> J'ai repris les valeurs de Munz et al. (2009), choisies pour produire des dynamiques intéressantes, pas calibrées sur des données réelles — il n'en existe pas, enfin j'espère.

## Le taux de reproduction de base $R_0$

### Un indicateur qu'on a appris à connaître pendant le COVID

Pendant l'épidémie de COVID-19, le $R_0$ était omniprésent dans les médias. On nous expliquait que si ce nombre passait sous 1, l'épidémie refluerait. Ce même indicateur, construit sur les mêmes bases mathématiques, s'applique sans modification à notre épidémie de zombies.

Il répond à une question simple : **combien de nouveaux zombies un seul zombie crée-t-il, dans une population entièrement humaine ?**

- Si $R_0 > 1$ : chaque zombie en fait plus d'un autre — l'épidémie s'emballe.
- Si $R_0 < 1$ : chaque zombie disparaît avant d'en créer un autre — l'épidémie s'éteint.

Pour le modèle SIZRQ, Munz et al. obtiennent *via* la méthode de la matrice de prochaine génération (van den Driessche & Watmough, 2002) :

$$R_0 = \frac{\rho \beta N}{(\rho + \delta + \kappa)(\alpha N + \sigma)}$$

Lisons cette formule comme une fraction **"ce qui alimente les zombies" sur "ce qui les élimine"** :

- Au numérateur : $\rho \beta N$ — la transmission ($\beta$) amplifiée par la vitesse de zombification ($\rho$) sur une population de taille $N$.
- Au dénominateur : $(\rho + \delta + \kappa)$ est le taux total de sortie du compartiment I — un mordu peut zombifier ($\rho$), mourir naturellement ($\delta$), ou être mis en quarantaine ($\kappa$). Le terme $(\alpha N + \sigma)$ représente le taux d'élimination des zombies.

**Le levier quarantaine est visible directement dans la formule** : augmenter $\kappa$ ou $\sigma$ fait grossir le dénominateur, donc réduit $R_0$. C'est exactement ce que cherchait
le confinement COVID.

## Résoudre numériquement en Python

Le système SIZRQ ne peut pas être résolu analytiquement — chaque compartiment dépendant des autres, on recourt à une intégration numérique [^1]. L'essentiel du code tient en quelques
lignes avec `scipy` :

```python
from scipy.integrate import solve_ivp
import numpy as np


def model_SIZRQ(t, y, Pi, delta, beta, zeta, alpha, rho, kappa, sigma, gamma):
    """SIZRQ zombie epidemic model ODEs."""
    Si, Ii, Zi, Ri, Qi = y
    dS_dt = Pi - beta*Si*Zi - delta*Si
    dI_dt = beta*Si*Zi - rho*Ii - delta*Ii - kappa*Ii
    dZ_dt = rho*Ii + zeta*Ri - alpha*Si*Zi - sigma*Zi
    dR_dt = delta*Si + delta*Ii + alpha*Si*Zi - zeta*Ri + gamma*Qi
    dQ_dt = kappa*Ii + sigma*Zi - gamma*Qi
    return [dS_dt, dI_dt, dZ_dt, dR_dt, dQ_dt]


S0, Z0 = 500.0, 2.0
y0 = [S0, 0.0, Z0, 0.01*S0, 0.0]
t = np.linspace(0, 80, 5000)

sol = solve_ivp(
    model_SIZRQ, [t[0], t[-1]], y0,
    args=(0.0, 0.0001, 0.0095, 0.0001, 0.005, 0.025, 0.01, 0.01, 0.01),
    t_eval=t, method='RK45'
)
```

[^1]: Le système SIZRQ forme un ensemble d'équations différentielles couplées — chaque compartiment dépend des autres — ce qui rend une solution analytique (formule exacte) impossible dans le cas général. On recourt donc à une résolution numérique : RK45, un intégrateur de la
famille Runge-Kutta, estime l'erreur à chaque pas de temps en comparant deux approximations d'ordres différents (4 et 5), et ajuste automatiquement la taille du pas en conséquence. C'est la méthode par défaut de `solve_ivp` dans la librairie `scipy`.

## Un graphique ou deux

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/Zombie_article/sizrq_quarantine.png"
       width="80%"
       alt="Impact de la quarantaine sur le pic de zombies.">
  <figcaption>
    Figure 1 - Impact de la quarantaine sur le pic de zombies.
  </figcaption>
</figure>

Les deux panneaux racontent la même histoire depuis deux angles différents.

**Panneau gauche — l'évolution temporelle.** Sans quarantaine ($\kappa = \sigma = 0$, $R_0 = 1.89$), le pic zombie est atteint dès le premier jour, puis la population zombie décroît lentement — mais ne disparaît jamais complètement. Augmenter progressivement le taux de quarantaine abaisse $R_0$ et réduit le pic, mais c'est seulement à partir de $\kappa = \sigma = 0.03$ que $R_0$ passe sous 1 et que les zombies tendent vers zéro à long terme. En dessous de ce seuil, la quarantaine ralentit sans éradiquer.

**Panneau droit — le bilan au jour 80.** La courbe rouge montre que $R_0$ décroît rapidement dès que la quarantaine s'intensifie — la ligne pointillée à $R_0 = 1$ matérialise le seuil critique. La courbe bleue montre la contrepartie : plus on met en quarantaine, plus il reste d'humains survivants au jour 80. Les deux courbes se croisent visuellement autour de $\kappa = \sigma \approx 0.02$, précisément là où $R_0 = 1$ — c'est la définition même du seuil épidémique.

> **Ce que ce graphique dit en une phrase** : il existe un taux de quarantaine minimal en dessous duquel aucun effort ne suffit — exactement comme le seuil d'immunité collective pour un vaccin.

## Ce que les zombies nous apprennent vraiment

La conclusion de Munz et al. est sans appel : face aux zombies, seule une intervention massive et rapide peut éviter le scénario catastrophe. La quarantaine ralentit, mais comme on vient de le voir sur les figures, elle ne suffit pas si elle arrive trop tard ou reste trop partielle.

Mais le parallèle le plus frappant avec le COVID concerne le traitement. Munz et al. explorent un scénario où un traitement permet de reconvertir un zombie en humain. Résultat : la coexistence devient possible — mais à des niveaux de population humaine très bas, et de façon instable. Pourquoi ? Parce que **le traitement ne confère aucune immunité** : un humain guéri retombe immédiatement dans le compartiment $S$, et peut être remordu le lendemain.

C'est exactement la différence entre un **antiviral** et un **vaccin** dans une vraie épidémie. L'antiviral guérit l'individu mais ne protège pas la population. Le vaccin, lui, déplace définitivement des individus du compartiment $S$ vers un compartiment immunisé, hors d'atteinte du virus. C'est ce déplacement irréversible qui crée l'immunité collective et fait s'effondrer $R_0$ — non pas parce que le virus est moins transmissible, mais parce que la population
susceptible se raréfie structurellement.

Les zombies, finalement, ne sont qu'un prétexte commode — mais remarquablement efficace — pour comprendre pourquoi le COVID n'a pas été maîtrisé par les antiviraux seuls, et pourquoi la vaccination de masse était la seule issue mathématiquement robuste.

---

*Le notebook complet, avec une interface interactive pour explorer les paramètres, est
disponible sur [un repo GitHub](#).*

**Références**

- Munz, P., Hudea, I., Imad, J., & Smith?, R. J. (2009). *When Zombies Attack! : Mathematical
  Modelling of an Outbreak of Zombie Infection*. In J. M. Tchuenche & C. Chiyaka (Eds.),
  Infectious Disease Modelling Research Progress (pp. 133–150). Nova Science Publishers.
- van den Driessche, P., & Watmough, J. (2002). Reproduction numbers and sub-threshold endemic
  equilibria for compartmental models of disease transmission. *Mathematical Biosciences*,
  180(1–2), 29–48. https://doi.org/10.1016/S0025-5564(02)00108-6


**Notes**