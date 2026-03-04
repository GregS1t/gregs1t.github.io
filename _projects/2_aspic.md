---
layout: page
title: ASPIC
description: A Python/Jupyter tool for visualising and processing seismic MiniSEED data from the NASA InSight mission on Mars.
img: assets/img/projects/aspic.png
importance: 3
category: work
related_publications: false
---

## Overview

**ASPIC** (*Analyzing SEIS Products for Investigations and Commissioning*) is a Python
and Jupyter Notebook-based tool developed for the **NASA InSight mission** to quickly
visualise and process seismic data recorded by the SEIS instrument on Mars — the first
seismometer ever deployed on another planet.

The tool was built to meet the operational needs of the InSight science team during both
the commissioning phase and routine scientific analysis, providing a fast and flexible
interface for exploring MiniSEED data directly from the SEIS Data Portal.

## Features

- **MiniSEED visualisation** — rapid display of raw seismic waveforms from the SEIS instrument
- **Signal processing** — filtering (FIR), detrending, and component rotation
- **Martian time support** — integrated with [Mars Time Converter](https://github.com/GregS1t/marstimeconverter) for LMST timestamps
- **Configurable** — mission parameters set via `.ini` configuration files
- **Cross-platform** — runs on Linux/macOS (shell launcher) and Windows (`.bat` launcher)
- **Interactive** — available both as a CLI tool and as a Jupyter Notebook (`aspic.ipynb`)

## Technical Details

- **Language:** Python / Jupyter Notebook
- **License:** MIT
- **Dependencies:** MarsConverter (Mars Time Converter), SEIS Data Portal credentials
- **Key modules:** FIR filters, signal processing utilities, configuration management
- **Documentation:** [PSS GitLab wiki](https://pss-gitlab.math.univ-paris-diderot.fr/sainton/aspic/wikis/home)

## Context

ASPIC was developed at **IPGP** (Institut de Physique du Globe de Paris) in the framework
of the NASA InSight mission. The SEIS instrument recorded seismic activity on Mars from
2019 to 2022, detecting hundreds of marsquakes including
[S1222a](https://doi.org/10.1029/2022GL101543), the largest event observed during the mission.

A better graphical version is still on going using PyQT and PyQTGraph hence an 
innovative adaptative zooming tool for high frequency files.

## Links

- [GitHub repository](https://github.com/GregS1t/aspic)
- [Documentation](https://pss-gitlab.math.univ-paris-diderot.fr/sainton/aspic/wikis/home)
- [NASA InSight mission](https://www.seis-insight.eu/fr/)