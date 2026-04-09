---
layout: page
title: Mars Time Converter
description: A Python library and CLI tool for converting Earth UTC time to Martian time systems, supporting multiple Mars missions.
#img: assets/img/projects/marstimeconverter.png
importance: 2
category: work
related_publications: false
---

## Overview

**Mars Time Converter** is an open-source Python library and command-line tool designed
to convert Earth UTC time into Martian time references, including Local Mean Solar Time
(LMST), Mars Sol Date (MSD), Coordinated Mars Time (MTC), and Equation of Time (EOT).

Developed in the context of the NASA InSight mission, the tool addresses a concrete
operational need: synchronising scientific data timestamps with the Martian day cycle,
which differs fundamentally from Earth's 24-hour system. A Martian sol lasts approximately
24 hours and 39 minutes, making direct time comparison non-trivial.

The library's flexible architecture supports multiple Mars missions through a mission-specific
XML configuration file, making it straightforward to extend to future missions.

## Features

- **UTC → LMST** conversion (Local Mean Solar Time)
- **LMST → UTC** reverse conversion
- Computation of Mars Sol Date (MSD), Coordinated Mars Time (MTC), and Equation of Time (EOT)
- Mission-specific configuration via XML files
- Command-line interface (`marsnow`) for quick terminal use
- Full documentation hosted on the [OBSPM GitLab wiki](https://gitlab.obspm.fr/gsainton/marstimeconverter)

## Technical Details

- **Language:** Python (90%) / Shell (10%)
- **License:** MIT
- **Package:** installable via `setup.py` / `pyproject.toml`
- **Tests:** dedicated test suite in `tests/`
- **Documentation:** ReadTheDocs-compatible (`.readthedocs.yml`)

## Links

- [GitHub repository](https://github.com/GregS1t/marstimeconverter)
- [Full documentation](https://gitlab.obspm.fr/gsainton/marstimeconverter/)