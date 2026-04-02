---
layout: page
title: CIANNA On-The-Fly
description: A client–server system for asynchronous, batched deep-learning inference on astronomical FITS images, built on the CIANNA framework and IVOA standards.
#img: assets/img/projects/cianna_otf.png
importance: 1
category: work
related_publications: true
---

## Overview

**CIANNA On-The-Fly (CIANNA\_OTF)** is a client–server system designed to run
asynchronous, batched inference on astronomical FITS images using the
[CIANNA deep-learning framework](https://github.com/Deyht/CIANNA).

The system addresses a concrete operational challenge in scientific ML pipelines:
deep-learning models used for radio source detection are expensive to load and
initialise. Running inference naïvely — one request triggering one model load —
leads to poor GPU utilisation, long response times, and fragile workflows.

CIANNA\_OTF solves this by introducing an intermediate execution layer that
**groups compatible requests into batches**, loads the model once, and processes
multiple jobs together, while exposing a clean, job-oriented API to users.

The architecture is designed for **Virtual Observatory interoperability**, following
the IVOA Universal Worker Service (UWS) pattern for asynchronous execution and
returning results in the IVOA VOTable format.

## Key Concepts

**Job** — a single inference request consisting of an XML description
(parameters, model choice, region of interest) and a FITS image file.
Each job is assigned a unique identifier and progresses through well-defined
execution phases.

**Batch** — a group of jobs sharing the same model identifier and quantization
mode, created automatically by the server according to configurable rules
(minimum size, maximum size, waiting time).

## Features

- Asynchronous job submission and polling following **IVOA UWS** semantics
- Automatic **request batching** for efficient GPU utilisation
- Results returned as **IVOA VOTable** files
- Server-side **provenance tracking** for scientific traceability
- Three client interfaces: CLI (headless), TTY dashboard (interactive), GUI (PyQt)
- Conda-based environment, compatible with CPU and GPU servers

## Technical Details

- **Language:** Python / Shell
- **License:** Apache 2.0
- **Status:** Active development — functional job lifecycle, stable API
- **IVOA compliance:** UWS (partial), VOTable (full), PROV (initial)
- **Dependency:** [CIANNA framework](https://github.com/Deyht/CIANNA)

## Context

Developed at **Observatoire de Paris — LUX Team** in the framework of the
**SKA (Square Kilometre Array)** project. The system is designed to serve
inference from the YOLO-CIANNA source detection pipeline on radio astronomy
data, where processing volumes make manual, sequential inference impractical.

## Related Publication

- Cornu, D., Salomé, P., Semelin, B., ..., **Sainton, G.**, et al. (2024).
  **YOLO-CIANNA: Galaxy detection with deep learning in radio data.**
  *Astronomy & Astrophysics*, 690, A211.
  [https://doi.org/10.1051/0004-6361/202449548](https://doi.org/10.1051/0004-6361/202449548)

## Links

- [GitHub repository](https://github.com/GregS1t/CIANNA_OTF)
- [CIANNA framework](https://github.com/Deyht/CIANNA)
- [SKA Observatory](https://www.skao.int/en)