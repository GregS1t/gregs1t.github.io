---
layout: page
title: CIANNA On-The-Fly
description: A client–server system for asynchronous, batched deep-learning inference on astronomical FITS images, built on the CIANNA framework and IVOA standards.
#img: assets/img/projects/cianna_otf.png
importance: 1
category: work
related_publications: false
---


*CIANNA_OTF is developed at **Observatoire de Paris — LUX Team** in the framework of the **SKA (Square Kilometre Array)** project. The system is designed to serve inference from the YOLO-CIANNA source detection pipeline on radio astronomy data, where processing volumes make manual, sequential inference impractical.*


## Overview

**CIANNA On-The-Fly (CIANNA\_OTF)** is a spin-off of the CIANNA framework, recently developed and still in the testing phase.


It is a client-server system designed to perform asynchronous and batch inferences on astronomical images in FITS format using CIANNA.

The system addresses a concrete operational challenge in scientific machine learning pipelines:
deep learning models used for radio source detection are computationally expensive to load and initialize. Performing inference naively (one image -> one query) can lead to inefficient GPU utilization, long response times, and fragile workflows. 

CIANNA\_OTF attempts to solve this problem by introducing an intermediate execution layer that **groups compatible queries into batches**. This allows the model to be loaded only once and multiple tasks to be processed simultaneously, while exposing a clear, task-oriented API to users.

The architecture is designed for **interoperability with virtual observatories**, following the IVOA Universal Worker Service (UWS) model for asynchronous execution and returning results in IVOA VOTable format. Ongoing development also places a strong emphasis on provenance, another IVOA standard intended to ensure the reproducibility of runs. 


## Key Concepts

**Task** — a single inference request consisting of an XML description (parameters, model selection, region of interest, etc.) and a FITS image file. Each task is assigned a unique identifier and progresses through clearly defined execution phases.

**Batch** — a group of tasks sharing the same model identifier and quantification mode (these criteria may be subject to change), created automatically by the server according to configurable rules (minimum size, maximum size, waiting time).


## Features

- Asynchronous job submission and polling following **IVOA UWS** semantics ;
- Automatic **request batching** for efficient GPU utilisation ;
- Results returned as **IVOA VOTable** files ;
- Server-side **provenance tracking** for scientific traceability ;
- Three client interfaces: CLI (headless), TTY dashboard (interactive), GUI (PyQt) ;
- Conda-based environment, compatible with CPU and GPU servers.

## Technical Details

- **Language:** Python / Shell
- **License:** Apache 2.0
- **Status:** Active development — functional job lifecycle, stable API
- **IVOA compliance:** UWS (partial), VOTable (full), PROV (initial)
- **Dependency:** [CIANNA framework](https://github.com/Deyht/CIANNA)

## Status

CIANNA OTF is currently undergoing intensive testing. Following a series of successful local tests, the server stack has been successfully deployed on a dedicated machine. After a development phase using a Flask server, an NGINX server has been set up to handle HTTPS connections and forward requests to Flask. 


## Related Publication

- Cornu, D., Salomé, P., Semelin, B., ..., **Sainton, G.**, et al. (2024). **YOLO-CIANNA: Galaxy detection with deep learning in radio data.** *Astronomy & Astrophysics*, 690, A211. [https://doi.org/10.1051/0004-6361/202449548](https://doi.org/10.1051/0004-6361/202449548)

## Links

- [CIANNA OTF repository](https://github.com/GregS1t/CIANNA_OTF)
- [CIANNA framework](https://github.com/Deyht/CIANNA)
- [CIANNA Website](https://cianna.obspm.fr/index.html) 
- [SKA Observatory](https://www.skao.int/en)