---
layout: post
title: "Analyser ses données trail (3/3) — Lire sa physiologie"
date: 2026-04-01
description: "Dérive cardiaque, TRIMP, analyse croisée pente × FC, dégradation d'allure GAP-normalisée et heatmap. Ce que Garmin Connect ne calcule pas."
tags: trail données python analyse FC dérive TRIMP physiologie fatigue Minetti
categories: trail
---

*"Tu peux voir ta fréquence cardiaque sur Garmin Connect. Ce que tu ne peux pas y faire, c'est croiser ta dérive cardiaque avec le profil, calculer ta charge en TRIMP, ou comparer ta courbe pente/allure avec le modèle de Minetti. C'est ce qu'on fait ici."*

Ce troisième billet conclut la série. On suppose que tu as lu et exécuté les deux premiers — le DataFrame `df` contient déjà `slope_pct`, `gap_s_per_km`, `is_walk`, et les colonnes de base.

---

## 1. Dérive cardiaque

Sur un effort long, la fréquence cardiaque a tendance à monter progressivement, même si la vitesse reste constante. C'est la **dérive cardiaque** — un signe de fatigue cardiovasculaire, de déshydratation, ou de montée en température corporelle.

Le moyen le plus simple de la mesurer depuis tes données FIT : calculer le ratio FC / vitesse. Si l'effort est constant mais que la FC monte, ce ratio augmente. On le lisse sur une fenêtre de 5 minutes pour filtrer les variations dues au profil.

```python
def compute_cardiac_drift(df, smoothing_sec=300):
    """Compute smoothed HR/speed ratio as a cardiac drift proxy."""
    df = df.copy()
    spd = df["speed_kmh"].replace(0, np.nan)
    df["hr_speed_ratio"] = df["heart_rate"] / spd

    # Lissage sur fenêtre temporelle (5 min par défaut)
    df_t = df.set_index("timestamp")
    df["hr_speed_smooth"] = (
        df_t["hr_speed_ratio"]
        .rolling(f"{smoothing_sec}s", min_periods=30)
        .mean()
        .values
    )
    return df


df = compute_cardiac_drift(df)

fig, ax = plt.subplots(figsize=(13, 4))
ax.plot(df["time_h"], df["hr_speed_smooth"], color="purple", linewidth=1.5)

# Ravitos en référence
if len(RAVITO_KM) > 0:
    for km, nom in zip(RAVITO_KM, RAVITO_NOM):
        t_rav = df.loc[(df["dist_m"] / 1000.0 - km).abs().idxmin(), "time_h"]
        ax.axvline(t_rav, color="grey", linestyle=":", alpha=0.5, linewidth=1)
        ax.text(t_rav, ax.get_ylim()[1], nom, fontsize=7,
                rotation=90, va="top", ha="right", color="grey")

ax.set_xlabel("Temps (h)")
ax.set_ylabel("FC / vitesse — lissé 5 min")
ax.set_title("Dérive cardiaque")
ax.grid(True)
plt.tight_layout()
plt.show()
```

Une courbe qui monte régulièrement, même sur des sections de profil comparable, indique que le cœur travaille de plus en plus pour maintenir le même effort. Si la courbe fait un saut net à partir d'un certain ravito, c'est souvent le moment où quelque chose s'est dégradé — digestif, hydratation, chaleur.

---

## 2. Dégradation d'allure GAP-normalisée

L'allure brute se dégrade naturellement sur la deuxième moitié d'un ultra, mais une partie de cette dégradation est simplement due au profil — il est souvent plus difficile en fin de course. Pour isoler la **vraie fatigue** du profil, on compare le GAP (allure corrigée du dénivelé) section par section.

