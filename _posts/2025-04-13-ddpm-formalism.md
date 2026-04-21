---
layout: post
title: "Le formalisme DDPM — du bruit à l'image — DDPM 3"
date: 2025-11-13
description: >
  Processus forward, processus reverse, objectif d'entraînement simplifié
  et algorithmes d'échantillonnage de Ho et al. (2020).
tags: [deep-learning, diffusion, ddpm, generative-models, astrophysique]
categories: deep-learning
series: "Modèles génératifs pour la morphologie galactique"
series_order: 5
related_posts: false
toc:
  sidebar: left
math: true
---

## Introduction

Jusqu'ici, j'ai décrit l'architecture du réseau de débruitage — le U-Net. Dans ce billet, je pose le cadre mathématique qui donne son sens à ce réseau : le **processus de diffusion**. Je suis les notations de Ho et al. (2020) tout au long du billet, avec les correspondances vers le code.

---

## 1. Vue d'ensemble

Un DDPM repose sur deux processus Markoviens opposés :

- Le **processus forward** $q$ — fixé, sans paramètres apprenables — "corrompt" progressivement une image $\mathbf{x}_0$ en y ajoutant du "bruit gaussien", jusqu'à obtenir $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$.
- Le **processus reverse** $p_\theta$ — appris — reconstruit l'image propre en débruitant pas à pas depuis $\mathbf{x}_T$.

L'idée centrale est simple : si on sait débruiter une image légèrement bruitée, on peut enchaîner $T$ étapes de débruitage pour reconstruire une image propre depuis du bruit pur. Le U-Net apprend à réaliser chacune de ces étapes.

---

## 2. Le processus forward $q$

### 2.1 Transition élémentaire

À chaque pas $t$, le processus forward ajoute un bruit gaussien contrôlé par un **schedule de variance** $\beta_1, \dots, \beta_T$ :

$$
q(\mathbf{x}_t \mid \mathbf{x}_{t-1})
:= \mathcal{N}\!\left(\mathbf{x}_t;\;
\sqrt{1-\beta_t}\,\mathbf{x}_{t-1},\; \beta_t \mathbf{I}\right)
\tag{Ho, Eq. 2}
$$

Le facteur $\sqrt{1-\beta_t}$ atténue légèrement l'image à chaque pas, tandis que $\beta_t$ contrôle la quantité de bruit ajoutée. Ho et al. (2020) utilisent un schedule linéaire de $\beta_1 = 10^{-4}$ à $\beta_T = 0.02$ — dans notre implémentation, nous utilisons le **schedule cosinus** de Nichol & Dhariwal (2021), qui maintient un ratio signal/bruit plus élevé sur la première moitié du processus (voir billet suivant).

### 2.2 Le trick du bruitage direct

En posant $\alpha_t := 1 - \beta_t$ et $\bar{\alpha}_t := \prod_{s=1}^{t} \alpha_s$, le processus forward admet un **échantillonnage en forme close** à n'importe quel pas $t$ — sans simuler les $t$ transitions successives :

$$
q(\mathbf{x}_t \mid \mathbf{x}_0)
= \mathcal{N}\!\left(\mathbf{x}_t;\;
\sqrt{\bar{\alpha}_t}\,\mathbf{x}_0,\;
(1-\bar{\alpha}_t)\mathbf{I}\right)
\tag{Ho, Eq. 4}
$$

Ce qui se réécrit directement :

$$
\mathbf{x}_t = \sqrt{\bar{\alpha}_t}\,\mathbf{x}_0 + \sqrt{1-\bar{\alpha}_t}\,\boldsymbol{\epsilon}, \qquad \boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})
$$

C'est cette équation qui est au cœur de la méthode `q_sample` dans notre code :

```python
def q_sample(self, x_0, t, eps=None):
    if eps is None:
        eps = torch.randn_like(x_0)
    alpha_bar_t = self.alpha_bars[t].view(-1, 1, 1, 1)
    return alpha_bar_t.sqrt() * x_0 + (1 - alpha_bar_t).sqrt() * eps
```

**Pourquoi c'est utile ?** À l'entraînement, on tire $t$ aléatoirement à chaque itération et on génère $\mathbf{x}_t$ directement en $O(1)$, sans simuler la chaîne de Markov depuis $\mathbf{x}_0$.

---

## 3. Le processus reverse $p_\theta$

Le processus reverse est une chaîne de Markov apprise qui part de $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ et reconstruit $\mathbf{x}_0$ pas à pas :

$$
p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t)
:= \mathcal{N}\!\left(\mathbf{x}_{t-1};\;
\boldsymbol{\mu}_\theta(\mathbf{x}_t, t),\;
\sigma_t^2 \mathbf{I}\right)
$$

Ho et al. (2020) fixent la variance à $\sigma_t^2 = \beta_t$ (non apprise) et paramètrent uniquement la **moyenne** $\boldsymbol{\mu}_\theta$ via le réseau. Nichol & Dhariwal (2021) montrent que l'apprentissage de $\sigma_t^2$ permet de réduire le nombre de pas d'inférence d'un ordre de grandeur — c'est une perspective d'amélioration future non encore implémentée dans notre code.

---

## 4. L'objectif d'entraînement

### 4.1 Philosophie de l'ELBO

