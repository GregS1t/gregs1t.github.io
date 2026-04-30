---
layout: post
title: "La self-attention dans le U-Net — DDPM 4"
date: 2025-10-24
description: >
  Mécanisme de la self-attention, implémentation dans le U-Net du DDPM,
  et justification du choix des résolutions 8×8 et 16×16.
tags: [deep-learning, attention, transformer, unet, DDPM]
categories: deep-learning
series: "Modèles génératifs pour la morphologie galactique"
published: false
series_order: 4
related_posts: false
toc:
  sidebar: left
math: true
---

## Introduction

Dans le billet précédent, j'ai mentionné que le U-Net intègre des blocs de **self-attention** au bottleneck et à la résolution 16×16. Dans ce billet, je détaille un peu plus ce mécanisme que je trouve assez compliqué à appréhender.

---

## 1. La limite des convolutions locales

Une convolution 3×3 ne voit que les 9 pixels voisins d'une position donnée. Pour qu'une information "voyage" d'un coin de l'image à l'autre, il faut empiler de nombreuses couches — et l'information se dilue chemin faisant.

Pour nos images de galaxies de GZ2 (comme pour beaucoup d'images, d'ailleurs), c'est une vraie limitation. Par exemple, un bras spiral à gauche du noyau est physiquement corrélé avec le bras symétrique à droite — ils ont la même origine, la même morphologie. Une convolution locale ne peut pas capturer cette relation directement : elle nécessiterait une fenêtre de réception (*receptive field*) couvrant l'ensemble de l'image, ce qui demanderait un empilement très important de couches.

La *self-attention* résout ce problème en permettant à chaque position de "regarder" toutes les autres positions **en une seule opération**.

---

## 2. Intuition — avec les doigts

Imaginons qu'on regarde une *feature map* après l'encodeur : chaque position $(i, j)$ détient un vecteur de caractéristiques qui résume ce qui se passe dans cette région de l'image.

La *self-attention* donne à chaque position trois rôles simultanés :

- Elle **pose une question** (*query*, $Q$) : "Euh, je cherche des positions qui ressemblent à ceci..."
- Elle **affiche une étiquette** (*key*, $K$) : "Alors moi, je suis ce type de structure..."
- Elle **détient une valeur** (*value*, $V$) : "Voici l'information que je peux apporter"

Pour chaque position, on compare sa question à toutes les étiquettes de toutes les autres positions. Les positions dont l'étiquette correspond bien à la question reçoivent un poids élevé. La nouvelle représentation de la position est alors une **moyenne pondérée des valeurs** de toutes les autres positions — les plus "pertinentes" contribuant le plus.

Dans le cas de la galaxie spirale : une position dans un bras spiral pose la question "qui me ressemble ?", identifie les positions de l'autre bras comme similaires, et intègre leur information dans sa propre représentation. Le réseau apprend que ces deux bras sont corrélés — ce qui guidera le débruitage de manière cohérente sur l'ensemble de l'image.

Exemple imagé : 

>**Position A** — un pixel dans un bras spiral à gauche du noyau :
>
>- Query : "je cherche des structures allongées et lumineuses qui partent d'un noyau central"
>- Key : "je suis un segment de bras spiral, lumineux, orienté à 45°"
>- Value : "ma brillance de surface est X, mon gradient de couleur est Y"
>
>**Position B** — le bras spiral symétrique à droite :
>
>- Key : "je suis aussi un segment de bras spiral, lumineux, orienté à 225°"
>
>**Position C** — le fond de ciel, sombre, sans structure :
>
>- Key : "je suis une région homogène et sombre"

---

## 3. Formalisme

Si on revient à un formalisme mathématique, celui de (Vaswani et al. 2017), pour une séquence de vecteurs, on calcule trois projections linéaires — **query** $\mathbf{Q}$, **key** $\mathbf{K}$ et **value** $\mathbf{V}$ — puis :

$$
\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\!\left(\frac{\mathbf{Q}\mathbf{K}^\top}{\sqrt{d_k}}\right)\mathbf{V}
$$

- $\mathbf{Q}\mathbf{K}^\top$ est la matrice de similarité entre toutes les paires de positions — c'est ce produit scalaire qui donne à la self-attention son caractère global.
- $\sqrt{d_k}$ normalise les similarités pour éviter la saturation du softmax quand la dimension $d_k$ est grande.
- Le softmax transforme les similarités en poids somme à 1.
- La multiplication par $\mathbf{V}$ produit la somme pondérée des valeurs.

En pratique, on utilise la **multi-head attention** : on répète ce mécanisme $h$ fois en parallèle avec des projections différentes, puis on concatène les résultats. Chaque "tête" peut se spécialiser sur un type de relation différent — une tête peut capturer les symétries, une autre les gradients de brillance, *etc*.

$$
\text{MultiHead}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{Concat}(\text{head}_1, \dots, \text{head}_h)\,\mathbf{W}^O
$$

$$
\text{head}_i = \text{Attention}(\mathbf{Q}\mathbf{W}_i^Q,\, \mathbf{K}\mathbf{W}_i^K,\, \mathbf{V}\mathbf{W}_i^V)
$$

