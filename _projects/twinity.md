---
layout: page
title: Twinity
description: A digital twin for trail running — fitness, fatigue, and performance readiness modelling.
img: assets/img/twinity_preview.png
importance: 1
category: work
---

# Twinity

A digital twin for trail running — ingesting FIT files from GPS watches to model fitness, fatigue, and performance readiness using physiological models validated in the scientific literature.

## Overview

**Twinity** is a personal project aimed at building a **digital twin of a trail runner**, combining physiological data, training load models, and daily health metrics to simulate and optimise endurance training.

The system ingests `.FIT` files from GPS sports watches (Suunto, Garmin, etc.) to reconstruct training sessions and provide insights into **fitness**, **fatigue**, and **performance readiness** — with the long-term goal of personalising the model to each athlete using machine learning.

Because why apply data science only to Mars when you can also apply it to mountains? 🏔️

## Features

### 👤 Athlete management
Create and edit athlete profiles (HR max, resting HR, sports). Import `.FIT` sessions directly from the Athlete page — sport type, session type (training / competition / race…), and RPE are recorded at import time.

### 📈 Training load dashboard (PMC)
- **Performance Management Chart** — CTL (fitness), ATL (fatigue), TSB (form) computed from TRIMP (Banister 1991) using exponentially weighted moving averages (Coggan 2003).
- Competition markers ⭐ overlaid on the TSB curve.
- **Automatic calibration of τ_CTL and τ_ATL** — grid search over (14–60 j) × (3–14 j), maximising the Pearson correlation between TSB on the eve of competitions and normalised race performance. Requires ≥ 5 competition results. A heatmap of the full correlation grid is displayed.
- Monthly distance chart by sport (stacked bars), filterable by date range and sport type.
- Session list with direct access to individual analytics.

### 📊 Session analytics
Per-session dashboard with:
- Interactive time-series (HR, pace, altitude, cadence, power, vertical speed).
- Heart rate zone distribution.
- Elevation profile with terrain classification (steep climb / climb / flat / descent / steep descent).
- Splits and pause analysis with aid-station attribution.
- Advanced analytics (3D GPS trajectory, correlation matrix, moving averages).
- Full session editing (title, sport, **session type**, RPE, notes, aid stations).

### 🏔️ Race preparation
- Upload a GPX trace of the target race.
- Automatic segmentation using the Ramer–Douglas–Peucker algorithm.
- Per-segment speed estimation based on the Minetti (2002) energetic model, calibrated on the athlete's training history.
- Aid station entry with estimated split times and elevation profile.
- Historical comparison with past races of similar distance and D+.
- Save / reload race preparations from the database.

### 📉 Body metrics tracking
- Daily logging of weight, HRV (RMSSD), sleep duration, resting HR, and any custom metric.
- **GitHub-style heatmap calendar** — one year at a glance, colour-coded per metric.
- Multi-metric time-series with optional PMC overlay (CTL / ATL / TSB on a shared x-axis).
- CSV export.

## Scientific models

| Model | Reference | Usage |
|-------|-----------|-------|
| TRIMP | Banister EW (1991) | Session internal load |
| PMC / CTL–ATL–TSB | Coggan AR (2003) | Fitness–fatigue–form dynamics |
| Banister impulse–response | Morton RH et al. (1990) | Predictive performance modelling |
| PMC calibration | Hellard P et al. (2006) | Individual τ_CTL / τ_ATL estimation |
| Energetic cost vs slope | Minetti AE et al. (2002) | Race segment speed prediction |
| Ramer–Douglas–Peucker | Ramer (1972) | GPX trace segmentation |

Full equations, derivations, and references are documented in `docs/SCIENCE.md`.

## Technical details

- **Language:** Python 3.13
- **Interface:** [Streamlit](https://streamlit.io) ≥ 1.36
- **Key dependencies:** `fitparse`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `plotly`, `sqlalchemy`
- **Storage:** SQLite (local) or PostgreSQL / Supabase (cloud via `st.secrets`)
- **Status:** active development

## Links

- [GitHub repository](https://github.com/GregS1t/Twinity)
