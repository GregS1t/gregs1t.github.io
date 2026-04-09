---
layout: post
title: Quelle heure est-il sur Mars ?
date: 2020-11-22
description: "Comment convertir une date terrestre en heure martienne ? De la longitude julienne au LMST, en passant par l'anomalie moyenne et l'équation du temps."
tags: [astronomie, Mars, temps, planétologie, maths, SEIS, Insight, NASA]
categories: [planetology]
related_posts: false
toc:
  sidebar: left
---

> **MarsTimeConverter** — Cet article présente les équations implémentées dans la bibliothèque Python [MarsTimeConverter](https://github.com/GregS1t/marstimeconverter), développée à l'Institut de physique du Globe dans le cadre de la mission InSight.
> Les références aux équations notées « AM2000 » renvoient à l'algorithme Mars24 de Allison & McEwen (2000) tel que documenté par la NASA/GISS.

---

## 1. Pourquoi Mars a-t-elle besoin de sa propre horloge ?

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/2020-11-22_mars_time_converter/marsInsight.png"
       width="50%"
       alt="Lander Insight">
</figure>


Avant même que la sonde InSight ne se pose sur Mars le 26 novembre 2018 sur Elisyum Planitia, les équipes scientifiques ont immédiatement eu besoin de répondre à une question simple : *à quelle heure locale est-ce que le soleil se lève sur le site d'atterrissage ?*

Sur Terre, cette question semble triviale. Sur Mars, elle est plus subtile, pour deux raisons principales.

**Un jour plus long.** Une journée martienne — appelée **SOL** — dure environ 88 775 secondes, soit 24 h 39 min 35 s. C'est 2,7 % de plus qu'une journée terrestre. Cette différence, qui paraît petite, s'accumule : au bout d'un an martien (669 sols), le décalage avec le calendrier terrestre atteint plusieurs semaines.

**Une orbite elliptique et inclinée.** L'orbite de Mars autour du Soleil est plus excentrique que celle de la Terre (excentricité $e \approx 0.093$ contre $e \approx 0.017$ pour la Terre). Cela signifie que Mars se déplace plus vite près du périhélie que près de l'aphélie, ce qui introduit un écart entre l'heure **solaire vraie** (basée sur la position réelle du Soleil) et l'heure **solaire moyenne** (basée sur un Soleil fictif se déplaçant uniformément). Cet écart s'appelle l'**équation du temps**.

**Ce problème n'est pas nouveau.** Dès les premières missions martiennes, les équipes d'opérations ont eu besoin d'un système de temps local standardisé. La référence qui fait autorité aujourd'hui est l'article d'Allison & McEwen (2000), *A post-Pathfinder evaluation of areocentric solar coordinates with improved timing recipes for Mars seasonal/diurnal climate studies*, publié dans *Planetary and Space Science*. Cet article fournit une chaîne de calcul complète et rigoureuse, qui est depuis adoptée par la NASA/GISS et reprise par l'ensemble de la communauté planétaire. C'est directement cet algorithme — équation par équation — qui a servi de base à la bibliothèque **MarsTimeConverter**. Les équations de cet article seront notées « AM2000 » dans la suite.

L'objectif de cet article est de suivre pas à pas cette chaîne de calcul.

---

## 2. Le point de départ : la date julienne

Toute la chaîne de conversion commence par exprimer la date terrestre dans un référentiel continu et universel : le **jour julien** (Julian Day, JD).

Le jour julien est un comptage continu de jours depuis le 1er janvier 4713 av. J.-C. à midi (temps universel). C'est l'unité de référence en astronomie car elle évite les ambiguïtés des calendriers (années bissextiles, changements d'ère, fuseaux horaires).

La conversion depuis une date UTC est donnée par l'équation **A-2** de l'algorithme Mars24 :

$$
JD_\text{UTC} = 2440587.5 + \frac{t_\text{Unix}}{86400}
$$

où $t_\text{Unix}$ est le nombre de secondes écoulées depuis le 1er janvier 1970 à 00:00:00 UTC (l'époque Unix). La constante $2440587.5$ est le jour julien correspondant à cette époque.

> **Exemple numérique.** Le 1er janvier 2000 à 12:00:00 UTC correspond à $JD_\text{UTC} = 2451545.0$. C'est l'**époque J2000**, point de référence de nombreux calculs célestes.

---

## 3. Corriger le temps : de l'UTC au Temps Terrestre

L'UTC (Coordinated Universal Time) n'est pas exactement uniforme : il est maintenu synchronisé avec la rotation de la Terre par l'ajout occasionnel de **secondes intercalaires** (*leap seconds*). Pour les calculs orbitaux, on préfère utiliser le **Temps Terrestre** (TT, *Terrestrial Time*), qui est une échelle de temps strictement uniforme.

Le décalage entre UTC et TT vaut :

$$
\Delta_{UTC \to TT} = 32.184 + \Delta_{AT} \quad \text{(secondes)}
$$

où 32.184 s est la différence historique entre l'ancienne échelle ET (*Ephemeris Time*) et l'UTC, et $\Delta_{AT}$ est le nombre de secondes intercalaires accumulées depuis 1972 (37 s en 2026). Ce décalage est tabulé et vaut en pratique environ 69.2 s aujourd'hui.

On obtient ainsi le jour julien en temps terrestre :

$$
JD_\text{TT} = JD_\text{UTC} + \frac{\Delta_{UTC \to TT}}{86400} \quad \text{(éq. A-5)}
$$

---

## 4. L'offset depuis J2000

Pour simplifier les formules orbitales, on exprime le temps non pas en jours juliens absolus, mais en **offset depuis J2000** :

$$
\Delta t_{J2000} = JD_\text{TT} - 2451545.0 \quad \text{(éq. A-6)}
$$

$\Delta t_{J2000}$ est donc un simple nombre de jours (positif après le 1er janvier 2000 à midi TT, négatif avant). Toutes les formules orbitales martiennes qui suivent sont exprimées en fonction de cette variable.

---

## 5. La position de Mars dans son orbite

### 5.1 Anomalie moyenne $M$

L'**anomalie moyenne** $M$ est l'angle qu'aurait parcouru Mars si son orbite était parfaitement circulaire, à une vitesse angulaire constante. Elle augmente linéairement avec le temps :

$$
M = 19.3871 + 0.52402073 \cdot \Delta t_{J2000} \pmod{360°} \quad \text{(AM2000, éq. 16)}
$$

Le coefficient $0.52402073$ °/jour correspond au mouvement orbital moyen de Mars (une révolution de 360° en environ 686.97 jours terrestres).

### 5.2 Angle du Soleil Moyen Fictif $\alpha_\text{FMS}$

L'angle du **Soleil Moyen Fictif** (*Fictional Mean Sun*, FMS) est défini comme :

$$
\alpha_\text{FMS} = 270.3871 + 0.524038496 \cdot \Delta t_{J2000} \pmod{360°} \quad \text{(Mars24/NASA GISS)}
$$

Il représente la position d'un Soleil fictif se déplaçant uniformément sur l'orbite martienne, et sert de référence pour définir l'heure solaire moyenne.

> **Note.** Le papier AM2000 original donne la constante $270.3863°$. La valeur $270.3871°$ est la correction adoptée par la NASA/GISS dans l'implémentation Mars24, publiée après détection d'une erreur dans l'article original. C'est cette valeur corrigée qu'implémente MarsTimeConverter.

### 5.3 Perturbations périodiques (PBS)

L'orbite réelle de Mars est perturbée par les autres planètes (principalement Jupiter). Ces perturbations sont modélisées par une somme de sept termes cosinus :

$$
\text{PBS} = \sum_{i=1}^{7} A_i \cos\!\left( \frac{0.985626 \cdot \Delta t_{J2000}}{\tau_i} + \phi_i \right) \quad \text{(AM2000, éq. 18)}
$$

Les coefficients $(A_i, \tau_i, \phi_i)$ sont les suivants :

| $i$ | $A_i$ | $\tau_i$ | $\phi_i$ (°) |
|-----|-------|----------|--------------|
| 1   | 0.0071 | 2.2353  | 49.409       |
| 2   | 0.0057 | 2.7543  | 168.173      |
| 3   | 0.0039 | 1.1177  | 191.837      |
| 4   | 0.0037 | 15.7866 | 21.736       |
| 5   | 0.0021 | 2.1354  | 15.704       |
| 6   | 0.0020 | 2.4694  | 95.528       |
| 7   | 0.0018 | 32.8493 | 49.095       |

L'amplitude totale du PBS reste inférieure à 0.02°, ce qui correspond à un décalage horaire de moins d'une minute. C'est négligeable pour la vie quotidienne, mais important pour la précision des éphémérides.

### 5.4 Équation du centre (EOC)

L'**équation du centre** $\nu - M$ traduit l'écart entre la position angulaire réelle de Mars (anomalie vraie $\nu$) et sa position fictive à vitesse constante (anomalie moyenne $M$). Pour une orbite elliptique, elle se développe en série de Fourier :

$$
\begin{aligned}
\nu - M \approx\; & (10.691 + 3\times10^{-7}\cdot\Delta t_{J2000})\,\sin M \\
                  & + 0.623\,\sin 2M + 0.050\,\sin 3M \\
                  & + 0.005\,\sin 4M + 0.0005\,\sin 5M \\
                  & + \text{PBS}
\end{aligned}
\quad \text{(AM2000, éq. B-4)}
$$

Le terme dominant, $\approx 10.7° \sin M$, est directement lié à l'excentricité de l'orbite ($\nu - M \approx 2e \sin M$ au premier ordre, avec $e \approx 0.093$).

### 5.5 Longitude solaire aréocentrique $L_s$

La **longitude solaire** $L_s$ est l'angle entre la direction Mars–Soleil et la direction du point vernal martien (l'équinoxe de printemps de l'hémisphère nord). C'est le calendrier des saisons martiennes :

- $L_s = 0°$ : printemps boréal (équinoxe de printemps)  
- $L_s = 90°$ : été boréal (solstice d'été)  
- $L_s = 180°$ : automne boréal (équinoxe d'automne)  
- $L_s = 270°$ : hiver boréal (solstice d'hiver), proche du périhélie

$$
L_s = (\alpha_\text{FMS} + \nu - M) \pmod{360°} \quad \text{(AM2000, éq. B-5)}
$$

---

## 6. Du temps orbital au temps martien

### 6.1 Le Mars Sol Date (MSD)

Le **Mars Sol Date** est l'équivalent martien du Jour Julien : un comptage continu de sols depuis une origine fixe (le 29 décembre 1873 à 12:00 TT, par convention). Il est défini par :

$$
\text{MSD} = \frac{\Delta t_{J2000} - 4.5}{1.0274912517} + 44796.0 - 0.0009626 \quad \text{(Mars24, éq. C-2)}
$$

Le diviseur $1.0274912517$ est le rapport entre la durée d'un sol martien et celle d'un jour terrestre (un sol dure 1.02749... jours terrestres). La constante 44796.0 cale le comptage à zéro à l'origine.

### 6.2 Temps Martien Coordonné (MTC)

Le **Temps Martien Coordonné** (MTC, *Mars Coordinated Time*) est l'équivalent martien de l'UTC : c'est l'heure solaire moyenne au **méridien de référence de Mars** (0° de longitude, correspondant au cratère Airy-0).

$$
\text{MTC} = 24 \times \left(\text{MSD} \bmod 1\right) \quad \text{(heures décimales)}
$$

Il donne l'heure qu'il serait à Paris si Paris était sur Mars au méridien zéro. C'est le temps de référence commun à toutes les missions martiennes.

### 6.3 L'équation du temps (EOT)

Tout comme sur Terre, l'heure solaire vraie (basée sur la position réelle du Soleil) diffère de l'heure solaire moyenne (basée sur un Soleil fictif uniforme). Cette différence est l'**équation du temps** :

$$
\text{EOT} = 2.861 \sin(2 L_s) - 0.071 \sin(4 L_s) + 0.002 \sin(6 L_s) - (\nu - M) \quad \text{(AM2000, éq. C-1)}
$$

L'EOT est exprimée en degrés (15° = 1 heure). Sur Mars, elle varie de -51 min à +40 min au cours d'une année, bien plus que sur Terre (±16 min), à cause de l'excentricité plus élevée.

---

## 7. L'heure locale : LMST et LTST

On dispose maintenant de tous les ingrédients pour calculer l'heure locale à un point donné de la surface martienne.

### 7.1 Local Mean Solar Time (LMST)

Le **LMST** (*Local Mean Solar Time*) est l'heure solaire moyenne locale, calculée à partir du MTC en ajoutant la correction de longitude :

$$
\text{LMST} = \text{MTC} + \frac{\lambda_E}{15} \pmod{24\,\text{h}}
$$

où $\lambda_E$ est la **longitude est** du site en degrés (positive vers l'est, négative vers l'ouest). Chaque degré de longitude correspond à $\frac{88775}{360} \approx 246.6$ secondes de sol.

> **Convention.** La documentation NASA/GISS Mars24 formule le LMST avec la longitude **ouest** $\Lambda_W$ : $\text{LMST} = \text{MST} - \Lambda_W/15$. Les deux expressions sont équivalentes puisque $\lambda_E = -\Lambda_W$. MarsTimeConverter utilise les longitudes est, conformément aux conventions des missions InSight et Curiosity.

En pratique, dans le code, la conversion utilise directement le nombre de secondes écoulées depuis une date d'origine propre à chaque mission. Pour InSight (Sol 1 = 27 novembre 2018 à 05:50:25.58 UTC) :

$$
\text{sol décimal} = \frac{t_\text{UTC} - t_\text{origine}}{\text{durée d'un sol}} + \text{sol\_ref}
$$

Le sol est ensuite décomposé en partie entière (numéro du sol) et partie fractionnaire (heure dans le sol), au format `SSSST HH:MM:SS`.

### 7.2 Local True Solar Time (LTST)

Le **LTST** (*Local True Solar Time*) est l'heure solaire vraie : c'est la position réelle du Soleil dans le ciel. C'est cette heure qui détermine si le Soleil est au zénith (midi vrai), à l'horizon est (lever) ou à l'horizon ouest (coucher).

Elle s'obtient en ajoutant l'équation du temps au LMST :

$$
\text{LTST} = \text{LMST} + \frac{\text{EOT}}{15°} \quad \text{(en heures)}
$$

La différence LTST − LMST peut atteindre ±50 minutes sur Mars. Pour les opérations scientifiques sensibles à l'heure locale (mesures atmosphériques, imagerie solaire), c'est le LTST qu'il faut utiliser.

---

## 8. Bonus : la déclinaison solaire et la position du Soleil

Pour aller plus loin et calculer l'élévation du Soleil au-dessus de l'horizon, il faut connaître la **déclinaison solaire** $\delta$ : l'angle entre le plan de l'équateur martien et la direction Mars–Soleil.

$$
\delta = \arcsin(0.42565 \cdot \sin L_s) + 0.25° \cdot \sin L_s \quad \text{(A1997, éq. 5)}
$$

La constante $0.42565$ correspond à $\sin(25.19°) \approx 0.4256$, où $25.19°$ est l'obliquité de l'axe de Mars (la Terre a une obliquité de $23.44°$). Le second terme, $0.25° \sin L_s$, est une correction empirique de petite amplitude.

L'**élévation solaire** locale s'obtient ensuite par la formule classique de trigonométrie sphérique :

$$
\sin(\text{élévation}) = \sin\delta \cdot \sin\phi + \cos\delta \cdot \cos\phi \cdot \cos H
$$

où $\phi$ est la latitude du site et $H = \lambda - \lambda_\odot$ est l'angle horaire (différence entre la longitude du site et la longitude sub-solaire $\lambda_\odot$, calculée à partir du MTC et de l'EOT).

---

## 9. Récapitulatif de la chaîne de calcul

| Étape | Quantité | Formule | Référence |
|-------|----------|---------|-----------|
| 1 | Jour Julien UTC ($JD_\text{UTC}$) | $2440587.5 + t_\text{Unix} / 86400$ | A-2 |
| 2 | Décalage UTC → TT | Tables de secondes intercalaires | A-3/A-4 |
| 3 | Jour Julien TT ($JD_\text{TT}$) | $JD_\text{UTC} + \Delta_{TT}/86400$ | A-5 |
| 4 | Offset J2000 ($\Delta t$) | $JD_\text{TT} - 2451545.0$ | A-6 |
| 5 | Anomalie moyenne ($M$) | $19.3871 + 0.52402073 \cdot \Delta t$ | B-1 |
| 6 | Angle FMS ($\alpha_\text{FMS}$) | $270.3871 + 0.524038496 \cdot \Delta t$ | B-2 |
| 7 | Perturbations (PBS) | Somme de 7 cosinus | B-3 |
| 8 | Équation du centre (EOC) | Série en $\sin(nM)$ + PBS | B-4 |
| 9 | Longitude solaire ($L_s$) | $\alpha_\text{FMS} + \text{EOC} \pmod{360}$ | B-5 |
| 10 | Mars Sol Date (MSD) | $(\Delta t - 4.5) / 1.0274912517 + 44796 - 0.0009626$ | C-2 |
| 11 | Équation du temps (EOT) | Série en $\sin(2nL_s) - \text{EOC}$ | C-1 |
| 12 | MTC | $24 \times (\text{MSD} \bmod 1)$ | C-3 |
| 13 | **LMST** | $\text{MTC} + \lambda/15$ | C-4 |
| 14 | **LTST** | $\text{LMST} + \text{EOT}/15$ | C-5 |

---



## 10. Généraliser à n'importe quelle mission

Toutes les équations présentées jusqu'ici (sections 2 à 8) sont **universelles** : elles ne dépendent d'aucune mission en particulier. Ce qui change d'une mission à l'autre, c'est uniquement un petit ensemble de paramètres géographiques et temporels propres au site d'atterrissage.

### 10.1 Les paramètres nécessaires

Pour calculer le LMST d'une mission donnée, il suffit de connaître :

- La **longitude est** du site ($\lambda_E$, en degrés), pour calculer le LMST depuis le MTC.
- La **latitude** du site ($\phi$, en degrés), pour calculer l'élévation solaire.
- La **date d'origine des sols** (`solorigin`) : la date UTC à laquelle commence le comptage des sols (Sol 0 ou Sol 1 selon la convention de la mission).
- La **référence de sol** (`ref`) : 0 ou 1, selon que la mission numérote le premier sol "Sol 0" (InSight, Perseverance, Curiosity) ou "Sol 1" (Spirit, Opportunity).

La durée d'un sol est, elle, une constante physique indépendante de la mission : 88 775.244 secondes.

### 10.2 Le fichier de configuration XML

MarsTimeConverter regroupe ces paramètres (difficilement collectés) dans un fichier XML, ce qui permet de passer d'une mission à l'autre sans modifier le code. En voici le contenu pour les missions actuellement supportées :

| Mission | Site | Longitude E (°) | Latitude (°) | Sol origine (UTC) | Sol ref | Statut |
|---|---|---|---|---|---|---|
| InSight | Elysium Planitia | 224.03 | +4.50 | 2018-11-26T05:10:50 | 0 | Terminée (Sol 1688) |
| Perseverance | Jezero Crater | 282.74 | +18.44 | 2021-02-18T04:24:16 | 0 | Active |
| Curiosity | Gale Crater | 137.40 | −4.59 | 2012-08-05T13:49:59 | 0 | Active |
| Opportunity | Meridiani Planum | 354.47 | −1.95 | 2004-01-24T15:09:00 | 1 | Terminée (Sol 5111) |
| Spirit | Gusev Crater | 175.47 | −14.57 | 2004-01-03T13:36:16 | 1 | Terminée (Sol 2208) |


*Remarque*: Une entrée `Mars` (méridien 0°, lat 0°) sert de référence pour calculer le MTC, indépendamment de tout site d'atterrissage.

## Références

1. Allison, M. & McEwen, M. (2000). *A post-Pathfinder evaluation of areocentric solar coordinates with improved timing recipes for Mars seasonal/diurnal climate studies.* Planetary and Space Science, 48(2–3), 215–235. [doi:10.1016/S0032-0633(99)00092-6](https://doi.org/10.1016/S0032-0633(99)00092-6)

2. Allison, M. (1997). *Accurate analytical representations of solar time and seasons on Mars with applications to the Pathfinder/Surveyor missions.* Geophysical Research Letters, 24(16), 1967–1970. [doi:10.1029/97GL01950](https://doi.org/10.1029/97GL01950)

3. NASA/GISS Mars24 Algorithm Documentation. [https://www.giss.nasa.gov/tools/mars24/help/algorithm.html](https://www.giss.nasa.gov/tools/mars24/help/algorithm.html)

4. Sainton, G. (2019). *MarsTimeConverter* — bibliothèque Python pour la conversion des temps terrestres/martiens. Institut de Physique du Globe / PSS Team. [github.com/GregS1t/marstimeconverter](https://github.com/GregS1t/marstimeconverter)

---

*Code source complet de toutes les conversions présentées ici : [github.com/GregS1t/marstimeconverter](https://github.com/GregS1t/marstimeconverter)*
