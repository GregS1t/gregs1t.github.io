---
layout: post
title: "[data] La météo comme donnée d'entraînement — comment la récupérer et l'exploiter"
date: 2026-04-05
description: >
  Ta montre enregistre la température, mais ce qu'elle mesure c'est en partie la chaleur de ton poignet. Comment récupérer la météo réelle de ta course ?
tags: [trail, météo, python, open-meteo, ERA5, température, humidité, analyse, running]
categories: trail
related_posts: false
toc:
  sidebar: left
math: true
---

*"J'avais trop chaud sur la deuxième partie. Mais est-ce que c'est vraiment la température qui a joué ?"*

Sur n'importe quelle course, les conditions thermiques influencent directement la physiologie. La dérive cardiaque peut s'expliquer autant par la chaleur que par la fatigue musculaire. Mais pour le savoir, il faut la météo réelle — pas l'estimation de ta montre.

>Au passage, je conseille la lecture de l'article ou l'écoute du podcast sur Courir-mieux à propos de l'acclimatation à la chaleur : [Chaleur et trail : Comment s’acclimater ? (2025)](https://courir-mieux.fr/acclimatation-chaleur-trail/) 

Bref, retour à la data...

---
 
## Le problème avec la température de ta montre

La colonne `temperature` dans un fichier FIT est mesurée par un capteur intégré dans le boîtier de la montre. Sur le poignet, ce capteur reçoit à la fois la chaleur ambiante et la chaleur corporelle conduite par le tissu cutané. Du coup, les données sont faussée. Ce biais est systématique et documenté :

- Au repos, la température montre est proche de la température ambiante.
- À l'effort, le biais monte à **+2 à +5°C** selon le modèle, la position du bracelet et l'intensité de l'effort.
- Le biais est plus fort en montée (vasodilatation cutanée) qu'en descente ou aux ravitos.

La montre est donc utile pour observer les **variations** de température au fil du parcours mais ses valeurs absolues ne sont pas fiables pour comparer à des seuils physiologiques.

---

## La solution : ERA5-Land via Open-Meteo

