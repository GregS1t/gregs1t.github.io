---
layout: page
title: SeisDeglitch
description: MATLAB tools for automated detection and removal of glitches in seismic data from the SEIS instrument onboard NASA InSight.
#img: assets/img/projects/seisdeglitch.png
importance: 4
category: work
related_publications: true
---

## Overview

**SeisDeglitch** is a set of MATLAB tools developed for the **NASA InSight mission** to
identify and remove *glitches* from the seismic data recorded by the SEIS instrument on Mars.

Glitches are transient non-seismic artefacts that contaminate the signal — they arise from
mechanical and thermal perturbations of the instrument itself, and their presence can mask
genuine seismic events or distort waveform analysis. Detecting and removing them reliably
is a critical step in the scientific processing pipeline of SEIS data.

This work contributed directly to the publication
[Scholz et al. (2020)](https://doi.org/10.1029/2020EA001317), which established the
methodology for glitch detection, analysis, and removal for the InSight seismic dataset.

Primary algorithms were developped by Philippe Lognonné. Here, we derived the code to fit and remove "multi-glitches" (entangled glitches). 

## Features

- **Glitch detection** — automated identification of transient artefacts in SEIS time series
- **Glitch fitting** — parametric fitting of glitch waveforms (`fit_glitch.m`, `fit_glitch_v2.m`)
- **FIR filtering** — dedicated filter bank for pre-processing seismic channels
- **Event catalogue** — curated set of reference glitch events for validation
- **Modular architecture** — separate subroutine sets (`subroutines_GS`, `subroutines_PL`)
  allowing independent development by multiple contributors
- **Documentation** — bibliographic references and presentation materials included

## Technical Details

- **Language:** MATLAB (95%) / Jupyter Notebook (5%)
- **License:** not specified
- **Key files:** `fit_glitch.m`, `fit_glitch_v2.m`
- **Structure:** `FIR/`, `Events/`, `metadata/`, `Biblio/`, `Presentation/`

## Context

Developed at **IPGP** (Institut de Physique du Globe de Paris) in the framework of
the NASA InSight mission. The SEIS instrument was the first seismometer successfully
deployed on Mars, operating from 2019 to 2022. Glitch removal was a prerequisite for
the scientific exploitation of the dataset, enabling the detection and characterisation
of marsquakes including
[S1222a](https://doi.org/10.1029/2022GL101543), the largest event of the mission.

## Related Publication

- Scholz, J.-R., Widmer-Schnidrig, R., Davis, P., Lognonné, P., Pinot, B., Garcia, R. F.,
  **Sainton, G.**, et al. (2020). **Detection, analysis, and removal of glitches from
  InSight's seismic data from Mars.** *Earth and Space Science*, 7, e2020EA001317.
  [https://doi.org/10.1029/2020EA001317](https://doi.org/10.1029/2020EA001317)

## Links

- [GitHub repository](https://github.com/GregS1t/seisdeglitch)
- [NASA InSight mission](https://www.seis-insight.eu/fr/)