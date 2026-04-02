---
layout: post
title: "TRIMP & Co — quantifier la charge d'entraînement"
date: 2026-03-28
description: >
  Petite revue du TRIMP de Banister, du modèle fitness-fatigue de Morton et al.,
  et de la déclinaison CTL/ATL/TSB de Coggan. Quelles limites de ces approches ?
tags: trail entraînement charge modélisation fréquence-cardiaque
categories: trail
related_posts: true
toc:
  sidebar: left
math: true
---

Après une sortie, ta montre te donne un score de « charge ». Sur Garmin c'est l'*acute load*, sur Suunto le *Training Load*... Derrière ces étiquettes "marketing" se cache — souvent — le même modèle, vieux de cinquante ans : le modèle **TRIMP** de Banister. 
Toujours avec cette envie de comprendre mes données, je me suis dit que j'allais explorer de ce côté.

Je ne saurai que trop te conseiller de lire/écouter le travail de Cyril Forester du Podcast Courir Mieux sur la charge d'entrainement : [Comprendre et utiliser la charge d’entraînement en trail](https://courir-mieux.fr/charge-dentrainement-trail)

---

## La charge d'entrainement : pourquoi la quantifier ?

Bon, quand tu mets tes baskets (mais c'est vrai pour n'importe quel sport), ton entraînement génère **un stress biologique** volontairement appliqué. 
- Trop peu : aucune adaptation. 
- Trop : surmenage, blessure, ... 
- La *zone productive* se situe entre les deux. Elle est étroite et variable. 

Il nous faut donc un indicateur de la **dose totale** absorbée — pas seulement la durée, pas seulement l'intensité, mais la combinaison des deux pour savoir où se situer.

On distingue deux types de charge :

- **Charge externe** : ce que tu fais — distance, dénivelé, vitesse. $\rightarrow$ Indépendante de ton état physiologique du jour.
- **Charge interne** : ce que ton organisme ressent — réponse cardiaque, lactatémie, perturbation hormonale. $\rightarrow$ C'est elle qui génère l'adaptation.

Et bien le TRIMP vise justement la charge interne en s'appuyant sur la fréquence cardiaque. C'est un proxy accessible de l'effort physiologique.

---

## Le TRIMP de Banister (1975 / 1991)

Pour la suite, on va se baser sur ces deux articles : 

- Banister et al. (1975). A system model of training for athletic performance. Aust. J. Sports Med., 7, 57–61. 
- Banister (1991). Modeling Elite Athletic Performance. In : Physiological Testing of Elite Athletes. Human Kinetics.


L'affaire commence en 1975. Eric Banister propose de quantifier chaque séance par un **Training IMPulse** :

$$\mathrm{TRIMP} = D \times \Delta\mathrm{HR} \times y$$

Les trois termes :

- **$D$** — durée de la séance en minutes.

- **$\Delta\mathrm{HR}$** — fraction de la réserve cardiaque utilisée (*Heart Rate Reserve*) :

$$\Delta\mathrm{HR} = \frac{\overline{\mathrm{HR}} - \mathrm{HR_{rest}}}{\mathrm{HR_{max}} - \mathrm{HR_{rest}}}$$

- **$y$** — facteur de pondération exponentiel, calé sur la relation HR–lactatémie
observée lors d'un test incrémental :

$$y = a \cdot e^{\,b \cdot \Delta\mathrm{HR}}$$

Banister donne deux jeux de constantes selon le sexe (Banister, 1991, cité dans Morton et al., 1990) :

| Sexe   | $a$  | $b$  |
|--------|------|------|
| Hommes | 0,64 | 1,92 |
| Femmes | 0,86 | 1,67 |

La non-linéarité de $y$ est l'idée forte du modèle : passer de 60 % à 80 % de réserve cardiaque coûte proportionnellement beaucoup plus cher que passer de 40 % à 60 %. Une heure à 80 % FCréserve produit ainsi bien plus de TRIMP qu'une heure à 60 %, conformément à ce qu'on observe sur la lactatémie.

**Petit graphique**

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/2025-12_analyse_data_trail/courbe_Banister.png"
       width="80%"
       alt="Modèle de Banister pour homme et femme">
  <figcaption>
    Modèle de Banister pour homme et femme et exemple de charge sur une séance de 60 min.
  </figcaption>
