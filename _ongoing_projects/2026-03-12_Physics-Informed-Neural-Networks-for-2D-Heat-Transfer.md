---
title: "Physics-Informed Neural Networks (PINNs) for 2D Heat Transfer"
date: 2026-03-12
summary: "An ongoing research project exploring Physics-Informed Neural Networks (PINNs) for solving heat transfer problems and accelerating traditional CFD simulations."
tags: [AI, PINN, Heat Transfer, Scientific Machine Learning, CFD]
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

<a href="/ongoing-projects.html">← Back to Ongoing Projects</a>

---

# Physics-Informed Neural Networks for Heat Transfer

## Introduction

With the rapid development of artificial intelligence, researchers have begun exploring the possibility of using neural networks to accelerate or even replace traditional numerical solvers. Classical computational fluid dynamics (CFD) and heat transfer simulations often require significant computational resources and long simulation times, especially for high-resolution or multi-physics problems.

Recently, **Physics-Informed Neural Networks (PINNs)** have emerged as a promising approach that integrates physical laws directly into neural network training. Instead of learning purely from data, PINNs embed governing equations such as partial differential equations (PDEs) into the loss function of a neural network.

This allows the model to respect the underlying physics while learning the solution of the system.

---

## What are Physics-Informed Neural Networks?

Physics-Informed Neural Networks combine deep learning with physical constraints.

For example, a neural network can be trained to approximate the solution of a PDE:

$$
\frac{\partial u}{\partial t} = \alpha \nabla^2 u
$$

Instead of solving the equation using traditional numerical methods such as finite difference, finite volume, or lattice Boltzmann methods, PINNs minimize the residual of the governing equation during training.

The loss function typically consists of several components:

- PDE residual loss
- Boundary condition loss
- Initial condition loss
- Data loss (if experimental or simulation data are available)

This framework enables the neural network to approximate the solution field while satisfying physical constraints.

---

## Project Goal

The goal of this project is to explore the capability of PINNs for solving heat transfer problems and evaluate their potential for accelerating traditional CFD workflows.

Specifically, this project focuses on:

- Implementing PINN models for heat transfer PDEs
- Investigating training stability and convergence behavior
- Comparing PINN solutions with traditional numerical methods
- Evaluating computational efficiency and accuracy

---

## Current Progress