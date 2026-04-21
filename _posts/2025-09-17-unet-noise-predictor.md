---
layout: post
title: "Le U-Net comme prédicteur de bruit — DDPM 3"
date: 2025-09-17
description: >
  Architecture du U-Net adapté au DDPM : encodeur, bottleneck, décodeur,
  skip connections, conditionnement temporel.
tags: [deep-learning, unet, architecture, DDPM, astrophysique]
categories: deep-learning
series: "Modèles génératifs pour la morphologie galactique"
series_order: 3
related_posts: false
toc:
  sidebar: left
math: true
---

## Introduction

Dans les deux billets précédents, j'ai posé les briques de base : les convolutions, le gradient vanishing et les connexions résiduelles. Je peux maintenant assembler ces éléments dans l'architecture centrale du DDPM — le **U-Net** — qui joue le rôle de prédicteur de bruit $\boldsymbol{\varepsilon}_\theta(\mathbf{x}_t, t)$. Je prends exactement les développements effectués dans le papier de Ho et al. (2020)

---

## 1. Le U-Net : une architecture en forme de U

Le U-Net (Ronneberger et al. 2015) a été conçu à l'origine pour la segmentation d'images biomédicales. Son architecture en forme de "U" lui a valu son nom : un **encodeur** qui compresse l'information spatiale, un **bottleneck** au niveau de résolution minimale, et un **décodeur** qui reconstruit la résolution — le tout relié par des **skip connections** entre niveaux symétriques.

Dans le contexte du DDPM, le U-Net remplit un rôle différent de la segmentation : il prend en entrée une image bruitée $\mathbf{x}_t$ et prédit le bruit $\boldsymbol{\varepsilon}$ qui y a été ajouté, produisant une sortie de même taille.

---

## 2. L'encodeur — extraire et compresser

L'encodeur est une suite de `ResBlock` séparés par des opérations de sous-échantillonnage. À chaque niveau, la résolution spatiale est divisée par 2 tandis que le nombre de canaux augmente, forçant le réseau à construire des représentations de plus en plus abstraites.

Dans l'U-Net original (Ronneberger et al. 2015), le sous-échantillonnage est réalisé par un **max-pooling 2×2**. Dans notre implémentation, nous le remplaçons par une **convolution stride-2** (Springenberg et al. 2014), qui permet au réseau d'apprendre lui-même comment réduire la résolution plutôt que d'appliquer une sélection fixe.

```
Entrée    (B,   3, 64, 64)
enc1  ->  (B,  64, 64, 64)  — 2 × ResBlock
down1 ->  (B,  64, 32, 32)  — Conv2d stride=2
enc2  ->  (B, 128, 32, 32)  — 2 × ResBlock
down2 ->  (B, 128, 16, 16)  — Conv2d stride=2
enc3  ->  (B, 256, 16, 16)  — 2 × ResBlock + SelfAttention
down3 ->  (B, 256,  8,  8)  — Conv2d stride=2
```

---

## 3. Le bottleneck et la self-attention

Le bottleneck est le niveau de résolution minimale (8×8). C'est là que le réseau dispose de la vision la plus globale de l'image — chaque position de la feature map 8×8 représente une région de 8×8 pixels de l'image originale 64×64.

Un bloc de **self-attention** est inséré au bottleneck, conformément à Ho et al. (2020). Les convolutions locales ne peuvent établir des relations qu'entre pixels voisins (fenêtre 3×3) ; la self-attention permet d'établir des corrélations entre **toutes les positions spatiales simultanément**, quelle que soit leur distance. Pour une galaxie spirale, cela signifie que le réseau peut apprendre que les structures symétriques de part et d'autre du noyau sont corrélées.

```python
# Bottleneck : (B, 256, 8, 8) -> (B, 256, 8, 8)
self.mid_a    = ResBlock(256, 256, time_emb_dim)
self.mid_attn = SelfAttention(256, n_heads=8)
self.mid_b    = ResBlock(256, 256, time_emb_dim)
```

Le coût de la *self-attention* est en $O((H \times W)^2)$ — à 8×8 il est de $64^2 = 4\,096$ opérations, tout à fait raisonnable. J'y reviendrai dans le billet suivant dédié à la self-attention.

---

## 4. Le décodeur et les *skip connections*

Le décodeur reconstruit progressivement la résolution spatiale par des **convolutions transposées** (*upsampling*). La spécificité du U-Net est que chaque niveau du décodeur reçoit en entrée **deux flux concaténés** :

1. La sortie du niveau précédent du décodeur (information sémantique globale)
2. La sortie du niveau **symétrique** de l'encodeur (information locale fine)