[Open-Meteo](https://open-meteo.com) est une API (*Application Programming Interface* -  interface de programmation d’application) météo **gratuite**, destinée à la recherche et aux usages non commerciaux. 

On utlise le modèle **ERA5-Land**. C'est une réanalyse climatique qui assimile des millions de mesures *in-situ*(stations météo, radiosondages, satellites, bouées) dans un modèle physique. Elle est recalculée *a posteriori*, ce qui améliore l'homogénéïté des data.

Ses caractéristiques pour notre usage :

| Paramètre | Valeur |
|---|---|
| Résolution spatiale | 0,1° × 0,1° ≈ **9 km** |
| Résolution temporelle | **1 heure** |
| Disponibilité | **2001 à aujourd'hui** (ERA5-Land) |
| Couverture | Mondiale |
| Accès | Gratuit, sans inscription |

Avec 9 km de résolution, ERA5-Land ne capture pas les effets micro-locaux (fond de vallon froid, effet de vent sur une crête), mais elle fournit une référence bien supérieure à la mesure de la montre.

---

## Et comment je récupère les données ? 

Le point d'entrée (on dit *endpoint*) pour les données historiques est `https://archive.open-meteo.com/v1/archive`. 

La page Internet fourni le code Python à utilisé. Après quelques adataptations, voiçi ce que je propose : 

```python
import requests
import pandas as pd
import openmeteo_requests

def fetch_weather_hourly(lat, lon, date_str, timezone="Europe/Paris",
                         client=None, model="era5_land", date_end=None):

    if client is None:
        client = openmeteo_requests.Client()

    end_str = date_end if date_end is not None else date_str

    params = {                             # Les paramètres que je récupère
        "latitude":        lat,
        "longitude":       lon,
        "start_date":      date_str,
        "end_date":        end_str,
        "hourly":          _HOURLY_VARS,
        "wind_speed_unit": "kmh",
        "timezone":        timezone,
        "models":          model,
    }

    try:
        responses = client.weather_api(_ARCHIVE_URL, params=params)
        r = responses[0]
        hourly = r.Hourly()

        # Reconstruction du DataFrame depuis le buffer FlatBuffers.
        # Le SDK retourne toujours des timestamps UTC (secondes epoch).
        times = pd.date_range(
            start=pd.to_datetime(hourly.Time(),    unit="s", utc=True),
            end=pd.to_datetime(hourly.TimeEnd(),   unit="s", utc=True),
            freq=pd.Timedelta(seconds=hourly.Interval()),
            inclusive="left",
        )

        def _safe_var(i):
            """Extract variable i; return NaN array if unavailable."""
            try:
                arr = hourly.Variables(i).ValuesAsNumpy()
                return arr if arr is not None else np.full(len(times), np.nan)
            except Exception:
                return np.full(len(times), np.nan)

        df_w = pd.DataFrame({
            "time":                   times,
            "temperature_2m":         _safe_var(0),
            "relative_humidity_2m":   _safe_var(1),
            "apparent_temperature":   _safe_var(2),
            "precipitation":          _safe_var(3),
            "wind_speed_10m":         _safe_var(4),
            "shortwave_radiation":    _safe_var(5),
        })

        # Remplacer les sentinelles -9999 du SDK par NaN
        for col in df_w.columns:
            if col == "time":
                continue
            df_w[col] = pd.to_numeric(df_w[col], errors="coerce")
            df_w.loc[df_w[col] < -999, col] = np.nan
        return df_w

    except Exception as exc:
        warnings.warn(f"Open-Meteo SDK error ({date_str}→{end_str}) : {exc}")
        return None


```

L'appel prend moins d'une seconde. La réponse est un fichier JSON contenant 24 valeurs horaires (une par heure de la journée).

> Si la course s'étale sur plusieurs jours, on veillera bien à mettre un jour de début et de fin différent.

Tu retrouveras le code complet et documenté dans le fichier annexe `trail_analysis_pub.py`.

---

## Interpoler au pas d'une seconde

Le DataFrame GPS du notebook contient un point par seconde, c'est ce qu'on a récupéré de notre montre. Les données météo sont horaires (24 points). On doit donc les interpoler linéairement en alignant sur le timestamp :

```python
def enrich_df_with_weather(df, df_weather):
    """Interpolate hourly weather onto the GPS DataFrame."""
    t_w   = df_weather["time"].astype("int64") // 10**9
    t_gps = df["timestamp"].dt.tz_localize(None).astype("int64") // 10**9

    df["temp_api"]         = np.interp(t_gps, t_w, df_weather["temperature_2m"])
    df["humidity_api"]     = np.interp(t_gps, t_w, df_weather["relative_humidity_2m"])
    df["wind_kmh_api"]     = np.interp(t_gps, t_w, df_weather["wind_speed_10m"])
    df["apparent_temp_api"] = np.interp(t_gps, t_w, df_weather["apparent_temperature"])
    return df
```

L'interpolation linéaire, c'est une approximation acceptable pour la température, l'humidité qui évoluent lentement *a priori*. La précipitation en revanche ne s'interpole pas — on la garde à la résolution horaire.


---

## Ce que le graphique révèle

La figure produite par `plot_weather_along_race()` comprend deux panneaux :

**Panel 1 — Températures.** La courbe montre (lissée sur 3 km) et la courbe ERA5-Land interpolée. 

**Panel 2 — Humidité et vent.** L'humidité est la variable clé pour analyser certaines données physio. en conditions nocturnes. Un vent fort peut compenser partiellement une humidité élevée — mais sur un parcours en forêt, le vent mesuré par ERA5 (vent de surface libre) surestime ce qui est ressenti sous couvert.

---

## Petit regard critique..

**Résolution spatiale de 9 km.**: ERA5-Land ne voit pas les effets locaux : fond de vallon 3°C plus froid, vent catabatique en descente, îlot de chaleur en ville... Sur un parcours haut-alpin (TMBD, Samoëns, Ventoux...), l'écart entre ERA5 et la météo réelle au sol peut atteindre 3–5°C sur les crêtes.

**Interpolation linéaire.**: La météo ne varie pas linéairement, surtout pour le vent. L'interpolation horaire est acceptable pour la température et l'humidité, moins pour les rafales ou les phénomènes convectifs locaux (orage).

**ERA5-Land *vs* stations météo locales.**: Pour une analyse scientifique rigoureuse, croiser ERA5-Land avec les données des stations Météo-France proches (via l'[API publique de Météo-France](https://portail-api.meteofrance.fr/web/fr/)) serait plus robuste. Mais pour un usage terrain et coaching, je pense qu'ERA5-Land est suffisant.

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

- Muñoz Sabater, J. (2019). ERA5-Land hourly data from 2001 to present. ECMWF. [DOI: 10.24381/CDS.E2161BAC](https://doi.org/10.24381/CDS.E2161BAC)
- Hersbach, H., et al. (2023). ERA5 hourly data on single levels from 1940 to present. ECMWF. [DOI: 10.24381/cds.adbb2d47](https://doi.org/10.24381/cds.adbb2d47)
- Open-Meteo (2024). Historical Weather API — ERA5-Land. [open-meteo.com](https://open-meteo.com/en/docs/historical-weather-api)
