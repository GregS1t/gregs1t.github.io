---
layout: post
title: "[data] Analyser ses données trail — Cartographier sa trace GPS"
date: 2026-01-15
description: >
  Ton fichier .fit contient des coordonnées GPS. Comment représenter la trace sur une carte ? 
tags: [tutoriel, trail, GPS, cartographie, folium, FIT, python, télédétection, data, running]
categories: [tutoriel, trail, data, running, FIT, python]
related_posts: false
toc:
  sidebar: left
math: true
---

*"Ta montre a enregistré 38 000 points GPS. Voilà comment les afficher sur une carte — et les rendre utiles."*

Le fichier `.fit` produit par ta montre trail est un fichier *binaire* qui encode beaucoup plus qu'une trace GPS. Il contient ta fréquence cardiaque, ta cadence, l'altitude barométrique, la puissance, la température... au rythme d'environ une mesure par seconde (on dit à 1 Hertz (Hz)).
Dans ce billet, je te propose d'afficher la trace de ta course ou de ton entrainement sur une carte interactive en utilisant la librairie Folium.
Là encore, il faut que tu aies prétraité les données avec le notebook 1 (lire les données).

---

## "Les semi-circles"

Le format FIT (*Flexible and Interoperable Data Transfer*) est un standard binaire développé par Garmin et maintenu par la société ANT+ depuis 2008. Il est aujourd'hui utilisé par la quasi-totalité des montres sport du marché (Garmin, Suunto, Polar, Coros...). Chaque fichier contient une série de **messages** structurés — un message `record` par seconde de course, un message `session` de résumé, des messages `lap`, etc.

Les coordonnées GPS dans un fichier FIT ne sont **pas stockées en degrés décimaux** mais en **semicircles** — une unité propriétaire qui encode la position sur un entier 32 bits signé.

La conversion est la suivante :

$$\text{latitude (°)} = \text{position\_lat} \times \frac{180}{2^{31}}$$

$$\text{longitude (°)} = \text{position\_long} \times \frac{180}{2^{31}}$$

Le facteur $2^{31} = 2\,147\,483\,648$ correspond au fait qu'un entier 32 bits signé couvre $[-2^{31}, 2^{31}-1]$, soit une plage de $2^{32}$ valeurs au total. En divisant la plage terrestre de 360° (ou 180° pour la latitude) par $2^{31}$, on obtient une résolution angulaire d'environ $8,4 \times 10^{-8}$ degrés, soit approximativement **0,009 mètre à l'équateur** — bien en dessous de la précision réelle du GPS, qui est de l'ordre du mètre.

En Python avec `fitparse` et `pandas`, c'est assez simple. Quand tu as lu les données de ton fichier, il te suffit d'ajouter deux colonnes supplémentaires :

```python
# Après chargement du FIT en DataFrame df
df["lat"] = df["position_lat"] * (180.0 / 2**31)
df["lon"] = df["position_long"] * (180.0 / 2**31)
```

Juste à titre de curiosité, les coordonnées qu'on obtient sont exprimées dans le "système géodésique" **WGS84** (*World Geodetic System 1984*), c'est-à-dire le référentiel utilisé par le GPS civil mondial, et le même que celui utilisé par Google Maps, OpenStreetMap, et la plupart des fonds cartographiques en ligne. C'est pratique pour nous qui voulons mettre notre trace sur la carte ! Et si vous voulez en savoir plus sur le système WGS84, j'ai ajouté une référence à la fin.

Et comme ça m'intriguait, j'ai cherché pourquoi Garmin avait choisi ce système d'encodage appelé *semicircles* avec un encodage sur 32 bits :

> Il semblerait qu'il y ait plusieurs raisons liées aux contraintes du matériel embarqué des années 1990–2000 : les processeurs GPS manquaient de coprocesseur virgule flottante (les entiers sont 5–10× plus rapides à calculer), la mémoire flash était précieuse (32 bits vs 64 bits = moitié de l'espace), et un entier 32 bits offre une meilleure précision qu'un flottant 32 bits pour les coordonnées géographiques. 
> Le nom "semicircle" vient du fait que $\pi$ radians = $2^{31}$ semicircles — un demi-cercle couvre exactement $180°$, la plage d'une latitude.