$$
\mathbf{f}_\text{dec}^{(\ell)} = \text{ResBlock}\!\left(
  \left[\mathbf{f}_\text{enc}^{(\ell)} \;\|\; \text{Up}\!\left(\mathbf{f}_\text{dec}^{(\ell+1)}\right)\right],\; t
\right)
$$

où $[\cdot \| \cdot]$ désigne la concaténation selon l'axe des canaux et $t$ le pas de temps injecté via le *time embedding*.

**On a vraiment besoin des *skip connections* ?** Dans notre contexte de prédiction de bruit, le réseau doit produire une image de même résolution que l'entrée. Sans les skip connections, l'encodeur compresserait l'information spatiale de manière irréversible — les détails fins (localisation précise des structures galactiques, gradients de brillance locaux) seraient perdus dans le goulot d'étranglement (le *bottleneck*). Grâce aux skip connections, ces détails sont directement accessibles au décodeur, contournant le bottleneck.

En pytorch, ça donne un truc comme ça :
```python
# Décodeur avec skip connections
d3 = self.dec3_a(torch.cat([e3, self.up3(mid)], dim=1), t_emb)  # (B, 256, 16, 16)
d3 = self.dec3_b(d3, t_emb)                                      # (B, 128, 16, 16)
d3 = self.attn_dec3(d3)                                          # self-attention 16x16

d2 = self.dec2_a(torch.cat([e2, self.up2(d3)], dim=1), t_emb)  # (B, 128, 32, 32)
d2 = self.dec2_b(d2, t_emb)                                      # (B,  64, 32, 32)

d1 = self.dec1_a(torch.cat([e1, self.up1(d2)], dim=1), t_emb)  # (B,  64, 64, 64)
d1 = self.dec1_b(d1, t_emb)                                      # (B,  64, 64, 64)
```

---

## 5. Le conditionnement temporel

Le U-Net décrit ci-dessus est un réseau image-vers-image. Pour l'utiliser dans un DDPM, il faut lui communiquer le pas de temps $t$ courant, qui détermine le niveau de bruit de l'image en entrée et donc le type de débruitage attendu.

Cette injection se fait en deux étapes. D'abord, $t$ est converti en vecteur par des **embeddings sinusoïdaux** de dimension 256 (voir l'un des billets suivants). Ensuite, ce vecteur passe par un **MLP** à deux couches :

$$
\mathbf{e}_\text{proj}(t) = \text{Linear}_{d \to d}\!\left(\text{SiLU}\!\left(\text{Linear}_{d \to 4d}\!\left(\mathbf{e}(t)\right)\right)\right)
$$

Le vecteur projeté $\mathbf{e}_\text{proj}(t)$ est ensuite injecté **à l'intérieur de chaque ResBlock entre les deux convolutions**, via une projection linéaire broadcast-additionnée à la feature map intermédiaire :

$$
\mathbf{f}_\text{mid} \leftarrow \mathbf{f}_\text{mid} + \text{Linear}_{d \to C}\!\left(\mathbf{e}_\text{proj}(t)\right)
$$

Ce placement entre les deux convolutions — conforme à Ho et al. (2020) — permet à chaque bloc de moduler sa transformation intermédiaire en fonction du pas de temps courant.

Vous reprendrez bien un peu de code ? 
```python
# unet_v2.py — UNetV2.forward()
t_emb = self.time_mlp(self.time_embed(t))   # (B, time_emb_dim)

# Dans chaque ResBlock
out = self.activation(self.conv1(self.norm1(x)))
out = out + self.time_proj(t_emb).reshape(B, -1, 1, 1)  # injection entre conv1 et conv2
out = self.activation(self.conv2(self.norm2(out)))
return out + self.shortcut(x)
```
---

## Références

- Ronneberger, O., Fischer, P., & Brox, T. (2015). *U-Net: Convolutional Networks for Biomedical Image Segmentation*. MICCAI 2015. [arXiv:1505.04597](https://arxiv.org/abs/1505.04597)

- Springenberg, J. T., Dosovitskiy, A., Brox, T., & Riedmiller, M. (2014). *Striving for Simplicity: The All Convolutional Net*. [arXiv:1412.6806](https://arxiv.org/abs/1412.6806)

- Ho, J., Jain, A., & Abbeel, P. (2020). *Denoising Diffusion Probabilistic Models*. NeurIPS 2020. [arXiv:2006.11239](https://arxiv.org/abs/2006.11239)

- Nichol, A., & Dhariwal, P. (2021). *Improved Denoising Diffusion Probabilistic Models*. ICML 2021. [arXiv:2102.09672](https://arxiv.org/abs/2102.09672)

