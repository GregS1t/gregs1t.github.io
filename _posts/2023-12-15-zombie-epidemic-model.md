---
layout: post
title: "Quand les maths rencontrent les zombies !"
date: 2023-12-15
description: >
  Un week-end pluvieux, une série à regarder, et une question qui s'impose : et si on modélisait ça ?.
tags: [EDO, simulation, zombies, last of us, epidemiology]
categories: simulation
series_order: 1
related_posts: true
toc:
  sidebar: left
math: true

---

C'est en regardant *The Last of Us* que l'idée m'est venue. La série, adaptée du jeu vidéo, met en scène une pandémie fongique qui transforme progressivement les humains en créatures agressives, des zombies quoi ! En parallèle, le souvenir du COVID-19 était encore frais — cette période étrange où chacun suivait des courbes, des R₀, des taux de reproduction. On était tous agrégés d'épidémiologie. 
Et là, entre deux épisodes, la question s'est posée naturellement : *est-ce qu'on peut modéliser une épidémie de zombies de la même façon qu'une vraie épidémie ?*

La réponse est oui. Et quelqu'un l'a fait avant moi.

## L'article qui a tout déclenché

En 2009, Munz, Hudea, Imad et Smith ont publié un article intitulé *"When Zombies Attack! : Mathematical Modelling of an Outbreak of Zombie Infection"* dans *Infectious Disease Modelling Research Progress*. L'article est sérieux, peer-reviewed, et utilise des outils épidémiologiques standard. C'est cette base que j'ai reprise pour coder les modèles en Python, histoire de passer un bon moment et de réviser quelques notions au passage.

## Les modèles SIR, et pourquoi les zombies compliquent tout

En épidémiologie classique, on découpe la population en compartiments. Le modèle de base — le SIR — distingue les **Susceptibles** (S), les **Infectés** (I) et les **Retirés** (R, c'est-à-dire guéris ou décédés). Les équations qui gouvernent l'évolution de ces trois populations forment un **système d'équations différentielles ordinaires (EDO) couplées** :

$$\frac{dS}{dt} = -\beta SI, \quad \frac{dI}{dt} = \beta SI - \gamma I, \quad \frac{dR}{dt} = \gamma I$$

Le terme $\beta SI$ est dit de *masse d'action* : il capture le fait que les rencontres entre susceptibles et infectés sont proportionnelles aux deux populations. Le paramètre $\beta$ est le taux de transmission, $\gamma$ le taux de guérison.

Ce qui rend les zombies particulièrement "embêtants", c'est que **les morts peuvent revenir**. Ça introduit un flux supplémentaire depuis les retirés vers les zombies, *via* un taux de résurrection $\zeta$. Le modèle de base de Munz et al. (noté SZR) s'écrit :

$$\frac{dS}{dt} = \Pi - \beta SZ - \delta S$$

$$\frac{dZ}{dt} = \beta SZ + \zeta R - \alpha SZ$$

$$\frac{dR}{dt} = \delta S + \alpha SZ - \zeta R$$

où $\Pi$ est le taux de natalité, $\delta$ la mortalité naturelle, $\alpha$ le taux de destruction des zombies par les humains. L'analyse des équilibres montre que **la coexistence humains-zombies est impossible** : soit les zombies sont éradiqués, soit ils nous envahissent tous. Pas de milieu.

## Vers un modèle plus réaliste : le SIZRQ

Munz et al. proposent ensuite plusieurs raffinements. Celui que j'ai trouvé le plus intéressant est le modèle **SIZRQ**, qui ajoute deux compartiments :

- **I** (Infectés) : une classe de personnes mordues mais pas encore zombifiées — la période d'incubation observée dans la plupart des œuvres de fiction, *The Last of Us* inclus.
- **Q** (Quarantaine) : les infectés et zombies mis à l'écart de la population générale.

Le système devient alors :

$$\frac{dS}{dt} = \Pi - \beta SZ - \delta S$$

$$\frac{dI}{dt} = \beta SZ - \rho I - \delta I - \kappa I$$

$$\frac{dZ}{dt} = \rho I + \zeta R - \alpha SZ - \sigma Z$$

$$\frac{dR}{dt} = \delta S + \delta I + \alpha SZ - \zeta R + \gamma Q$$

$$\frac{dQ}{dt} = \kappa I + \sigma Z - \gamma Q$$

Ici $\rho$ est le taux de passage de I vers Z (la vitesse de zombification), $\kappa$ et $\sigma$ les taux de mise en quarantaine des infectés et des zombies, $\gamma$ le taux d'évasion (ceux qui s'échappent, et sont éliminés avant de rejoindre Z).