</figure>


**Exemple numérique.**  
Athlète masculin : HR$_\text{rest}$ = 45 bpm, HR$_\text{max}$ = 190 bpm.

| Séance | Durée | FC moy. | $\Delta$HR | $y$ | TRIMP |
|--------|-------|---------|-----------|-----|-------|
| Sortie longue | 90 min | 140 bpm | 0,66 | 1,81 | ~107 |
| Allure seuil  | 45 min | 170 bpm | 0,86 | 2,94 | ~113 |

Ca dit quoi ? Que deux séances très différentes, peuvent avoir une charge similaire — et pourtant l'une dure deux fois plus longtemps. C'est déjà plus juste que de compter les kilomètres.

---

## Le modèle fitness-fatigue (Morton, Fitz-Clarke & Banister, 1990)

La vraie innovation de l'équipe de Banister ne se limite pas au TRIMP comme score
de séance. Elle propose un **modèle dynamique** : chaque dose d'entraînement $w(t)$
produit simultanément deux réponses à décroissance exponentielle.

La composante positive (*fitness*) :

$$g(t) = g(t-1)\,e^{-1/\tau_1} + w(t)\left(1 - e^{-1/\tau_1}\right)$$

La composante négative (*fatigue*) :

$$h(t) = h(t-1)\,e^{-1/\tau_2} + w(t)\left(1 - e^{-1/\tau_2}\right)$$

Et la performance prédite :

$$\hat{p}(t) = p_0 + k_1\,g(t) - k_2\,h(t)$$

Les quatre paramètres du modèle ont une interprétation physiologique directe :

- **$\tau_1$** : constante de temps de la forme (combien de jours met-elle à disparaître sans entraînement). Morton et al. (1990) trouvent $\tau_1 \approx 49–50$ jours.
- **$\tau_2$** : idem pour la fatigue, bien plus courte : $\tau_2 \approx 11$ jours.
- **$k_1 < k_2$** : la fatigue est d'amplitude immédiate supérieure à la forme. 

L'entraînement dégrade d'abord avant d'améliorer — ce qui correspond à l'expérience de tout coureur.

De fait, l'idée du fameux **pic de forme** ressort naturellement du modèle : en réduisant $w(t)$ (affûtage), $h(t)$ chute vite ($\tau_2 = 11$ j) pendant que $g(t)$ persiste
($\tau_1 = 49$ j). Il existe donc un moment où la performance prédite est maximale.

---

## La déclinaison de Coggan : CTL, ATL, TSB

Andy Coggan simplifie le modèle en supprimant les gains $k_1, k_2$ et en reformulant les deux composantes comme des **moyennes exponentielles pondérées** directement lisibles :

$$\mathrm{CTL}(t) = \mathrm{CTL}(t-1)\,e^{-1/42} + \mathrm{TSS}(t)\left(1 - e^{-1/42}\right)$$

$$\mathrm{ATL}(t) = \mathrm{ATL}(t-1)\,e^{-1/7} + \mathrm{TSS}(t)\left(1 - e^{-1/7}\right)$$

$$\mathrm{TSB}(t) = \mathrm{CTL}(t) - \mathrm{ATL}(t)$$

Le **TSS** (*Training Stress Score*) remplace le TRIMP : il est normalisé de sorte qu'une heure à la puissance seuil (ou à l'allure seuil) vaille 100 points, ce qui le rend comparable entre athlètes.

Les trois courbes du *Performance Management Chart* résument l'état de l'athlète :

- **CTL** (*Chronic Training Load*) : la forme de fond, construite sur 42 jours. Elle monte lentement, descend lentement. C'est le « capital forme » accumulé.
- **ATL** (*Acute Training Load*) : la fatigue récente, sur 7 jours. Très réactive.
- **TSB** (*Training Stress Balance*) : la « fraîcheur ». Positif = reposé, négatif = chargé.


