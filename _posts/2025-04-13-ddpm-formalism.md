---
layout: post
title: "Formalisme DDPM — du bruit à l'image - DDPM 3"
date: 2025-04-13
description: >
  Dérivation complète du DDPM : processus forward, ELBO, simplification
  de la loss, processus inverse et algorithmes d'entraînement et
  d'échantillonnage de Ho et al. (2020).
tags: [deep-learning, diffusion, ddpm, generative-models, astrophysique]
categories: deep-learning
series: "Modèles génératifs pour la morphologie galactique"
series_order: 3
related_posts: true
toc:
  sidebar: left
math: true
---

> **Série — Modèles génératifs appliqués aux images de galaxies**
> Ce billet est le troisième d'une série consacrée à l'application des modèles
> de diffusion (DDPM) aux images de galaxies du catalogue Galaxy Zoo 2.
> Le code complet est disponible sur [github.com/GregS1t/DDPM_GalaxyZoo2](https://github.com/GregS1t/DDPM_GalaxyZoo2).
>
> **Notations.** Ce billet suit strictement les notations de Ho et al. (2020).
> Les correspondances avec le code sont indiquées au fil du texte.

---

## 1. Vue d'ensemble

Un DDPM repose sur deux processus Markoviens opposés (Fig. 1) :

- Le **processus forward** $q$ (fixé, sans paramètres) corrompu progressivement une image $\mathbf{x}_0$ en y ajoutant du bruit gaussien jusqu'à obtenir $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$.

- Le **processus reverse** $p_\theta$ (appris) reconstruit l'image propre en débruitant pas à pas depuis $\mathbf{x}_T$.

<!--<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/DDPM_3_article_Ho/20250317_autoencoder_reconstructed_images.gif"
       width="75%"
       alt="Processus forward et reverse du DDPM">
  <figcaption>
    Figure 1 — Modèle graphique dirigé du DDPM (Ho et al., 2020, Fig. 2).
    En haut : processus forward.
    En bas : processus reverse.
  </figcaption>
</figure>
-->
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

Ho et al. fixent $\beta_t$ constants (non appris), croissant linéairement de $\beta_1 = 10^{-4}$ à $\beta_T = 0.02$.

### 2.2 Bruitage direct à l'étape $t$ — le $trick$

En posant $\alpha_t := 1 - \beta_t$ et $\bar{\alpha}_t := \prod_{s=1}^{t} \alpha_s$,
le processus forward admet un **échantillonnage en forme close** à n'importe quel pas $t$ :

$$
q(\mathbf{x}_t \mid \mathbf{x}_0)
= \mathcal{N}\!\left(\mathbf{x}_t;\;
\sqrt{\bar{\alpha}_t}\,\mathbf{x}_0,\;
(1-\bar{\alpha}_t)\mathbf{I}\right)
\tag{Ho, Eq. 4}
$$

Ce qui se réécrit de manière équivalente — c'est la formule directement utilisée dans le code :

$$
\mathbf{x}_t = \sqrt{\bar{\alpha}_t}\,\mathbf{x}_0 + \sqrt{1-\bar{\alpha}_t}\,\boldsymbol{\epsilon}, \qquad \boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})
$$

Cette équation est au cœur de la méthode `diffuse` dans notre code :

```python
# ddpm_model.py — DDPMModel.diffuse()
# Eq. 4 de Ho et al. : x_t = sqrt(alpha_bar_t)*x0 + sqrt(1-alpha_bar_t)*eps
alpha_bar_t = self.alpha_bars[t].view(-1, 1, 1, 1)
x_noisy = alpha_bar_t.sqrt() * x0 + (1.0 - alpha_bar_t).sqrt() * noise
```

> **Pourquoi c'est utile ?** Sans cette propriété, il faudrait simuler les $t$ transitions successives pour obtenir $\mathbf{x}_t$.
> Ici, on saute directement à n'importe quel niveau de bruit en $O(1)$ — ce qui rend l'entraînement par stochastic gradient descent efficace : on tire $t$ aléatoirement à chaque itération.

