---
title: "Windkessel Parameter Estimation MATLAB Code"
date: 2023-06-15
summary: "MATLAB tool for estimating Windkessel (WK3) model parameters by solving the governing ODE and using optimization to match target pressure waveforms, supporting multi-scale CFD simulations of cardiovascular flow."
tags: [CFD, Biomedical Simulation, Windkessel Model, MATLAB, Numerical Methods]
---

<script>
window.MathJax = {
  tex: {
    inlineMath: [['$', '$'], ['\\(', '\\)']],
    displayMath: [['$$','$$'], ['\\[','\\]']]
  }
};
</script>

<script defer src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

<a href="/completed-projects.html">← Back to Completed Projects</a>

<div style="text-align: center;">
  <p style="font-size: 32px; font-weight: bold;">
    Windkessel Parameter Estimation MATLAB Code
  </p>
</div>

## Acknowledgment

Many thanks to my Ph.D. advisor **Dr. Wenbin Mao**  
(https://scholar.google.com/citations?user=FDHGidUAAAAJ&hl), who mentored me during the development of this MATLAB code. This was one of the first research codes I worked with during my Ph.D. Dr. Mao implemented most of the original framework, and I later learned from it and revised and extended parts of the code.

## Background

This MATLAB code was originally developed to estimate **Windkessel model parameters**, which serves as one step in our published paper:

**A fast approach to estimating Windkessel model parameters for patient-specific multi-scale CFD simulations of aortic flow**  
https://www.sciencedirect.com/science/article/abs/pii/S0045793023001196

Specifically, this code corresponds to the **second step described in Section 2.4.2** of that paper.

Beyond the specific application in the paper, the idea behind this code can be generalized to a broader class of problems. In general, the code can be used for situations where:

- a governing **ODE contains unknown coefficients**
- a **time-varying input function** is known
- the goal is to determine the unknown coefficients so that the **resulting waveform satisfies certain desired properties**

To illustrate the idea, we briefly revisit the example used in our paper.

## Governing Equation

The governing equation of the three-element Windkessel model can be written as

$$
\frac{dP(t)}{dt} =
\frac{R_c + R_p}{R_p C} Q(t)
+ R_c \frac{dQ(t)}{dt}
- \frac{P(t) - P_{ref}}{R_p C}
$$

where

- (\R_c\) denotes the **characteristic impedance**
- (\R_p\) represents the **downstream vascular resistance**
- (\C\) is the **total compliance of the downstream vascular system**
- (\P_{ref}\) is a **reference pressure**

In this problem, the inflow waveform \(Q(t)\) is known, and the reference pressure \(P_{ref}\) is prescribed. The goal is to determine the parameters \(R_c\), \(R_p\), and \(C\) such that the resulting pressure waveform \(P(t)\) has the desired maximum and minimum values.

In our study, the target values correspond to the **systolic and diastolic pressures**, which are set to **50 mmHg and 10 mmHg**, respectively (with \(P_{ref} = 70\) mmHg).

## Objective Function

To achieve this goal, we define an objective function that measures the difference between the desired pressures and those obtained from the ODE solution:

$$
obj =
\sqrt{
\frac{1}{2}
\left[
(P_{sys} - P_{sys}^{ODE})^2 +
(P_{dia} - P_{dia}^{ODE})^2
\right]
}
$$

Once this objective function is defined, the problem becomes a **parameter optimization problem**.

## Code Logic

The overall logic of the code is straightforward.

**Given**

- an input waveform \(Q(t)\)
- a governing ODE

**Goal**

Determine the unknown coefficients so that the output waveform reaches the desired maximum and minimum values.

The optimization algorithm automatically tests different sets of parameters and repeatedly solves the ODE until it finds the parameter set that minimizes the objective function.

Although this code was developed for Windkessel parameter estimation, the same framework can be applied to other similar problems by modifying:

- the **governing ODE**
- the **desired characteristics of the output waveform**

---

# MATLAB Code Structure

The MATLAB script is divided into several sections.

## Section A – Input Parameters

Section A defines the input data and configuration parameters required for estimating the Windkessel model parameters.

First, the flow waveform at the **ascending aorta (AAo)** is provided through a CSV file (`flowrate.csv`):

- Column 1: time \(T\)  
- Column 2: flow rate \(Q\) (mL/s)

Users can replace this file with their own measured or simulated waveform.

The parameter `QPercent` specifies the fraction of the AAo flow entering the outlet branch being modeled.

The desired systolic and diastolic pressures (`PMax`, `PMin`) define the **target pressure range** that the optimized Windkessel model should reproduce.

The compliance search range (`CMin`, `CMax`) defines bounds for the compliance parameter during optimization.

The initial pressure (`P0`) specifies the starting condition for the ODE solver.

To eliminate transient effects, multiple cardiac periods (`nT`) are simulated.

The time step (`dt`) is defined based on the cardiac period to ensure sufficient temporal resolution.

Finally, the geometric resistance (`Rgeo`) represents the resistance associated with the outlet geometry and should be obtained from **steady-flow simulation or measurement**.

## Section B – Pre-processing of Parameters

Section B performs several pre-processing steps to prepare the inputs required for the Windkessel parameter estimation.

First, the **average flow rate** over one cardiac cycle is computed.

Using the desired systolic and diastolic pressures, the **mean pressure** is estimated and used to calculate the **total vascular resistance**.

The geometric resistance (`Rgeo`) is then subtracted to obtain the resistance represented by the **Windkessel model**.

The flow waveform is repeated for multiple cardiac periods (`nT`) to generate a continuous flow profile, allowing transient effects to decay.

Finally:

- the flow waveform is **interpolated onto a uniform time grid**
- the **time derivative of flow** is computed

These quantities are required for solving the Windkessel model ODE during the optimization process.

## Section C – Optimization of Windkessel Parameters

Section C determines the optimal Windkessel model parameters using a **pattern search optimization algorithm**.

The optimization variables include:

- proximal resistance **R1** (corresponding to \(R_c\))
- compliance **C**

The distal resistance **R2** corresponds to \(R_p\) and is calculated as

$$
R_2 = R_{Total} - R_1
$$

For each candidate set of parameters, the Windkessel ODE is solved using the prescribed flow waveform.

The resulting pressure waveform is analyzed to obtain its **minimum and maximum values** over the final cardiac cycles.

An objective function compares these values with the target pressures (`PObjMin`, `PObjMax`). The pattern search algorithm iteratively adjusts the parameters until the objective function is minimized.

The optimization process returns the optimal values of **R1**, **R2**, and **C**.

## Section D – Pressure Waveform Verification

Section D evaluates the optimized parameters by solving the ODE again using the optimal values of `R1`, `R2`, and `C`.

The pressure waveform is plotted over the entire simulation period.

Since the last cycle may contain truncation effects from the flow waveform, the **second-to-last cardiac cycle** is used to evaluate the pressure extrema.

The maximum and minimum pressures from this cycle are printed and compared with the target pressures to verify the optimization result.

## Section E – Supporting Functions

Section E defines the functions used in the parameter estimation process.

**WK3ODE**

Represents the three-element Windkessel model ODE and computes the time derivative of pressure based on the current pressure, flow rate, and model parameters.

**RCtoODE**

Solves the ODE for a given set of Windkessel parameters and returns the minimum and maximum pressures from the final cardiac cycles. These values are used by the optimization algorithm to evaluate the objective function.

<a href="/completed-projects.html">← Back to Completed Projects</a>