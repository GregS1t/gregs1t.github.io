---
layout: post
title: Entendre un séisme de magnitude 5 sur Mars  
date: 2022-05-23
description: Dans la nuit du 4 au 5 mai 2022, Mars a tremblé pour la première fois à magnitude 5. Ecoutez le séisme.
tags: [sismologie, Mars, InSight, SEIS, IPGP, traitement du signal]
categories: [planetologie, mars, traitement du signal]
related_posts: false
toc:
  sidebar: left
---

*"T'as entendu parler du tremblement de Mars ?"*

Non, ce n'est pas le titre d'un film de SF des années 80. Dans la nuit du 4 au 5 mai 2022, Mars a vraiment tremblé — magnitude 5 sur l'échelle de Richter. Et c'est l'instrument français SEIS, posé sur le sol martien par l'atterrisseur InSight de la NASA, qui l'a capté en direct.

## Pourquoi c'est un événement

Depuis le début des opérations fin 2018, les équipes attendaient ça. Pas juste un petit frémissement, mais un vrai séisme qui rentre dans les objectifs scientifiques les plus ambitieux de la mission. Ce n'est pas anecdotique : en termes d'énergie libérée, c'est environ **10 fois plus** que le record précédent.

À titre de comparaison, sur Terre un magnitude 5, ça reste un séisme modéré. Sur Mars, on frôle la limite haute de ce que les scientifiques espéraient pouvoir observer avec InSight.

Les conditions d'observation étaient également idéales : l'événement s'est produit tôt le matin martien, avant les perturbations atmosphériques de l'aube, et à une distance épicentrale relativement courte — 2 250 km à l'échelle martienne. 

## Mon rôle là-dedans : le déglitch

La vidéo de sonification de l'événement publiée par le CNES et l'IPGP a été réalisée en deux étapes. La première, c'était mon boulot : le **déglitch** des données brutes de SEIS.

Une fois le signal nettoyé, Rémi Lapeyre (responsable des opérations InSight/SEIS au CNES) a pris le relai pour la sonification.

## Comment on transforme un séisme en son ?

**Un signal sismique, c'est quoi à la base ?** Les capteurs VBB (Very Broad Band) de SEIS couvrent la plage 0,01–10 Hz ([Lognonné et al., 2019, *Space Sci. Rev.*](https://link.springer.com/article/10.1007/s11214-018-0574-6)). C'est largement en dessous du seuil d'audibilité humain, qui démarre autour de 20 Hz. > Autrement dit, même posé directement sur Mars, votre oreille n'entendrait rien.

Pour rendre le signal audible, le principe est simple : accélérer la lecture d'un enregistrement revient à multiplier toutes ses fréquences par le même facteur, les faisant glisser mécaniquement dans le domaine audible. C'est une technique bien établie en sismologie terrestre ([USGS](https://www.usgs.gov/programs/earthquake-hazards/science/earthquake-sounds)).
Ici, le signal a été accéléré ×44, puis amplifié pour atteindre un niveau d'écoute confortable. Ce n'est pas une reconstruction artistique : c'est le signal physique brut transposé dans une gamme que nos oreilles peuvent percevoir.

La même approche est utilisée dans d'autres domaines : les ondes gravitationnelles détectées par LIGO sont ainsi converties en son en les jouant à des fréquences plus élevées que celles du signal original ([LIGO Lab, Caltech](https://www.ligo.caltech.edu/video/ligo20160211v2)).

Dernière subtilité propre à SEIS : En mixant leurs signaux sur deux canaux, l'équipe a obtenu un léger effet stéréo, les deux pistes ne sont pas tout à fait identiques, et ça s'entend.

## Mettez 🎧,  ↗️ un peu le son 🎚️ 
<center>
  <div style="display: flex; justify-content: center;">
    <figure style="width: 100%;">
      {% include video.liquid path="https://assets.science.nasa.gov/content/dam/science/psd/mars/downloadable_items/4/7/47251_PIA25281.mp4" controls=true width="60%" %}
      <figcaption style="text-align: center;">Sonification du séisme de magnitude 5 enregistré par SEIS dans la nuit du 4 au 5 mai 2022. © CNES/NASA/JPL-Caltech</figcaption>
    </figure>
  </div>
</center>

## Références

- IPGP (2022). *Premier séisme de magnitude 5 « entendu » sur Mars.* [ipgp.fr](https://www.ipgp.fr/fr/premier-seisme-de-magnitude-5-entendu-mars)
- CNES (2022). *InSight/SEIS : 1er séisme de magnitude 5 « entendu » sur Mars.* [cnes.fr](https://cnes.fr/actualites/insightseis-1er-seisme-de-magnitude-5-entendu-mars)
- Lognonné, P., et al. (2019). SEIS: Insight's Seismic Experiment for Internal Structure of
  Mars. *Space Sci. Rev.*, 215, 12.   [DOI: 10.1007/s11214-018-0574-6](https://doi.org/10.1007/s11214-018-0574-6)
- LIGO Lab, Caltech. *The Sound of Two Black Holes Colliding.* [ligo.caltech.edu](https://www.ligo.caltech.edu/video/ligo20160211v2)