---

## 3. Le processus reverse $p_\theta$

Le processus reverse est une chaîne de Markov apprise, définie comme :

$$
p_\theta(\mathbf{x}_{0:T}) := p(\mathbf{x}_T)
\prod_{t=1}^{T} p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t)
\tag{Ho, Eq. 1}
$$

avec $p(\mathbf{x}_T) = \mathcal{N}(\mathbf{0}, \mathbf{I})$ et des transitions gaussiennes :

$$
p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t)
:= \mathcal{N}\!\left(\mathbf{x}_{t-1};\;
\boldsymbol{\mu}_\theta(\mathbf{x}_t, t),\;
\boldsymbol{\Sigma}_\theta(\mathbf{x}_t, t)\right)
\tag{Ho, Eq. 1}
$$

Ho et al. fixent la variance à $\boldsymbol{\Sigma}_\theta = \sigma_t^2 \mathbf{I}$
avec $\sigma_t^2 = \beta_t$ (non appris).
Seule la **moyenne** $\boldsymbol{\mu}_\theta$ est paramétrée par le réseau.

---

## 4. L'objectif d'entraînement

### 4.1 La borne variationnelle (ELBO)

L'entraînement minimise une borne supérieure de la log-vraisemblance négative (ELBO) :

$$
\begin{aligned}
\mathbb{E}\!\left[-\log p_\theta(\mathbf{x}_0)\right]
&\leq \mathbb{E}_q\!\left[
  \underbrace{D_\text{KL}(q(\mathbf{x}_T \mid \mathbf{x}_0) \,\|\, p(\mathbf{x}_T))}_{L_T}
\right. \\
&\quad + \sum_{t=2}^{T}
  \underbrace{D_\text{KL}(q(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0)
  \,\|\, p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t))}_{L_{t-1}} \\
&\quad \left. -\; \underbrace{\log p_\theta(\mathbf{x}_0 \mid \mathbf{x}_1)}_{L_0}
\right]
\end{aligned}
\tag{Ho, Eq. 5}
$$

- $L_T$ : KL entre la distribution finale du forward et $\mathcal{N}(\mathbf{0},\mathbf{I})$ Constant car $\beta_t$ est fixé → ignoré pendant l'entraînement.
- $L_{t-1}$ : KL entre la **posterior forward** et la transition reverse apprise. C'est le terme principal.
- $L_0$ : terme de reconstruction au dernier pas.

Les détails de calculs sont disponibles dans l'annexe A (Extended derivations) du papier de (Ho et al., 2015)

### 4.2 La posterior forward $q(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0)$

La posterior forward **conditionnée sur $\mathbf{x}_0$ s'écrit comme une gaussienne explicite** — sa moyenne $\tilde{\boldsymbol{\mu}}_t$ et sa variance $\tilde{\beta}_t$ ont des expressions exactes calculables directement à partir de $\mathbf{x}_0$, $\mathbf{x}_t$ et des paramètres du schedule $\bar{\alpha}_t$, sans approximation ni intégrale numérique :


$$q(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0) = \mathcal{N}\!\left(\mathbf{x}_{t-1};\; \tilde{\boldsymbol{\mu}}_t(\mathbf{x}_t, \mathbf{x}_0),\; \tilde{\beta}_t \mathbf{I}\right) \tag{Ho, Eq. 6}
$$

avec :

$$
\begin{aligned}
\tilde{\boldsymbol{\mu}}_t(\mathbf{x}_t, \mathbf{x}_0)
&:= \frac{\sqrt{\bar{\alpha}_{t-1}}\,\beta_t}{1 - \bar{\alpha}_t}\,\mathbf{x}_0 + \frac{\sqrt{\alpha_t}(1 - \bar{\alpha}_{t-1})}{1 - \bar{\alpha}_t}\,\mathbf{x}_t \\[6pt]
\tilde{\beta}_t
&:= \frac{1 - \bar{\alpha}_{t-1}}{1 - \bar{\alpha}_t}\,\beta_t
\end{aligned}
\tag{Ho, Eq. 7}
$$