---

## 4. Implémentation sur une feature map

Dans le contexte d'un U-Net, les données ne sont pas des séquences de tokens mais des **feature maps** de forme $(B, C, H, W)$. Il faut donc adapter la self-attention à ce format.

La transformation est simple :

```
(B, C, H, W)  →  flatten(2)  →  (B, C, H*W)  →  permute(2,0,1)  →  (H*W, B, C)
```

Chaque position spatiale $(i, j)$ devient un token de dimension $C$. Après l'attention, on remet en forme :

```
(H*W, B, C)  →  permute(1,2,0)  →  (B, C, H*W)  →  reshape  →  (B, C, H, W)
```

Une **connexion résiduelle** est ajoutée — la sortie de l'attention est additionnée à l'entrée, conformément à la convention des Transformers (Vaswani et al. 2017) et au ResBlock du billet 2.

```python
class SelfAttention(nn.Module):
    """
    Self-attention block for spatial feature maps (Ho et al., 2020).
    Reshapes (B, C, H, W) -> (H*W, B, C), applies multi-head attention,
    then reshapes back. Residual connection added.
    """
    def __init__(self, n_channels, n_heads=8):
        super(SelfAttention, self).__init__()
        self.norm = nn.GroupNorm(32, n_channels)
        self.attn = nn.MultiheadAttention(n_channels, n_heads)

    def forward(self, x):
        B, C, H, W = x.shape
        out = self.norm(x)
        out = out.flatten(2).permute(2, 0, 1)       # (H*W, B, C)
        out, _ = self.attn(out, out, out)            # query = key = value
        out = out.permute(1, 2, 0).reshape(B, C, H, W)
        return x + out                               # connexion résiduelle
```

La normalisation utilisée est `GroupNorm(32)` — conforme au reste du U-Net — appliquée avant l'attention (convention *pre-norm*).

---

## 5. À quelles résolutions appliquer l'attention ?

Le coût de la self-attention est en $O((H \times W)^2)$ — il croît comme le carré du nombre de tokens. Le tableau suivant illustre l'impact selon la résolution :

| Résolution | Tokens | Coût relatif |
|---|---|---|
| 8×8 | 64 | 1× |
| 16×16 | 256 | 16× |
| 32×32 | 1 024 | 256× |
| 64×64 | 4 096 | 4 096× |

L'attention à 64×64 est hors de portée. À 32×32, le coût est 256× celui du bottleneck — difficilement justifiable pour des images 64×64. J'ai donc retenu **8×8** et **16×16**, conformément à Nichol & Dhariwal (2021) qui montrent que l'attention à plusieurs résolutions améliore la qualité des échantillons.

Ces deux résolutions ne sont pas équivalentes en termes de ce qu'elles capturent :

- **8×8 (bottleneck)** : chaque token représente une région de 8×8 pixels de l'image originale. L'attention entre 64 tokens capture les **relations globales** — symétrie générale, cohérence morphologique d'ensemble.
- **16×16 (encodeur et décodeur)** : chaque token représente une région de 4×4 pixels. L'attention entre 256 tokens capture les **corrélations à moyenne échelle** — cohérence des bras spiraux, structure du bulbe par rapport aux régions externes.

---

## 6. La self-attention dans notre U-Net

Dans `UNetV2`, la self-attention est placée à trois endroits :

- **Encodeur niveau 3** (16×16) : après les deux ResBlocks, avant le downsampling
- **Bottleneck** (8×8) : entre les deux ResBlocks du milieu
- **Décodeur niveau 3** (16×16) : après les deux ResBlocks, avant l'upsampling suivant

```python
# Encodeur 16x16
e3 = self.enc3_b(self.enc3_a(self.down2(e2), t_emb), t_emb)
e3 = self.attn_enc3(e3)                            # (B, 256, 16, 16)

# Bottleneck 8x8
mid = self.mid_a(self.down3(e3), t_emb)
mid = self.mid_attn(mid)                           # (B, 256, 8, 8)
mid = self.mid_b(mid, t_emb)

# Décodeur 16x16
d3 = self.dec3_b(d3, t_emb)
d3 = self.attn_dec3(d3)                            # (B, 128, 16, 16)
```

La symétrie encodeur/décodeur à 16×16 est intentionnelle : les skip connections relient ces deux niveaux, il est donc cohérent de traiter l'information spatiale de la même façon des deux côtés.

---

## Références

- Vaswani, A., et al. (2017). *Attention Is All You Need*. NeurIPS 2017. [arXiv:1706.03762](https://arxiv.org/abs/1706.03762)

- Ho, J., Jain, A., & Abbeel, P. (2020). *Denoising Diffusion Probabilistic Models*. NeurIPS 2020. [arXiv:2006.11239](https://arxiv.org/abs/2006.11239)

- Nichol, A., & Dhariwal, P. (2021). *Improved Denoising Diffusion Probabilistic Models*. ICML 2021. [arXiv:2102.09672](https://arxiv.org/abs/2102.09672)

---

*Prochain billet : [Le formalisme DDPM — du bruit à l'image →](/blog/2025/ddpm-formalism/)*
