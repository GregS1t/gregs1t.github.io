---
layout: post
title: "[data] weather as training data: Retrieving real conditions with ERA5-Land"
date: 2026-04-23
description: >
  Your watch temperature is biased by body heat. How to retrieve actual weather
  conditions during your race using the ERA5-Land reanalysis via Open-Meteo.
tags: [trail, data, python, weather, open-meteo, ERA5, temperature, humidity, running, explained]
categories: [explained, trail, data, running]
related_posts: false
toc:
  sidebar: left
math: false
---

*"I was too hot on the second half. But was it really the temperature, or was it fatigue doing all the talking?"*

On any race, thermal conditions directly affect physiology. Cardiac drift can result from heat just as much as from muscular fatigue. But to tell which one, you need the actual weather, not your watch's estimate.

> On the topic of heat acclimatization, I recommend the article and podcast on Courir-Mieux: [Heat and trail: how to acclimatize?](https://courir-mieux.fr/acclimatation-chaleur-trail/)

---

## The problem with wrist temperature

The `temperature` column in a FIT file comes from a sensor embedded in the watch case. On the wrist, that sensor picks up both ambient heat and body heat conducted through the skin. The data is systematically biased.

At rest, the watch reads close to ambient temperature. During effort, the bias climbs to **+2 to +5 degrees C** depending on the model, bracelet position, and exercise intensity. The bias is higher during climbs (cutaneous vasodilation) than during descents or at aid stations.

The watch is therefore useful for observing **variations** in temperature along the course, but its absolute values are not reliable for comparison with physiological thresholds.

---

## The solution: ERA5-Land via Open-Meteo

[Open-Meteo](https://open-meteo.com) is a **free** weather API for research and non-commercial use.

We use the **ERA5-Land** model, a climate reanalysis that assimilates millions of in-situ measurements (weather stations, radiosondes, satellites, buoys) into a physical model. It is recomputed a posteriori, which improves data homogeneity.

Its characteristics for our purpose:

| Parameter | Value |
|---|---|
| Spatial resolution | 0.1 x 0.1 degrees, roughly **9 km** |
| Temporal resolution | **1 hour** |
| Availability | **2001 to present** |
| Coverage | Global |
| Access | Free, no registration |

At 9 km resolution, ERA5-Land does not capture micro-local effects (cold valley bottoms, wind on ridgelines), but it provides a reference far superior to the watch sensor.

---

## Fetching the data

The API endpoint for historical data is `https://archive-api.open-meteo.com/v1/archive`. Here is the retrieval function:

```python
import openmeteo_requests

_ARCHIVE_URL = "https://archive-api.open-meteo.com/v1/archive"
_HOURLY_VARS = [
    "temperature_2m", "relative_humidity_2m",
    "apparent_temperature", "precipitation",
    "wind_speed_10m", "shortwave_radiation",
]


def fetch_weather_hourly(lat, lon, date_str, timezone="Europe/Paris",
                         client=None, model="era5_land", date_end=None):
    """Fetch hourly weather data from Open-Meteo ERA5-Land."""
    if client is None:
        client = openmeteo_requests.Client()
    end_str = date_end if date_end is not None else date_str

    params = {
        "latitude": lat, "longitude": lon,
        "start_date": date_str, "end_date": end_str,
        "hourly": _HOURLY_VARS,
        "wind_speed_unit": "kmh",
        "timezone": timezone,
        "models": model,
    }

    responses = client.weather_api(_ARCHIVE_URL, params=params)
    r = responses[0]
    hourly = r.Hourly()

    times = pd.date_range(
        start=pd.to_datetime(hourly.Time(), unit="s", utc=True),
        end=pd.to_datetime(hourly.TimeEnd(), unit="s", utc=True),
        freq=pd.Timedelta(seconds=hourly.Interval()),
        inclusive="left",
    )

    def safe_var(i):
        """Extract variable i; return NaN array if unavailable."""
        try:
            arr = hourly.Variables(i).ValuesAsNumpy()
            return arr if arr is not None else np.full(len(times), np.nan)
        except Exception:
            return np.full(len(times), np.nan)

    df_w = pd.DataFrame({
        "time": times,
        "temperature_2m": safe_var(0),
        "relative_humidity_2m": safe_var(1),
        "apparent_temperature": safe_var(2),
        "precipitation": safe_var(3),
        "wind_speed_10m": safe_var(4),
        "shortwave_radiation": safe_var(5),
    })

    for col in df_w.columns:
        if col == "time":
            continue
        df_w[col] = pd.to_numeric(df_w[col], errors="coerce")
        df_w.loc[df_w[col] < -999, col] = np.nan

    return df_w
```

The call takes less than a second. The response contains 24 hourly values (one per hour of the day). If the race spans multiple days, set `date_end` accordingly.

---

## Interpolating to 1-second resolution

The GPS DataFrame has one point per second. The weather data has 24 points per day. We interpolate linearly by aligning on timestamps:

```python
def make_utc_naive(ts):
    """Force any timestamp Series to UTC-naive for safe comparison."""
    if ts.dt.tz is not None:
        return ts.dt.tz_convert("UTC").dt.tz_localize(None)
    return ts


def enrich_df_with_weather(df, df_weather):
    """Interpolate hourly weather onto the GPS DataFrame."""
    # Normalize both timestamp columns to UTC-naive
    t_gps_naive = make_utc_naive(df["timestamp"])
    t_w_naive = make_utc_naive(df_weather["time"])

    # Debug
    print(f"  GPS range    : {t_gps_naive.iloc[0]} -> {t_gps_naive.iloc[-1]}")
    print(f"  Weather range: {t_w_naive.iloc[0]} -> {t_w_naive.iloc[-1]}")

    # Convert to float seconds for np.interp
    ref = pd.Timestamp("2000-01-01")
    t_gps_s = (t_gps_naive - ref).dt.total_seconds().to_numpy()
    t_w_s = (t_w_naive - ref).dt.total_seconds().to_numpy()

    print(f"  Epoch GPS    : {t_gps_s[0]:.0f} -> {t_gps_s[-1]:.0f}")
    print(f"  Epoch weather: {t_w_s[0]:.0f} -> {t_w_s[-1]:.0f}")

    if t_gps_s[0] > t_w_s[-1] or t_gps_s[-1] < t_w_s[0]:
        print("  ERROR: no overlap between GPS and weather timestamps!")
        return df

    for src_col, dst_col in [
        ("temperature_2m", "temp_api"),
        ("relative_humidity_2m", "humidity_api"),
        ("wind_speed_10m", "wind_kmh_api"),
        ("apparent_temperature", "apparent_temp_api"),
    ]:
        if src_col in df_weather.columns:
            vals = df_weather[src_col].to_numpy(dtype=float)
            if np.all(np.isnan(vals)):
                df[dst_col] = np.nan
                print(f"  {dst_col}: all NaN, skipped")
            else:
                df[dst_col] = np.interp(t_gps_s, t_w_s, vals)
                print(f"  {dst_col}: {df[dst_col].min():.2f} -> {df[dst_col].max():.2f}")
        else:
            df[dst_col] = np.nan

    return df

df = enrich_df_with_weather(df, df_weather)
```

Linear interpolation is an acceptable approximation for temperature and humidity, which evolve slowly. Precipitation, on the other hand, does not interpolate well and should be kept at hourly resolution.

---

## What the figure reveals

The plot produced by the notebook has two panels:

**Panel 1: Temperatures.** The watch curve (smoothed over 3 km) and the ERA5-Land interpolated curve side by side. Expect the watch to read 2-5 degrees higher than ERA5 during effort, with the gap narrowing at aid stations and during descents.

<div class="plotly-container">
  <iframe src="{{ site.baseurl }}/assets/2026-04_weather/weather_along_race.html"
          width="100%" height="450" frameborder="0" scrolling="no">
  </iframe>
</div>

**Panel 2: Humidity and wind.** Humidity is the key variable for analyzing physiology in nocturnal conditions. Strong wind can partially compensate for high humidity. But on a forested course, ERA5's surface wind (measured in open terrain) overestimates what is actually felt under canopy.

---

## A critical look

**9 km spatial resolution.** ERA5-Land does not see local effects: a valley bottom 3 degrees colder, katabatic wind on a descent, urban heat islands. On a high-alpine course (UTMB, Ventoux, TGVF...), the gap between ERA5 and actual ground-level weather can reach 3-5 degrees on ridgelines.

**Linear interpolation.** Weather does not vary linearly, especially wind. Hourly interpolation is acceptable for temperature and humidity, less so for gusts or convective events (thunderstorms).

**ERA5-Land vs. local weather stations.** For a scientifically rigorous analysis, cross-referencing ERA5-Land with nearby weather station data (via the [French national weather API](https://portail-api.meteofrance.fr/web/fr/) or equivalent) would be more robust. But for field use and coaching, ERA5-Land is sufficient.

---

The notebook for this article is available on [GitHub](https://github.com/GregS1t/trail-lab/). The only cell you need to change is the one at the top.

## References

- Munoz Sabater, J. (2019). ERA5-Land hourly data from 2001 to present. ECMWF. [DOI: 10.24381/CDS.E2161BAC](https://doi.org/10.24381/CDS.E2161BAC)
- Hersbach, H., et al. (2023). ERA5 hourly data on single levels from 1940 to present. ECMWF. [DOI: 10.24381/cds.adbb2d47](https://doi.org/10.24381/cds.adbb2d47)
- Open-Meteo (2024). Historical Weather API. [open-meteo.com](https://open-meteo.com/en/docs/historical-weather-api)

---

> **Disclaimer:** I am a research engineer, not a sports physiologist. What you read here is the notebook of a curious trail runner who likes understanding his data. Sources are provided so you can verify for yourself.