```python
def gap_drift(df, ravito_km, ravito_nom, n_bins=20):
    """Compute rolling median GAP to visualize pace degradation independent of profile."""
    df = df.copy()
    df["dist_km"] = df["dist_m"] / 1000.0
    df["dist_bin"] = pd.cut(df["dist_km"],
                            bins=np.linspace(0, df["dist_km"].max(), n_bins + 1))

    gap_by_bin = (
        df.dropna(subset=["gap_s_per_km"])
        .groupby("dist_bin", observed=True)["gap_s_per_km"]
        .median()
    )
    bin_centers = gap_by_bin.index.map(lambda x: x.mid)

    fig, ax = plt.subplots(figsize=(13, 4))
    ax.plot(bin_centers, gap_by_bin.values, color="darkorange",
            linewidth=2, marker="o", markersize=4)

    for km, nom in zip(ravito_km, ravito_nom):
        ax.axvline(km, color="grey", linestyle=":", alpha=0.5, linewidth=1)
        ax.text(km, ax.get_ylim()[1], nom, fontsize=7,
                rotation=90, va="top", ha="right", color="grey")

    ax.set_xlabel("Distance (km)")
    ax.set_ylabel("GAP médian (s/km)")
    ax.set_title("Dégradation d'allure GAP-normalisée\n"
                 "(une hausse = ralentissement non expliqué par le profil)")
    ax.invert_yaxis()   # convention : allure rapide en haut
    ax.grid(True)
    plt.tight_layout()
    plt.show()


gap_drift(df, RAVITO_KM, RAVITO_NOM)
```

L'axe y est inversé par convention : les allures rapides (faibles valeurs en s/km) sont en haut. Une montée de cette courbe au fil des kilomètres, indépendante des montées du profil, est la signature de la fatigue cumulée.

---

## 3. Analyse croisée pente ↔ FC ↔ allure

On regroupe par bins de pente et on calcule les médianes (robuste aux outliers). C'est l'analyse qui permet de construire ta **courbe de coût individuelle** — comparable au polynôme de Minetti.

```python
SLOPE_BINS = [-30, -15, -10, -7, -5, -3, -1, 1, 3, 5, 7, 10, 15, 30]


def slope_crossanalysis(df, bins=SLOPE_BINS):
    """Aggregate pace and HR by slope bin (median)."""
    df = df.copy()
    df["slope_bin"] = pd.cut(df["slope_pct"], bins=bins)

    cols = ["slope_pct", "gap_s_per_km"]
    if "heart_rate" in df.columns:
        cols.append("heart_rate")

    tmp = df.dropna(subset=cols)
    agg = {
        "n": ("slope_pct", "size"),
        "slope_med": ("slope_pct", "median"),
        "gap_med": ("gap_s_per_km", "median"),
    }
    if "heart_rate" in df.columns:
        agg["hr_med"] = ("heart_rate", "median")

    return tmp.groupby("slope_bin", observed=True).agg(**agg).reset_index()


summary = slope_crossanalysis(df)
print(summary.to_string(index=False))
```

```python
# Courbe individuelle vs polynôme de Minetti
def minetti_gap_curve(slope_range, pace_flat_s_km):
    """Predicted GAP from Minetti polynomial for a given flat pace."""
    i = slope_range / 100.0
    cr = (155.4 * i**5 - 30.4 * i**4 - 43.3 * i**3
          + 46.3 * i**2 + 19.5 * i + 3.6)
    ratio = np.clip(cr / 3.6, 0.1, None)
    return pace_flat_s_km * ratio


slopes_ref = np.linspace(-20, 25, 100)
# Allure de référence sur le plat estimée depuis la médiane du GAP
pace_flat_est = float(summary.loc[
    summary["slope_med"].abs() < 3, "gap_med"
].median())

ncols = 2 if "heart_rate" in df.columns else 1
fig, axes = plt.subplots(1, ncols, figsize=(6 * ncols, 5))
if ncols == 1:
    axes = [axes]

# GAP vs pente
axes[0].plot(summary["slope_med"], summary["gap_med"],
             marker="o", label="Données réelles", color="darkorange")
axes[0].plot(slopes_ref,
             minetti_gap_curve(slopes_ref, pace_flat_est),
             linestyle="--", color="steelblue", label="Minetti (2002)")
axes[0].set_xlabel("Pente médiane (%)")
axes[0].set_ylabel("GAP médian (s/km)")
axes[0].set_title("Courbe GAP individuelle vs Minetti")
axes[0].legend()
axes[0].grid(True)

# FC vs pente
if "heart_rate" in df.columns and "hr_med" in summary.columns:
    axes[1].plot(summary["slope_med"], summary["hr_med"],
                 marker="o", color="crimson")
    axes[1].set_xlabel("Pente médiane (%)")
    axes[1].set_ylabel("FC médiane (bpm)")
    axes[1].set_title("FC vs pente")
    axes[1].grid(True)

plt.tight_layout()
plt.show()
```

