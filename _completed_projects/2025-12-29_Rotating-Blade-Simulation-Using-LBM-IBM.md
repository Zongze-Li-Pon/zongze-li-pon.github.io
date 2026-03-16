---
title: "Rotating Wing Simulation using LBM–IBM"
date: 2026-03-15
summary: "Numerical reconstruction of a rotating-wing experiment using the Lattice Boltzmann Method coupled with the Immersed Boundary Method."
tags: [CFD, Lattice Boltzmann Method, Immersed Boundary Method, Rotating Wing, Low Reynolds Number Aerodynamics]
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
    Rotating Wing Simulation using LBM–IBM
  </p>
</div>

---

# Project Overview

This project presents a **numerical reconstruction of the rotating-wing experiment by Manar & Jones (2014)**.

The simulation is implemented using the **Lattice Boltzmann Method (LBM)** coupled with the **Immersed Boundary Method (IBM)** to model rotating wings at low Reynolds numbers.

Although the setup originates from a classical experimental study, the goal here is not only to reproduce the original results but also to demonstrate how **LBM–IBM can be applied to more general rotating-blade problems**.

The simulation produces detailed information such as:

- vortex structures
- pressure distributions
- transient flow fields

These quantities are often difficult to measure experimentally.

Such simulations are relevant to applications including

- rotor aerodynamics
- turbomachinery blade analysis
- conceptual aircraft engine safety studies

---

# Geometry Model Construction

In the original experiment, the rotating wing was attached to the shaft using a thin supporting rod.

In the numerical simulation this auxiliary structure is unnecessary.  
The wing can be rotated directly about its axis, which allows us to isolate the aerodynamic effects generated purely by the wing itself.

This is one advantage of numerical simulation compared with physical experiments.

The wing geometry used in the simulation is:

| Parameter | Value |
|---|---|
| chord $c$ | 7.62 cm |
| span $s$ | 15.24 cm |
| thickness $h$ | 0.254 cm |

The wing is placed at an **angle of attack of 45°** and rotates around a shaft located at

$$
r_t = 3.81 \text{ cm}
$$

from the wing root.

---

# Reynolds Number Definition

The Reynolds number used in the rotating-wing simulation is

$$
Re =
\frac{\Omega_{max} R_{75} c}{\nu}
$$

where

- $\Omega_{max}$ : maximum angular velocity  
- $\nu$ : kinematic viscosity  
- $c$ : wing chord  
- $R_{75}$ : reference radius

The reference radius is defined as

$$
R_{75} = 0.75 (r_t + s)
$$

This definition follows the convention used in the original experiment.

---

# Resolution Convergence Check

A simple **resolution convergence study** was performed.

For this check:

- Reynolds number: **Re = 120**
- angular velocity: **constant**

The original paper used a **time-dependent profile**, but here a constant angular velocity is used to simplify the resolution test.

Simulations were conducted with

40–100 nodes

along the spanwise direction.

The simulation becomes **mesh independent** as resolution increases.  
Finally,

$$
N = 70
$$

was chosen for all cases.

---

# Transient Kinematics

The original experiment divides the rotational motion into three phases:

1. acceleration
2. constant angular velocity
3. deceleration

During acceleration and deceleration the angular velocity changes linearly.

To avoid infinite acceleration caused by discontinuous changes, a **smoothing function** is applied.

The angular velocity profile is defined as

$$
\Omega(t) =
\begin{cases}

\frac{\Omega_{max}}{2a(t_2 - t_1)}
\log
\left(
\frac{\cosh[a(t-t_1)]}{\cosh[a(t-t_2)]}
\right)
+
\frac{\Omega_{max}}{2}

&
0 \le t \le \frac{t_2 + t_3}{2}
\\

-\frac{\Omega_{max}}{2a(t_4 - t_3)}
\log
\left(
\frac{\cosh[a(t-t_3)]}{\cosh[a(t-t_4)]}
\right)
+
\frac{\Omega_{max}}{2}

&
\frac{t_2 + t_3}{2} \le t

\end{cases}
$$

where

- $a$ : smoothing parameter (set to **50**)  
- $t_1, t_2$ : start and end of acceleration  
- $t_3, t_4$ : start and end of deceleration

The wing completes **two full revolutions** during the motion.

---

# Animation

The animations below illustrate two types of simulations.

**Constant case**

Used for the **resolution convergence check**.

**Transient case**

Designed to reproduce the **time-dependent motion described in the original paper**.

---

# Final Notes

The simulations were conducted using **Palabos** [2] together with our own **rigid-body solver package**.

The animations were generated using **ParaView** [3], while the geometry and prescribed transient motion were created using **MATLAB** [4] scripts.

The solid body solver package is **not publicly available at this time**.

Once potential conflicts of interest are resolved, we may consider releasing it as **open-source in the future**.

---

# References

[1] Manar, F., & Jones, A. R. (2014).  
*The effect of tip clearance on low Reynolds number rotating wings.*  
AIAA Aerospace Sciences Meeting.

[2] J. Latt et al.  
*Palabos: Parallel lattice Boltzmann solver.*  
Computers & Mathematics with Applications, 81, 334–350 (2021).

[3] J. Ahrens, B. Geveci, and C. Law.  
*ParaView: An End-User Tool for Large Data Visualization.*  
The Visualization Handbook (2005).

[4] MATLAB, version R2024a.  
The MathWorks Inc., Natick, Massachusetts, United States.

---

<a href="/completed-projects.html">← Back to Completed Projects</a>

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<div style="text-align: center; margin-top: 20px;">
  <span id="busuanzi_container_page_pv">
    👁️ Views: <span id="busuanzi_value_page_pv"></span><br>
    Powered by <a href="https://busuanzi.ibruce.info/" target="_blank" style="color: #007acc; text-decoration: none;">busuanzi</a>
  </span>
</div>