L'entraînement optimal consisterait à maximiser la log-vraisemblance $\log p_\theta(\mathbf{x}_0)$ — la probabilité d'observer les vraies images sous le modèle. Ce calcul est intractable directement, mais on peut dériver une **borne inférieure variationnelle** (ELBO) qui décompose l'objectif en termes interprétables.

Sans entrer dans les calculs (cf. Ho et al. 2020, Annexe A pour la dérivation complète), la philosophie est la suivante : le processus reverse $p_\theta$ doit imiter au mieux la **posterior forward** $q(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0)$ — la distribution exacte de $\mathbf{x}_{t-1}$ sachant $\mathbf{x}_t$ et $\mathbf{x}_0$, qui est une gaussienne calculable analytiquement. Minimiser la divergence KL entre ces deux distributions à chaque pas donne l'objectif d'entraînement.

### 4.2 La paramétrisation par le bruit

Ho et al. (2020) montrent qu'il est plus efficace de paramétrer $\boldsymbol{\mu}_\theta$ non pas comme une prédiction directe de la moyenne, mais via la **prédiction du bruit** $\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)$ :

$$
\boldsymbol{\mu}_\theta(\mathbf{x}_t, t) = \frac{1}{\sqrt{\alpha_t}}
\left(
  \mathbf{x}_t - \frac{\beta_t}{\sqrt{1 - \bar{\alpha}_t}}
    \,\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)
\right)
\tag{Ho, Eq. 11}
$$

Le U-Net joue le rôle de $\boldsymbol{\epsilon}_\theta$ — il prédit le bruit qui a été ajouté à $\mathbf{x}_0$ pour obtenir $\mathbf{x}_t$.

### 4.3 La loss simplifiée $L_\text{simple}$

Après simplification des poids de l'ELBO — qui défavorisent les petits $t$ (faible bruit) et nuisent à la qualité des échantillons — Ho et al. (2020) proposent une loss remarquablement simple :

$$
\boxed{
L_\text{simple}(\theta)
:= \mathbb{E}_{t,\,\mathbf{x}_0,\,\boldsymbol{\epsilon}}\!\left[
  \left\|
    \boldsymbol{\epsilon}
    - \boldsymbol{\epsilon}_\theta\!\left(
        \sqrt{\bar{\alpha}_t}\,\mathbf{x}_0
        + \sqrt{1-\bar{\alpha}_t}\,\boldsymbol{\epsilon},\;
        t
      \right)
  \right\|^2
\right]
}
\tag{Ho, Eq. 14}
$$

C'est une **MSE entre le bruit réel et le bruit prédit** — directement implémentée dans `compute_loss` :

```python
def compute_loss(self, x_0, t, eps=None):
    if eps is None:
        eps = torch.randn_like(x_0)
    x_t = self.q_sample(x_0, t, eps)
    eps_theta = self.predict_eps(x_t, t)
    return torch.mean((eps - eps_theta) ** 2)
```

---

## 5. Les algorithmes

### Algorithme 1 — Entraînement

À chaque itération : tirer une image $\mathbf{x}_0$, un pas de temps $t$ aléatoire, un bruit $\boldsymbol{\epsilon}$, construire $\mathbf{x}_t$ par le trick de l'Eq. 4, prédire le bruit avec le U-Net, descendre le gradient de $L_\text{simple}$.

```python
x_0 = batch[0].to(device)
t   = torch.randint(0, ddpm.n_steps, (len(x_0),)).to(device)
loss = ddpm.compute_loss(x_0, t)
```

### Algorithme 2 — Génération

Partir de $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ et appliquer $T$ fois la mise à jour du processus reverse :

$$
\mathbf{x}_{t-1} = \frac{1}{\sqrt{\alpha_t}} \left( \mathbf{x}_t - \frac{1 - \alpha_t}{\sqrt{1 - \bar{\alpha}_t}}\,\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t) \right) + \sigma_t\,\mathbf{z}, \qquad \mathbf{z} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})
$$

```python
x_t = torch.randn(n_samples, c, h, w).to(self.device)
for t in reversed(range(self.n_steps)):
    t_batch = (torch.ones(n_samples, 1) * t).long().to(self.device)
    eps_theta = self.predict_eps(x_t, t_batch)
    alpha_t = self.alphas[t]
    alpha_bar_t = self.alpha_bars[t]
    denom = (1 - alpha_bar_t).sqrt().clamp(min=1e-8)
    x_t = (1 / alpha_t.sqrt()) * (x_t - (1 - alpha_t) / denom * eps_theta)
    if t > 0:
        x_t = x_t + self.betas[t].sqrt() * torch.randn_like(x_t)
```
---




---

## Références

- Ho, J., Jain, A., & Abbeel, P. (2020). *Denoising Diffusion Probabilistic Models*. NeurIPS 2020. [arXiv:2006.11239](https://arxiv.org/abs/2006.11239)

- Nichol, A., & Dhariwal, P. (2021). *Improved Denoising Diffusion Probabilistic Models*. ICML 2021. [arXiv:2102.09672](https://arxiv.org/abs/2102.09672)

- Sohl-Dickstein, J., et al. (2015). *Deep Unsupervised Learning using Nonequilibrium Thermodynamics*. ICML 2015. [arXiv:1503.03585](https://arxiv.org/abs/1503.03585)

---

*Prochain billet : [Les embeddings sinusoïdaux — conditionner un réseau sur le temps →](/blog/2025/sinusoidal-embedding/)*