La comparaison entre ta courbe réelle (orange) et Minetti (bleu pointillé) est directement interprétable. Si tu es systématiquement au-dessus de Minetti en descente, tu freines plus que la moyenne de son échantillon. Si tu es en dessous en montée, ton économie de course en côte est meilleure que la moyenne du modèle — ou bien tu t'es moins épargné.

---

## 4. TRIMP — quantifier la charge d'entraînement

Le TRIMP (Training IMPulse, Banister 1991) est une façon de résumer la charge d'une sortie en une seule valeur, comparable entre des sessions de durée et d'intensité différentes. La formule prend en compte la durée, la FC de réserve (FC moyenne normalisée entre FC repos et FC max), et un facteur exponentiel qui pénalise davantage les zones hautes.

```python
def compute_trimp(df, fc_min, fc_max, hr_col="heart_rate", sexe="H"):
    """Compute TRIMP (Banister 1991) from HR time series.

    sexe : 'H' (homme, facteur 1.92) ou 'F' (femme, facteur 1.67).
    """
    b = 1.92 if sexe == "H" else 1.67

    tmp = df.dropna(subset=[hr_col, "timestamp"]).copy()
    tmp = tmp.sort_values("timestamp")

    # Durée de chaque point en minutes (différence de timestamp)
    dt_min = tmp["timestamp"].diff().dt.total_seconds().fillna(1.0) / 60.0

    # FC de réserve normalisée
    hrr = (tmp[hr_col] - fc_min) / (fc_max - fc_min)
    hrr = hrr.clip(0, 1)

    # Contribution TRIMP de chaque point
    trimp_increments = dt_min * hrr * np.exp(b * hrr)
    return float(trimp_increments.sum())


trimp = compute_trimp(df, FC_MIN, FC_MAX)
print(f"TRIMP de la sortie : {trimp:.0f}")
print()
print("Repères indicatifs (hommes, valeurs très variables selon l'individu) :")
print("  Sortie facile 1h Z2       : ~50–80")
print("  Sortie longue 3h endurance : ~150–250")
print("  Course de 80 km            : ~400–700")
print("  Ultra 160 km               : ~700–1200+")
```

Le TRIMP est une approximation — la même valeur peut correspondre à des morphologies de course très différentes. Sa vraie utilité est la **comparaison longitudinale** : sur plusieurs semaines, tracer l'évolution du TRIMP par sortie pour visualiser la charge d'entraînement. C'est ce que font des outils comme Runalyze (qui l'intègre nativement) ou TrainingPeaks (sous le nom ATL/CTL).

```python
# TRIMP par section entre ravitos
if len(RAVITO_KM) > 0:
    bounds = np.concatenate((
        [float(df["dist_m"].min() / 1000.0)],
        np.array(RAVITO_KM, dtype=float),
        [float(df["dist_m"].max() / 1000.0)]
    ))
    labels_sec = []
    for i in range(len(bounds) - 1):
        if i == 0:
            labels_sec.append(f"Départ → {RAVITO_NOM[0]}")
        elif i == len(bounds) - 2:
            labels_sec.append(f"{RAVITO_NOM[-1]} → Arrivée")
        else:
            labels_sec.append(f"{RAVITO_NOM[i-1]} → {RAVITO_NOM[i]}")

    df["section_id"] = np.searchsorted(
        bounds[1:], df["dist_m"].to_numpy() / 1000.0, side="right"
    )

    trimp_by_section = {}
    for i, lbl in enumerate(labels_sec):
        sec = df[df["section_id"] == i]
        if len(sec) > 10:
            trimp_by_section[lbl] = round(compute_trimp(sec, FC_MIN, FC_MAX), 1)

    trimp_df = pd.DataFrame(trimp_by_section.items(),
                            columns=["Section", "TRIMP"])
    print(trimp_df.to_string(index=False))
```