<!--La preuve de cette formule (par règle de Bayes + propriétés de la loi normale)
est en [page détail →](/blog/pages/ddpm-posterior-derivation/).-->

### 4.3 Paramétrisation par la prédiction du bruit

Ho et al. montrent qu'il est plus efficace de paramétrer $\boldsymbol{\mu}_\theta$ **non pas** comme une prédiction directe de la moyenne, mais *via* la prédiction du bruit $\boldsymbol{\epsilon}$.

En substituant la reparamétrisation de l'Eq. (4) dans l'expression de $\tilde{\boldsymbol{\mu}}_t$, on obtient :

$$\boldsymbol{\mu}_\theta(\mathbf{x}_t, t) = \frac{1}{\sqrt{\alpha_t}}
\left(
  \mathbf{x}_t - \frac{\beta_t}{\sqrt{1 - \bar{\alpha}_t}}
    \,\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)
\right)
\tag{Ho, Eq. 11}
$$

où $\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)$ est le **réseau U-Net** décrit dans le [billet 1](/blog/2025/resnet-unet/), conditionné sur $t$ par les embeddings sinusoïdaux du [billet 2](/blog/2025/sinusoidal-embedding/).

### 4.4 LOSS simplifiée $L_\text{simple}$

En substituant l'Eq. (11) dans les termes $L_{t-1}$ de l'ELBO, et en simplifiant les poids, Ho et al. proposent la fonction de Loss suivante :

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

C'est la **MSE entre le bruit réel et le bruit prédit** — directement implémentée dans `compute_loss` :

```python
# ddpm_model.py — DDPMModel.compute_loss()
# L_simple (Ho et al., Eq. 14)
x_noisy, noise = self.diffuse(x0, t, noise=noise)    # Eq. 4
noise_pred      = self.predict_noise(x_noisy, t)     # eps_theta(x_t, t)
return torch.mean((noise - noise_pred) ** 2)         # MSE
```

> **Pourquoi $L_\text{simple}$ et non l'ELBO complet ?**
> L'ELBO pondère chaque terme $L_{t-1}$ par $\frac{\beta_t^2}{2\sigma_t^2 \alpha_t (1-\bar{\alpha}_t)}$ (Ho, Eq. 12). 
> Cette pondération défavorise les petits $t$ (bruit faible), ce qui nuit à la qualité des échantillons. $L_\text{simple}$ supprime cette pondération, donnant un poids égal à tous les pas de temps — résultat : meilleure qualité d'image en pratique (Ho et al., Table 2).

---

## 5. Algorithmes

### Algorithme 1 — Entraînement

$$
\begin{array}{l}
\textbf{Algorithme 1 — Entraînement} \\
\hline
\textbf{repeat} \\
\quad \mathbf{x}_0 \sim q(\mathbf{x}_0) \\
\quad t \sim \text{Uniform}(\{1, \ldots, T\}) \\
\quad \boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I}) \\
\quad \text{Descente de gradient sur} \\
\quad\quad \nabla_\theta \left\| \boldsymbol{\epsilon} - \boldsymbol{\epsilon}_\theta\!\left( \sqrt{\bar{\alpha}_t}\,\mathbf{x}_0 + \sqrt{1 - \bar{\alpha}_t}\,\boldsymbol{\epsilon},\; t \right) \right\|^2 \\
\textbf{until } \text{convergence} \\
\hline
\end{array}
$$

Implémentation dans `train_gz2_ddpm.py` :

```python
# train_gz2_ddpm.py — boucle d'entraînement
x0 = batch[0].to(device)
t  = torch.randint(0, ddpm.n_steps, (batch_size,), device=device)

with autocast():
    loss = ddpm.compute_loss(x0, t)   # L_simple, Eq. 14

scaler.scale(loss).backward()
```

### Algorithme 2 — Échantillonnage