## Le taux de reproduction de base $R_0$

Pour le modèle SIZRQ, le **taux de reproduction de base** $R_0$ peut être calculé analytiquement via la méthode de la matrice de prochaine génération (van den Driessche & Watmough, 2002). Le résultat est :

$$R_0 = \frac{\rho \beta N}{(\rho + \delta + \kappa)(\alpha N + \sigma)}$$

Si $R_0 > 1$, l'épidémie persiste. Si $R_0 < 1$, elle s'éteint. La quarantaine agit en augmentant $\kappa$ et $\sigma$, ce qui réduit $R_0$ — exactement comme le confinement pendant le COVID cherchait à abaisser $R_e$ sous 1.

## Résoudre numériquement en Python

L'intégration numérique de ce type de système est directe avec `scipy`. L'essentiel est de choisir un bon intégrateur — Euler (utilisé dans l'article original en MATLAB) est simple mais instable sur des intervalles longs. On lui préfère **RK45** :

```python
from scipy.integrate import solve_ivp
import numpy as np

def model_SIZRQ(t, y, Pi, delta, beta, zeta, alpha, rho, kappa, sigma, gamma):
    Si, Ii, Zi, Ri, Qi = y
    dS_dt = Pi - beta*Si*Zi - delta*Si
    dI_dt = beta*Si*Zi - rho*Ii - delta*Ii - kappa*Ii
    dZ_dt = rho*Ii + zeta*Ri - alpha*Si*Zi - sigma*Zi
    dR_dt = delta*Si + delta*Ii + alpha*Si*Zi - zeta*Ri + gamma*Qi
    dQ_dt = kappa*Ii + sigma*Zi - gamma*Qi
    return [dS_dt, dI_dt, dZ_dt, dR_dt, dQ_dt]

S0, Z0 = 500.0, 2.0
y0 = [S0, 0.0, Z0, 0.01*S0, 0.0]
t = np.linspace(0, 30, 5000)

sol = solve_ivp(
    model_SIZRQ, [t[0], t[-1]], y0,
    args=(0.0, 0.0001, 0.0095, 0.0001, 0.005, 0.025, 0.01, 0.01, 0.01),
    t_eval=t, method='RK45'
)
```

 ## Un graphique ou deux

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/Zombie_article/sizrq_quarantine.png"
       width="80%"
       alt="Impact de la quarantaine sur le pic de zombies.">
  <figcaption>
    Figure 1 - Impact de la quarantaine sur le pic de zombies.
  </figcaption>
</figure>


## Ce que ça dit sur les vraies épidémies

La conclusion de Munz et al. est sans appel : face aux zombies, seule une attaque massive et rapide peut éviter le scénario catastrophe. La quarantaine ralentit, mais ne suffit pas. Un traitement (retour des zombies à l'état humain) permet la coexistence, mais à des niveaux de population très bas.

Ce qui est frappant, c'est que ces conclusions résonnent avec ce qu'on a observé pendant le COVID. La quarantaine partielle réduit $R_0$ mais ne l'amène pas forcément sous 1. Un vaccin efficace (analogue au traitement zombie) change fondamentalement la dynamique en créant une immunité durable — contrairement au traitement zombie qui ne confère aucune immunité, condamnant les humains guéris à être à nouveau susceptibles.

Les zombies, finalement, ne sont qu'un prétexte commode pour explorer des outils mathématiques bien réels. Les systèmes d'EDO couplées, les équilibres et leur stabilité, le calcul de $R_0$ par matrice de prochaine génération — tout ça s'applique sans modification aux maladies infectieuses, aux dynamiques de population, ou encore à la propagation de rumeurs sur les réseaux sociaux.


---

*Le notebook complet une interface interactive pour explorer les paramètres, est disponible sur [un repo GitHub](#).*

**Références**

- Munz, P., Hudea, I., Imad, J., & Smith?, R. J. (2009). *When Zombies Attack! : Mathematical Modelling of an Outbreak of Zombie Infection*. In J. M. Tchuenche & C. Chiyaka (Eds.), Infectious Disease Modelling Research Progress (pp. 133–150). Nova Science Publishers.