C'est typiquement les courbes que vous retrouvées dans l'appli Nolio quand vous cliquez sur "Voir les stats" ou dans "Suivi $\rightarrow$ Statistiques" : 

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/2025-12_analyse_data_trail/screenshot_Coggan.png"
       width="80%"
       alt="Exemple de charge d'entrainement selon Coggan">
  <figcaption>
    Exemple de charge d'entrainement selon Coggan.
  </figcaption>
</figure>


**Application pratique pour la planification.**  
Allen & Coggan (2010) et Friel (cité dans TrainingPeaks) proposent les repères suivants :

| TSB | Interprétation |
|-----|---------------|
| +15 à +25 | Pic de forme optimal pour une compétition |
| 0 à +15   | Bien récupéré, prêt à s'entraîner dur |
| −10 à −30 | Zone productive d'entraînement |
| < −30     | Surcharge — risque d'overreaching |

Pour un ultra, l'objectif est d'arriver avec un TSB légèrement positif (form) sans avoir trop sacrifié de CTL (fitness). 

**L'affûtage idéal cherche à maximiser CTL − ATL à la date de course.**

⚠️ NOTE : Il faudra que j'ajoute une section sur la charge d'entrainement selon FOSTER. 


## Évolutions : Edwards (1993) et Lucia et al. (2003)

Le TRIMP de Banister requiert la FC moyenne de la séance. Deux alternatives proposent une approche par zones.

**Edwards (1993)** décompose la séance en cinq zones de 10 % FCmax et les pondère linéairement :

$$\mathrm{TRIMP_{Edwards}} = \sum_{z=1}^{5} t_z \times c_z$$

avec $c_z \in \{1, 2, 3, 4, 5\}$ selon la zone. Simple à calculer, mais les seuils (50–60 %, 60–70 %, etc.) et les coefficients sont **arbitraires** — aucun ancrage physiologique validé. Rien ne prouve que la zone 5 représente cinq fois le stress de la zone 1.


**Lucia et al. (2003)** ancrent les zones sur les **seuils ventilatoires** (SV1, SV2), délimitations physiologiquement fondées :

$$\mathrm{TRIMP_{Lucia}} = t_1 + 2\,t_2 + 3\,t_3$$

C'est une amélioration réelle car SV1 et SV2 délimitent des domaines métaboliques distincts. En revanche, les coefficients 1/2/3 restent arbitraires — et les seuils ventilatoires nécessitent un test en laboratoire, ce qui limite l'accessibilité.


## Limites du modèle

### La FC moyenne reste aveugle aux intervalles

C'est la limite la plus documentée. 
- Deux séances d'une heure avec la même FC moyenne produisent le même TRIMP. Pourtant, une séance continue à 145 bpm et une séance de 6 × 5 min à 175 bpm / récup 5 min à 115 bpm ne sollicitent pas les mêmes systèmes.
- La séance en intervalles perturbe davantage le lactate, le système neuromusculaire et la cinétique de VO₂. Le TRIMP intégré sur la FC instantanée corrige partiellement ce biais. 

<!-- c'est ce que font certains appareils Firstbeat — mais la FC elle-même présente des limites à haute intensité (inertie, drift thermique).-->


### Les coefficients $y$ ne sont pas individualisés

Les constantes $a$ et $b$ de Banister ont été calées sur un petit échantillon et ne distinguent que le sexe. Manzi et al. ont proposé un **iTRIMP** (*individualized TRIMP*) qui ajuste le facteur de pondération sur la relation FC–vitesse propre à chaque athlète. Il montre une meilleure corrélation dose-réponse en course à pied et en cyclisme que le TRIMP classique.

### La nature de l'effort n'est pas prise en compte

Courir 2 heures en descente sur terrain technique, avec un fort stress excentrique et neuromusculaire, peut produire un TRIMP modéré alors que la charge réelle est élevée. Idem pour les séances de côtes courtes, la force-résistance, ou la chaleur. 
La charge externe et la charge interne divergent, et la FC seule ne capture pas tout.

### Les constantes temporelles sont fixes, pas validées individuellement

42 jours pour le CTL, 7 jours pour l'ATL : ce sont des valeurs par défaut, pas des mesures physiologiques. Coggan lui-même précise que ces constantes varient selon l'athlète, l'âge, le type de charge, et recommande de les ajuster sur l'historique personnel dès que suffisamment de données sont disponibles.


