---
layout: page
title: Rattlesnake
description: A PyQt-based GUI for controlling an optical seismometer test bench — integrating an ATTOCUBE interferometer, a Newport PicoMotor, and an Agilent power supply.
importance: 4
category: work
related_publications: false
---

## Overview

**Rattlesnake** is a modular graphical application built with PyQt, developed at
**IPGP** in the context of the **PIONEERS** project. It provides a unified control
interface for the instruments of an optical seismometer test bench, combining
hardware control, real-time signal acquisition, and data visualisation in a
single desktop application.

The name reflects both the hardware it controls — precision instruments that
are sensitive to the slightest vibration — and the modular philosophy of the
codebase, designed to easily accommodate new instrument modules.

## Controlled Instruments

- **ATTOCUBE IDS3010** — high-precision optical interferometer for sub-nanometre
  displacement measurements (proprietary drivers required)
- **Newport PicoMotor 8742** — piezoelectric motor controller for fine positioning
- **Agilent 3631A** — programmable triple-output DC power supply

## Features

- Unified PyQt desktop interface with dynamic, plugin-style module loading
- Real-time signal acquisition and display via PyQtGraph
- Hardware communication via PyVISA (GPIB/USB) and PyUSB
- Session management and data export
- Cross-platform: Linux/macOS shell launcher, Windows `.bat` launcher
- Modular architecture — new instrument modules can be added without modifying
  the core application

## Technical Details

- **Language:** Python (45%) / C (42%) / Jupyter Notebook (13%)
- **Interface:** PyQt
- **License:** MIT
- **Key dependencies:** PyQt, PyQtGraph, PyVISA, PyUSB, Matplotlib, Pandas, NumPy
- **Environment:** Conda (`environment.yml`)

## Context

Developed at **IPGP** (Institut de Physique du Globe de Paris) on behalf of the
**PIONEERS** project, which aims at developing next-generation seismometers for
planetary science. The test bench controlled by Rattlesnake is used to characterise
the optical readout of prototype seismometer components under controlled conditions.

## Usage

```bash
git clone https://github.com/GregS1t/rattlesnake.git
conda env create -f environment.yml
conda activate rs_env
python rattlesnake.py
```

## Note on ATTOCUBE drivers

The ATTOCUBE interferometer drivers are proprietary — controlling the IDS3010
through Rattlesnake requires a valid licence from ATTOCUBE. The rest of the
application runs without this dependency.

## Links

- [GitHub repository](https://github.com/GregS1t/rattlesnake)