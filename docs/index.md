---
icon:
    material/home
hide: 
    - navigation
    - toc 
---

<!-- !!! warning "Website under active construction"
    Please take a look around and explore the OpenMRF documentation. Just be aware that this is work in progress and more detailed information on certain aspects of the framework will be added soon.  -->

<br>

<div style="text-align:center">
  <img src="assets/OpenMRF.png" alt="OpenMRF logo" width="700">
</div>

<br>

<div class="center-buttons">
  <a href="quickstart" class="md-button"> 🚀 Quickstart Guide</a>
  <a href="wiki" class="md-button"> 📚 Wiki</a>
  <!-- <a href="#citation" class="md-button"> 📝 Citation </a> -->
</div>

<br>

# Welcome to OpenMRF! 

OpenMRF is currently under active development. A corresponding publication describing the framework in detail will be released soon. Please watch out for this publication and use it for citation once available.

## New to OpenMRF? 
We strongly recommend getting started by reading the [Quickstart guide](quickstart.md) carefully. For more detailed information on specific topics, refer to the [wiki](wiki/index.md).

## Introduction
OpenMRF is a modular and vendor-neutral framework for Magnetic Resonance Fingerprinting (MRF) built on the open-source Pulseq standard. It is built upon the MATLAB version of Pulseq by Kelvin J. Layton and Maxim Zaitsev (<https://doi.org/10.1002/mrm.26235>). OpenMRF unifies all core components of the MRF workflow within a single MATLAB-based toolbox: flexible sequence generation, automated Bloch-based dictionary simulation, and low-rank image reconstruction. The provided tools support a wide range of contrast preparations and readouts and include integrated solutions for trajectory calibration, spin-lock modeling, slice profile simulation, and metadata storage. Designed for reproducibility and portability, OpenMRF enables easy deployment of MRF protocols across multiple scanner platforms, including Siemens and GE systems.

## Codebase overview
- `include_miitt/`: Contains the low-rank reconstruction code provided by the [MIITT group](https://miitt.med.umich.edu/) and Jeffrey Fessler's [MIRT toolbox](https://web.eecs.umich.edu/~fessler/code/). Includes an installation script. 
- `include_misc/`: Miscellaneous utilities and helper functions.
- `include_pre_sim_library/`: Library of SLR optimized RF pulse waveforms (including `sigpy` generated pulses) and flip angle patterns for MRF. Also used to store pre-simulated slice profiles, adiabatic efficiencies and compressed dictionaries.
- `include_pulseq_original/`: Copy of the [official Pulseq repo](https://github.com/pulseq/pulseq) (v1.5, April 01, 2025). Includes minor modifications to the plotting functions for visualizing trigger inputs/outputs.
- `include_pulseq_toolbox/`: Contains standard imaging readouts (cartesian, radial, spiral, rosette) combined with various preparation modules (inversion recovery, saturation, MLEV-T2, spin-lock, adiabatic spin-lock, CEST). Also includes simulation tools for MRF dictionary generation.
- `main_sequences/`: Example Pulseq sequences and reconstruction scripts.
- `projects/`: Collection of projects, which were published or presented on conferences or which are currently work in progress. For your own project, we recommend creating a subfolder and saving all your scripts here.
- `user_specifications/`: User specific definitions (automatically generated via `install_OpenMRF.m`), MRI system specifications (create a `.csv` file for your system's gradient limits and timings) and optionally your python cmd specifications for `sigpy` pulses.

## System Requirements
- **MATLAB** tested with R2024b and R2025a on Win11 and Ubuntu 22.04.
<!-- - **Python** with the `sigpy` package — required for designing SLR and adiabatic RF pulses (e.g., BIR-4). More information on the Python setup can be found [here](wiki/python.md).  -->

## Citation
Coming soon! 

<br><br>
<div style="text-align:center">
  <img src="assets/OpenMRF_banner.png" alt="OpenMRF banner">
</div>