---

## 5. Heatmap vitesse × pente → FC médiane

La heatmap est la visualisation la plus dense de ce billet. Elle croise deux variables continues (vitesse et pente) et encode une troisième (FC médiane) en couleur. Tu obtiens une carte de ton effort physiologique sur l'ensemble du parcours.

```python
def plot_heatmap_speed_slope_hr(df, fc_max):
    """2D heatmap: speed (x) × slope (y) → median HR (color)."""
    req = ["speed_kmh", "slope_pct", "heart_rate"]
    dh = df.dropna(subset=req).copy()

    speed_bins = np.arange(0, 18.5, 0.5)
    slope_bins = np.arange(-25, 26, 1.0)

    sidx = np.digitize(dh["speed_kmh"].to_numpy(), speed_bins) - 1
    pidx = np.digitize(dh["slope_pct"].to_numpy(), slope_bins) - 1

    sumhr = np.zeros((len(slope_bins), len(speed_bins)))
    count = np.zeros_like(sumhr)

    for si, pi, hr in zip(sidx, pidx, dh["heart_rate"].to_numpy()):
        if 0 <= pi < len(slope_bins) and 0 <= si < len(speed_bins):
            sumhr[pi, si] += hr
            count[pi, si] += 1

    with np.errstate(invalid="ignore"):
        heat = np.where(count > 5, sumhr / count, np.nan)

    fig, ax = plt.subplots(figsize=(11, 7))
    im = ax.imshow(
        heat,
        aspect="auto",
        origin="lower",
        extent=[0, 18, -25, 25],
        cmap="RdYlGn_r",
        vmin=fc_max * 0.55,
        vmax=fc_max * 0.95
    )
    plt.colorbar(im, ax=ax, label="FC médiane (bpm)")
    ax.axhline(0, color="white", linewidth=0.8, linestyle="--", alpha=0.4)
    ax.set_xlabel("Vitesse (km/h)")
    ax.set_ylabel("Pente (%)")
    ax.set_title("FC médiane par (vitesse × pente)\n"
                 "Cellules avec < 5 points masquées")
    fig.tight_layout()
    plt.show()


if "heart_rate" in df.columns:
    plot_heatmap_speed_slope_hr(df, FC_MAX)
```

Comment lire cette carte : les cellules bleues/vertes sont les combinaisons vitesse/pente où ta FC est basse (effort facile ou absent) ; les cellules rouges sont les zones de fort engagement cardiaque. Les zones denses (beaucoup de points) correspondent à ton rythme de croisière habituel sur ce type de terrain.

Deux lectures pratiques intéressantes : la **diagonale montante** (plus tu vas vite sur le plat, plus ta FC monte — cohérent) et l'**asymétrie montée/descente** — à même vitesse, la montée engage plus le cardio que la descente, où la limitation est plutôt musculaire.

---

## Ce qu'on n'a pas fait

Cette série couvre l'analyse d'une seule course. La vraie valeur de ces outils apparaît sur la **durée** : comparer le TRIMP semaine après semaine, voir si le seuil de marche s'est abaissé entre deux compétitions, ou si le GAP médian à même effort s'est amélioré après un bloc de préparation spécifique.

C'est l'étape d'après — agréger plusieurs FIT, construire un historique, et commencer à voir les tendances d'entraînement. Mais ça, c'est pour une autre série.
