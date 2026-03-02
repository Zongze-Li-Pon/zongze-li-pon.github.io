---
title: "Fast Windkessel Model Parameter Estimation for Multi-Scale CFD of Aortic Flow"
date: 2023-06-15
summary: "Developed a fast and scalable multi-scale CFD framework for patient-specific aortic flow by coupling LBM simulations with Windkessel models, enabling efficient parameter estimation and accurate hemodynamic prediction."
tags: [CNC, Drilling, Milling, Manufacturing]
---
<a href="/completed-projects.html">← Back to Completed Projects</a>
## Fast Parameter Estimation for Multi-Scale CFD of Aortic Flow
## Project Overview
Developed a fast and scalable framework to enable patient-specific multi-scale CFD simulations of aortic hemodynamics by automatically estimating Windkessel model parameters.

This work bridges 3D high-fidelity CFD (LBM) and 0D reduced-order cardiovascular models, enabling efficient and realistic simulation of blood flow in complex vascular systems.

## Problem & Motivation

Accurate cardiovascular CFD simulations require physiological outlet boundary conditions, typically modeled using Windkessel (WK3) models.

However:

- WK3 parameters (Rc, Rp, C) are case-specific and unknown
- Existing methods:
  - Require iterative CFD simulations (very expensive)
  - Ignore geometry-induced resistance
  - Or rely on over-simplified assumptions (e.g., Poiseuille flow)

Result: High computational cost or reduced accuracy in patient-specific simulations.

## What I Built (Core Contribution)

Designed a fast parameter estimation pipeline with three key components:

1. Geometry-aware resistance extraction
   - Performed steady CFD simulation
   - Extracted branch-wise geometric resistance (Rg)
   - Captures real effects of:
     - complex geometry
     - bifurcations
     - stenosis

2. Reduced-order circuit modeling
   - Built a 0D circuit analog coupling:
     - geometric resistance (Rg)
     - Windkessel model (Rc, Rp, C)

3. Optimization-based parameter estimation
   - Implemented MATLAB global optimization (pattern search)
   - Solved WK3 ODE system to match:
     - systolic / diastolic pressure
     - flow distribution at outlets
   - Optimization runtime: < 1 minute

## Technical Stack

- CFD Solver: Lattice Boltzmann Method (LBM)
- Framework: C++ + MPI (Palabos-based)
- Optimization: MATLAB Global Optimization Toolbox
- Scale: ~21 million grid points (HPC cluster)
- Physics: Multi-scale coupling (3D CFD + 0D lumped model)

## Key Results

- Achieved accurate control of flow distribution and pressure waveform
- Pressure prediction error: as low as ~0.4 mmHg 
- Successfully handled: normal geometry, stenosed aorta, non-physiological flow distributions
- Reduced computational cost by:
  - eliminating iterative CFD loops
  - using only one steady CFD simulation

## Engineering Impact

- Enabled fast and reliable patient-specific simulations
- Improved robustness for:
  - complex geometries
  - pathological conditions (e.g., stenosis)
- Provides a scalable workflow for multi-scale modeling
- Applicable to:
  - biomedical simulation platforms
  - digital twins in healthcare
  - simulation-driven diagnosis tools

## Result Gifs
<div style="text-align: center;">
  <img src="/completed-projects/2023-06-15_Fast-Windkessel-Model-Parameter-Estimation-for-Multi-Scale-CFD-of-Aortic-Flow/velocity-field.gif" width="70%">
  <p>Velocity Field</p>
</div>
<div style="text-align: center;">
  <img src="/completed-projects/2023-06-15_Fast-Windkessel-Model-Parameter-Estimation-for-Multi-Scale-CFD-of-Aortic-Flow/vorticity-field.gif" width="70%">
  <p>Vorticity Field</p>
</div>
<div style="text-align: center;">
  <img src="/completed-projects/2023-06-15_Fast-Windkessel-Model-Parameter-Estimation-for-Multi-Scale-CFD-of-Aortic-Flow/surface-force.gif" width="70%">
  <p>Surface Force</p>
</div>

## Paper Link
The whole paper can be found [here](https://www.sciencedirect.com/science/article/abs/pii/S0045793023001196)

<a href="/completed-projects.html">← Back to Completed Projects</a>