### Le modèle ne prédit pas la performance absolue

CTL et TSB sont des indicateurs **relatifs** propres à chaque athlète.
Un CTL de 80 pour un coureur récréatif représente une charge très différente d'un CTL de 80 pour un athlète de niveau national. Le modèle de Banister, dans sa version complète avec les quatre paramètres $k_1, k_2, \tau_1, \tau_2$, nécessite des mesures de performance terrain régulières pour être calé — ce qui est rarement fait en pratique.

---

## Aller plus loin : l'approche data-driven

Les modèles précédents ont des **paramètres fixés** *a priori*. Comme je l'indiquais dans l'article sur Minetti, l'approche data-driven consisterai à les **estimer sur tes propres données**, en observant comment ta performance évolue en réponse à la charge.

**Ajustement des paramètres par optimisation.** Si tu mesures régulièrement ta performance sur un test terrain reproductible (chrono sur boucle fixe, VMA, Cooper, allure à FC cible), il est possible d'ajuster $k_1, k_2, \tau_1, \tau_2$ par minimisation de l'erreur entre le modèle et tes mesures réelles. 

**Estimation locale de $\tau$.**  Une période de repos prolongé (repos forcé, coupure de saison) est une opportunité : en observant la décroissance de ton CTL ou de tes performances sur cette période, on peut ajuster une exponentielle et estimer ton $\tau_1$ personnel. Si ta forme chute beaucoup plus vite que les 42 jours standards, c'est que ta constante de temps est plus courte — et que ton affûtage doit être moins long.

**Modèles non-linéaires et ML.** Des travaux récents explorent des variantes du modèle de Banister avec des paramètres variants dans le temps (Busso, 2003), ou des approches de machine learning (réseaux récurrents, Kalman filter) pour modéliser la relation charge-performance. Ces approches sont séduisantes mais nécessitent des volumes de données importants. On y reviendra..

---

## Références

- Banister EW, Calvert TW, Savage MV, Bach T (1975). *A system model of training 
  for athletic performance*. Aust. J. Sports Med., 7, 57–61.
- Banister EW (1991). *Modeling Elite Athletic Performance*. In : MacDougall JD, Wenger HA, Green HJ (eds.), Physiological Testing of Elite Athletes. Human Kinetics.
- Morton RH, Fitz-Clarke JR, Banister EW (1990). *Modeling human performance in running*. J. Appl. Physiol., 69(3), 1171–1177. 
  DOI: [10.1152/jappl.1990.69.3.1171](https://doi.org/10.1152/jappl.1990.69.3.1171)
- Allen H, Coggan A (2010). *Training and Racing with a Power Meter*. VeloPress.
- Coggan A (2023). *The Science of the TrainingPeaks Performance Manager*. [trainingpeaks.com](https://www.trainingpeaks.com/learn/articles/the-science-of-the-performance-manager/)
- Edwards S (1993). *The Heart Rate Monitor Book*. Polar Electro Oy.
- Lucia A, Hoyos J, Santalla A, Earnest C, Chicharro JL (2003). *Tour de France versus Vuelta a España: which is harder?* Med. Sci. Sports Exerc., 35(5), 872–878.
  DOI: [10.1249/01.MSS.0000064999.82036.B4](https://doi.org/10.1249/01.MSS.0000064999.82036.B4)
- Desgorces FD, Sénégas X, Garcia J, Decker L, Noirez P (2007). *Methods to quantify intermittent exercises*. Appl. Physiol. Nutr. Metab., 32(4), 762–769. DOI: [10.1139/H07-037](https://doi.org/10.1139/H07-037)
- Busso T (2003). *Variable dose-response relationship between exercise training and performance*. Med. Sci. Sports Exerc., 35(7), 1188–1195. DOI: [10.1249/01.MSS.0000074465.13621.37](https://doi.org/10.1249/01.MSS.0000074465.13621.37)

## Petit warning 

> Avertissement : je suis datas cientist, pas spécialiste 
> de physiologie de l'exercice. Ce que tu lis ici, c'est 
> le carnet de bord d'un trailer curieux qui aime comprendre 
> ses données — pas un cours magistral. Les sources sont là 
> pour que tu puisses vérifier.