<!--
WGS84 définit un ellipsoïde de référence (demi-grand axe $a = 6\,378\,137$ m, aplatissement $f = 1/298,257$) sur lequel la latitude et la longitude sont exprimées en degrés décimaux. L'altitude dans le FIT est une altitude **ellipsoïdale** (hauteur au-dessus de l'ellipsoïde WGS84), distincte de l'altitude orthométrique (hauteur au-dessus du géoïde) utilisée sur les cartes topographiques. L'écart entre les deux, appelé ondulation du géoïde, varie de −105 m à +85 m selon la zone géographique.
-->

---

## Afficher la trace avec Folium

[Folium](https://python-visualization.github.io/folium/) est une librairie Python qui génère des cartes interactives en HTML/JavaScript en utilisant la bibliothèque [Leaflet.js](https://leafletjs.com/). Les fonds cartographiques sont des **tuiles raster** servies par des fournisseurs en ligne (OpenStreetMap, CartoDB, Stamen...) — le même principe que Google Maps.
C'est une bibliothèque très utilisée dans le domaine de la télédétection par les programmeurs en Python.

N'oubliez pas d'installer la librairie Folium :

```bash
pip install folium
```

Une carte folium de base avec la trace GPS :
> On cherche le centre de la carte avec les coordonnées moyennes de la trace.
> La trace est constituée d'un petit polygone par point.

```python
import folium
import pandas as pd

# df contient les colonnes lat, lon (en degrés décimaux)
center = [df["lat"].mean(), df["lon"].mean()]
m = folium.Map(location=center, zoom_start=12, tiles="CartoDB positron")

# Tracer la route en une seule polyligne
coords = list(zip(df["lat"], df["lon"]))
folium.PolyLine(coords, color="steelblue", weight=3, opacity=0.85).add_to(m)

# Afficher dans Jupyter
m
```

---

## Marqueurs de départ et d'arrivée

La première chose qu'on cherche sur une carte de course, c'est où elle commence et où elle finit. Folium permet d'utiliser des icônes [FontAwesome](https://fontawesome.com/icons) pour différencier visuellement les deux points.

Le départ correspond au premier point GPS valide du DataFrame, l'arrivée au dernier. On utilise deux icônes distinctes : un drapeau vert pour le départ, un drapeau à damier (ou une maison) pour l'arrivée.

```python
def add_start_finish(m, df):
    """Add start and finish markers to a Folium map."""
    start = df.iloc[0]
    finish = df.iloc[-1]

    folium.Marker(
        location=[start["lat"], start["lon"]],
        popup="Départ",
        icon=folium.Icon(color="green", icon="flag", prefix="fa"),
    ).add_to(m)

    folium.Marker(
        location=[finish["lat"], finish["lon"]],
        popup="Arrivée",
        icon=folium.Icon(color="red", icon="flag-checkered", prefix="fa"),
    ).add_to(m)

    return m

m = add_start_finish(m, df)
m
```

> **Note** : `df.iloc[0]` et `df.iloc[-1]` supposent que le DataFrame est trié chronologiquement — ce qui est le cas si tu as utilisé le notebook de prétraitement. Si ce n'est pas le cas, un `df.sort_values("timestamp").reset_index(drop=True)` suffit.

---

## Colorier la trace par variable physio (FC, pente...)

L'intérêt de l'approche GPS + données physio est de **colorier la trace segment par segment** selon une variable au choix (FC, vitesse, pente...). C'est ce que font Strava et Garmin Connect de façon propriétaire — on peut le reproduire avec quelques dizaines de lignes.

Le principe : on découpe la trace en segments de deux points consécutifs, et on attribue à chaque segment la couleur correspondant à la valeur de la variable au premier point.

```python
import matplotlib as mpl
import matplotlib.pyplot as plt

def plot_map_colored(df, color_col, cmap_name="RdYlGn_r"):
    """Trace GPS colorée par une colonne du DataFrame."""
    col_vals = df[color_col].to_numpy(dtype=float)
    vmin = float(pd.Series(col_vals).quantile(0.02))
    vmax = float(pd.Series(col_vals).quantile(0.98))

    cmap = plt.get_cmap(cmap_name)
    norm = mpl.colors.Normalize(vmin=vmin, vmax=vmax)

    center = [df["lat"].mean(), df["lon"].mean()]
    m = folium.Map(location=center, zoom_start=12, tiles="CartoDB positron")

    coords = list(zip(df["lat"], df["lon"]))
    vals = col_vals

    for i in range(len(coords) - 1):
        v = vals[i]
        if pd.isna(v):
            continue
        hex_color = mpl.colors.to_hex(cmap(norm(v)))
        folium.PolyLine(
            [coords[i], coords[i + 1]],
            color=hex_color,
            weight=3,
            opacity=0.85,
        ).add_to(m)

    return m

# Trace colorée par FC - tu mets le champ que tu veux
m = plot_map_colored(df, "heart_rate", cmap_name="RdYlGn_r")
m
```

La colormap `RdYlGn_r` (rouge → jaune → vert, inversée) donne rouge pour les FC élevées et vert pour les FC basses — convention intuitive.

---

## Ajouter les ravitaillements

Si tu connais les ravitaillements (souvent en kilomètres). Tu n'as qu'à trouver le point GPS le plus proche dans le DataFrame en cherchant l'index qui minimise `|dist_m/1000 - ravito_km|` :
Tu fais ça comme ça en Pandas !

```python
idx = (df["dist_m"] / 1000.0 - km).abs().idxmin()
```

Le code complet, ça donne ça :

```python
RAVITO_KM  = [19.2, 34.0, 45.0, 58.8, 65.4]
RAVITO_NOM = ["St Christo", "Ste Catherine", "St Genou", "Soucieu", "Chaponost"]

for km, nom in zip(RAVITO_KM, RAVITO_NOM):
    idx = (df["dist_m"] / 1000.0 - km).abs().idxmin()
    folium.Marker(
        location=[df.loc[idx, "lat"], df.loc[idx, "lon"]],
        popup=f"{nom} ({km} km)",
        icon=folium.Icon(color="blue", icon="cutlery", prefix="fa"),
    ).add_to(m)
```

Et l'icône `fa-cutlery`, c'est une icône FontAwesome intégrée à folium. La liste complète est disponible sur [fontawesome.com](https://fontawesome.com/icons).

---

## Profil altimétrique en incrustation

La carte montre *où* tu es passé. Le profil altimétrique montre *comment* le terrain s'est présenté. Les afficher ensemble, sans quitter la carte, est la combinaison naturelle. C'est une info qu'on trouve couramment sur les sites [Tracedetrail](https://tracedetrail.fr/), [OpenRunner](https://www.openrunner.com)...

L'idée est d'injecter un graphe SVG directement dans le DOM de la carte Folium, en utilisant `branca.element.MacroElement`. Pas de fichier image externe, pas de serveur — le graphe est généré par matplotlib, converti en SVG en mémoire via `io.BytesIO`, puis inséré comme un élément HTML flottant positionné en bas de la carte.

```python
import io
import base64
import branca
from branca.element import MacroElement, Figure
from jinja2 import Template

def make_elevation_svg(df, dist_col="dist_m", alt_col="altitude",
                       figsize=(5, 1.6), dpi=110):
    """Generate an elevation profile as a base64-encoded SVG string."""
    dist_km = df[dist_col] / 1000.0
    alt = df[alt_col]

    fig, ax = plt.subplots(figsize=figsize, dpi=dpi)
    ax.fill_between(dist_km, alt, alpha=0.35, color="steelblue")
    ax.plot(dist_km, alt, color="steelblue", linewidth=1.2)
    ax.set_xlabel("Distance (km)", fontsize=7)
    ax.set_ylabel("Altitude (m)", fontsize=7)
    ax.tick_params(labelsize=6)
    ax.grid(True, linestyle="--", alpha=0.4)
    fig.tight_layout(pad=0.4)

    buf = io.BytesIO()
    fig.savefig(buf, format="svg", bbox_inches="tight")
    plt.close(fig)
    buf.seek(0)
    svg_b64 = base64.b64encode(buf.read()).decode("utf-8")
    return svg_b64
```

On crée ensuite un `MacroElement` qui injecte un `<div>` positionné en absolu dans le coin inférieur gauche de la carte :

{% raw %}
```python
class ElevationControl(MacroElement):
    """Folium MacroElement that overlays an elevation profile on the map."""

    def __init__(self, svg_b64):
        """Init with a base64-encoded SVG image."""
        super().__init__()
        self._name = "ElevationControl"
        self.svg_b64 = svg_b64
        self._template = Template("""
            {% macro script(this, kwargs) %}
            (function() {
                var div = document.createElement('div');
                div.style.cssText = [
                    'position:absolute',
                    'bottom:30px',
                    'left:10px',
                    'z-index:1000',
                    'background:rgba(255,255,255,0.88)',
                    'border-radius:6px',
                    'padding:6px',
                    'box-shadow:0 1px 5px rgba(0,0,0,0.3)'
                ].join(';');
                div.innerHTML = '<img src="data:image/svg+xml;base64,{{ this.svg_b64 }}"'
                    + ' width="320" style="display:block;">';
                document.querySelector('#{{ this._parent.get_name() }}').appendChild(div);
            })();
            {% endmacro %}
        """)

svg_b64 = make_elevation_svg(df)
ElevationControl(svg_b64).add_to(m)
m
```
{% endraw %}

Quelques points techniques à noter :

- `MacroElement` est la brique de base de branca pour injecter du JavaScript arbitraire dans une carte Folium. La méthode `script` est appelée après le rendu de la carte.
- Le graphe est encodé en **base64** et inséré comme `data URI` dans un `<img>` — aucune dépendance externe, le fichier HTML reste autoportant.
- `{% raw %}{{ this._parent.get_name() }}{% endraw %}` récupère l'identifiant unique du `<div>` de la carte Leaflet, ce qui permet de cibler précisément le bon élément dans le DOM si plusieurs cartes coexistent dans le notebook.

> Pour un notebook avec plusieurs courses, il suffit de passer un `df` différent à `make_elevation_svg` — chaque carte embarque son propre profil.

---

## Un petit plus : la trace animée avec "AntPath"

Jusqu'ici, la carte est statique. [AntPath](https://github.com/rubenspgcavalcante/leaflet-ant-path) est un plugin Leaflet qui anime la trace avec des "fourmis" se déplaçant dans le sens de course — une façon immédiate de visualiser la direction du parcours et de rendre la carte vivante.

>Non, ce n'est pas du tout indispensable, c'est juste que c'est joli. 

Folium intègre AntPath nativement via `folium.plugins.AntPath` :

```python
from folium.plugins import AntPath

def add_ant_path(m, df, color="steelblue", weight=4, delay=800):
    """Add an animated AntPath trace to a Folium map."""
    coords = list(zip(df["lat"], df["lon"]))
    AntPath(
        locations=coords,
        color=color,
        weight=weight,
        delay=delay,        # ms entre deux pas d'animation
        dash_array=[10, 20],
        pulse_color="#ffffff",
    ).add_to(m)
    return m

m = add_ant_path(m, df)
m
```

Les paramètres clés :

| Paramètre | Rôle | Valeur typique |
|---|---|---|
| `delay` | Vitesse d'animation (ms) | 400 (rapide) → 1200 (lent) |
| `dash_array` | Longueur trait / espace | `[10, 20]` |
| `pulse_color` | Couleur des "fourmis" | `"#ffffff"` |
| `weight` | Épaisseur de la ligne | 3–5 |

> **Remarque** : AntPath et `PolyLine` peuvent coexister sur la même carte. Une pratique courante est d'afficher la `PolyLine` colorée par FC (section précédente) *et* d'ajouter l'AntPath par-dessus avec une opacité réduite (`opacity=0.5`) pour conserver la lecture analytique tout en ajoutant l'animation.

---

## Assembler le tout

Voici la fonction finale qui combine tous les éléments vus dans ce billet :

```python
def build_race_map(df, ravito_km=None, ravito_nom=None,
                   color_col="heart_rate", cmap_name="RdYlGn_r",
                   animated=True):
    """Build a complete interactive race map with all overlays.

    Parameters
    ----------
    df : DataFrame with lat, lon, dist_m, altitude, heart_rate columns.
    ravito_km : list of float, aid station distances in km.
    ravito_nom : list of str, aid station names.
    color_col : str, column used for trace coloring.
    cmap_name : str, matplotlib colormap name.
    animated : bool, whether to add AntPath animation.
    """
    m = plot_map_colored(df, color_col, cmap_name)
    m = add_start_finish(m, df)

    if ravito_km and ravito_nom:
        for km, nom in zip(ravito_km, ravito_nom):
            idx = (df["dist_m"] / 1000.0 - km).abs().idxmin()
            folium.Marker(
                location=[df.loc[idx, "lat"], df.loc[idx, "lon"]],
                popup=f"{nom} ({km} km)",
                icon=folium.Icon(color="blue", icon="cutlery", prefix="fa"),
            ).add_to(m)

    svg_b64 = make_elevation_svg(df)
    ElevationControl(svg_b64).add_to(m)

    if animated:
        m = add_ant_path(m, df)

    return m


# Exemple d'utilisation
RAVITO_KM  = [19.2, 34.0, 45.0, 58.8, 65.4]
RAVITO_NOM = ["St Christo", "Ste Catherine", "St Genou", "Soucieu", "Chaponost"]

m = build_race_map(df, ravito_km=RAVITO_KM, ravito_nom=RAVITO_NOM)
m
```

## Le résultat en image 

<figure style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/img/blog/2025-12_analyse_data_trail/Profil_Folio_Ecotrail2026.png"
       width="80%"
       alt="Exemple de carte de profil de course pour l'Ecotrail 2026">
</figure>


---

## Petit lien avec la télédétection

Pour les étudiants en télédétection qui arrivent sur cet article (on ne sait jamais) : la chaîne que l'on vient de construire est un pipeline géospatial complet, simplifié mais fonctionnel.

- **Acquisition** : capteur GPS embarqué, enregistrement en WGS84, stockage binaire FIT
- **Prétraitement** : conversion d'unités (semicircles → degrés), nettoyage (distance monotone, filtre altitude)
- **Fusion de données** : jointure temporelle entre les coordonnées GPS et les mesures physiologiques (FC, puissance, cadence) — c'est exactement ce qu'on fait en télédétection quand on fusionne une image satellite avec des données terrain
- **Visualisation** : projection sur fond cartographique, symbolisation thématique par variable quantitative, incrustation de graphes, animation temporelle

La principale différence avec la télédétection classique est que la donnée est **ponctuelle et mobile** (une trace) plutôt que **surfacique et statique** (une image). Mais les outils sont les mêmes : systèmes de référence, projections, visualisation thématique — et maintenant, animation.

---

## Pour jouer avec tes données 

Le notebook complet associé à ce billet est disponible sur [GitHub](https://github.com/GregS1t/trail-lab/).

## Petit warning 

>Avertissement : je suis data scientist, pas spécialiste de physiologie de l'exercice. 
>Ce que tu lis ici, c'est le carnet de bord d'un trailer curieux qui aime comprendre 
>ses données — pas un conseil médical ou d'entraînement.
>Les analyses sont fournies à titre informatif uniquement ; je décline toute >responsabilité quant à leur usage. Les sources sont là pour que tu puisses vérifier >par toi-même.

---

## Références

- ANT+ / Garmin. FIT Protocol SDK. [developer.garmin.com](https://developer.garmin.com/fit/protocol/)
- Python-fitparse. [github.com/dtcooper/python-fitparse](https://github.com/dtcooper/python-fitparse)
- Folium documentation. [python-visualization.github.io/folium](https://python-visualization.github.io/folium/)
- Branca documentation. [python-visualization.github.io/branca](https://python-visualization.github.io/branca/)
- AntPath plugin. [github.com/rubenspgcavalcante/leaflet-ant-path](https://github.com/rubenspgcavalcante/leaflet-ant-path)
- EPSG Registry. Système WGS84 (EPSG:4326). [epsg.io/4326](https://epsg.io/4326)