$$
\begin{array}{l}
\textbf{Algorithme 2 — Échantillonnage} \\
\hline
\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I}) \\
\textbf{for } t = T, \ldots, 1 \textbf{ do} \\
\quad \mathbf{z} \sim \mathcal{N}(\mathbf{0}, \mathbf{I}) \text{ if } t > 1, \text{ else } \mathbf{z} = \mathbf{0} \\
\quad \mathbf{x}_{t-1} = \dfrac{1}{\sqrt{\alpha_t}} \left( \mathbf{x}_t - \dfrac{1 - \alpha_t}{\sqrt{1 - \bar{\alpha}_t}}\,\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t) \right) + \sigma_t\,\mathbf{z} \\
\textbf{end for} \\
\textbf{return } \mathbf{x}_0 \\
\hline
\end{array}
$$

Implémentation dans `ddpm_model.py` — `generate_samples()` :

```python
# ddpm_model.py — generate_samples()
for t in reversed(range(ddpm.n_steps)):
    t_batch    = torch.full((n_samples,), t, device=device, dtype=torch.long)
    noise_pred = ddpm.predict_noise(x, t_batch)        # eps_theta(x_t, t)
    alpha_t    = ddpm.alphas[t]
    alpha_bar_t = ddpm.alpha_bars[t]

    # Eq. 11 : mise à jour vers x_{t-1}
    x = (1.0 / alpha_t.sqrt()) * (
        x - (1.0 - alpha_t) / (1.0 - alpha_bar_t).sqrt() * noise_pred
    )
    if t > 0:
        sigma_t = ddpm.betas[t].sqrt()   # sigma_t^2 = beta_t (option 1)
        x = x + sigma_t * torch.randn_like(x)
```

---

## 6. Résumé des équations clés

| Équation | Formule | Référence |
|---|---|---|
| Transition forward | $q(\mathbf{x}_t \mid \mathbf{x}_{t-1}) = \mathcal{N}(\sqrt{1-\beta_t}\,\mathbf{x}_{t-1},\, \beta_t \mathbf{I})$ | Ho, Eq. 2 |
| Marginal forward | $q(\mathbf{x}_t \mid \mathbf{x}_0) = \mathcal{N}(\sqrt{\bar{\alpha}_t}\,\mathbf{x}_0,\, (1-\bar{\alpha}_t)\mathbf{I})$ | Ho, Eq. 4 |
| Posterior forward | $q(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{x}_0) = \mathcal{N}(\tilde{\boldsymbol{\mu}}_t,\, \tilde{\beta}_t \mathbf{I})$ | Ho, Eq. 6 |
| Transition reverse | $p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t) = \mathcal{N}(\boldsymbol{\mu}_\theta(\mathbf{x}_t, t),\, \sigma_t^2 \mathbf{I})$ | Ho, Eq. 1 |
| Moyenne reverse | $\boldsymbol{\mu}_\theta = \frac{1}{\sqrt{\alpha_t}}\!\left(\mathbf{x}_t - \frac{\beta_t}{\sqrt{1-\bar{\alpha}_t}}\boldsymbol{\epsilon}_\theta\right)$ | Ho, Eq. 11 |
| Objectif simplifié | $L_\text{simple} = \mathbb{E}\!\left[\|\boldsymbol{\epsilon} - \boldsymbol{\epsilon}_\theta(\sqrt{\bar{\alpha}_t}\mathbf{x}_0 + \sqrt{1-\bar{\alpha}_t}\boldsymbol{\epsilon},\, t)\|^2\right]$ | Ho, Eq. 14 |

---

## Références

1. Ho, J., Jain, A., & Abbeel, P. (2020).
   *Denoising Diffusion Probabilistic Models.* NeurIPS 2020.
   [arXiv:2006.11239](https://arxiv.org/abs/2006.11239)

2. Nichol, A., & Dhariwal, P. (2021).
   *Improved Denoising Diffusion Probabilistic Models.* ICML 2021.
   [arXiv:2102.09672](https://arxiv.org/abs/2102.09672)

---

*Prochain billet :
[Application à Galaxy Zoo 2 — résultats →](/blog/2025/ddpm-galaxyzoo2/)*
