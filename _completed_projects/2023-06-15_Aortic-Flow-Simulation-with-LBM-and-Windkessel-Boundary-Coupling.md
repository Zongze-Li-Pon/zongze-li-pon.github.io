---
title: "Aortic Flow Simulation with LBM and Windkessel Boundary Coupling"
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

## Introduction

This project presents a C++ implementation of a **three-element Windkessel (WK3) model** coupled with a pressure boundary condition for **Lattice Boltzmann Method (LBM)** simulations, built upon the open-source framework [**Palabos**](https://palabos.unige.ch/).

The full source code is available in the following repository: 👉 [**palabos-aortic-flow-windkessel3**](https://github.com/Zongze-Li-Pon/palabos-aortic-flow-windkessel3)


In patient-specific hemodynamic simulations, prescribing realistic outlet boundary conditions is essential to capture the effect of downstream vasculature. The Windkessel model provides a reduced-order representation of vascular impedance and enables **multi-scale coupling between 0D lumped models and 3D CFD simulations**.

This code integrates the Windkessel model into the Palabos framework and extends its capability for:

- pressure outlet boundary condition coupling
- surface force evaluation on complex geometries
- flexible data interpolation and waveform reconstruction

Beyond the specific application to aortic flow simulation, the structure of this code is designed to be **modular and reusable**, allowing extension to other boundary conditions, solvers, and multi-physics problems.

---

## Project Structure

The repository is organized as follows:

palabos-aortic-flow-windkessel3/

├── myTools/

│ ├── FluidEdgeFetcher3D.cpp

│ ├── SurfaceForceTools.cpp

│ ├── Interpolation.cpp

│ ├── WindkesselModel.cpp

│ ├── SimulationSetup.cpp

│ └── PostProcessingTools.cpp

├── myScript

├── param.xml

├── CMakeLists.txt

└── README.md

---

## 1. FluidEdgeFetcher3D and OpenValueFetcher3D in FluidEdgeFetcher3D.cpp

One important difficulty in patient-specific vascular simulations with **off-lattice boundary conditions** is that many opening-related quantities are not directly available in a convenient form.

For example, when evaluating outlet flow rate or average pressure/density at a vascular opening, one often needs a reliable way to identify the set of **fluid nodes located immediately adjacent to the triangulated boundary surface**. While Palabos internally handles geometric boundary intersections, it does not directly provide a reusable tool for collecting and organizing these fluid nodes in a form convenient for custom boundary treatment and post-processing.

To address this, I implemented two utility classes:

- `FluidEdgeFetcher3D`
- `OpenValueFetcher3D`

These two classes work together to identify boundary-adjacent fluid nodes, classify them according to opening and wall information, and then use those tagged nodes to extract physically meaningful opening quantities such as **flow rate** and **average density**.

<div align="center">a. FluidEdgeFetcher3D</div>

**Purpose:**
`FluidEdgeFetcher3D` is designed to locate and classify fluid nodes that lie next to the off-lattice triangulated boundary.

Its design follows a strategy similar to the Palabos off-lattice treatment: for each candidate fluid node near the boundary, the code checks neighboring lattice directions and determines whether the segment connecting the fluid node to a neighboring solid node intersects the triangulated surface mesh.

If an intersection exists, the node is considered associated with the corresponding boundary surface.

The logic of `FluidEdgeFetcher3D` can be summarized as follows.

**Step 1 – Scan the voxelized domain**

The constructor loops over the voxelized computational domain and searches for cells marked as **border nodes** in the voxel matrix.

These border nodes are the most relevant candidates because they are located near the triangulated boundary and may participate in boundary-related operations.

**Step 2 – Keep only fluid nodes**

For each candidate location, the code first checks whether the current cell belongs to the fluid region. This is determined according to the selected flow type:

- `voxelFlag::inside`
- `voxelFlag::outside`

This makes the utility flexible enough to support either internal-flow or external-flow configurations.

**Step 3 – Detect boundary intersections**

For each fluid node, the code loops over all lattice directions except the rest population.

If a neighboring node is solid, the code tests whether the segment from the current fluid node to that neighboring solid node intersects the triangulated mesh. This is implemented in:

- `pointOnMesh(...)`

If an intersection is found, the triangle ID is obtained, and the corresponding boundary tag is retrieved from the Palabos boundary object.

This step establishes the connection between:

- the **lattice fluid node**
- the **surface mesh**
- the **boundary tag** associated with a wall or opening

**Edge Nodes and Corner Nodes**

After counting how many intersecting directions belong to each boundary tag, the node is classified into one of two categories.

### Edge nodes

If a fluid node is associated with an opening but not with the wall, it is stored as an **edge node** of that opening.

These nodes are useful for:

- applying opening-related boundary conditions
- computing opening-averaged quantities

### Corner nodes

If a fluid node is influenced by both the wall and an opening, it is classified as a **corner node**.

This distinction is useful because corner nodes are geometrically special: they lie near the junction between wall and opening surfaces, and treating them separately can improve clarity in both post-processing and boundary tagging.

**Data Structures**

The class stores the collected nodes in several containers.

### `edgeNodes`

This is a vector of vectors:

- `edgeNodes[0]` stores wall edge nodes
- `edgeNodes[1]`, `edgeNodes[2]`, ... store opening edge nodes according to boundary tags

### `cornerNodes`

This is also a vector of vectors:

- `cornerNodes[iTag]` stores corner nodes associated with opening `iTag`
- `cornerNodes[0]` is empty in the current definition

### `tagBlock`

A scalar field is generated to store the tag of each collected node.  
This block is especially useful for:

- averaging quantities over selected opening regions
- later post-processing
- visualization in VTK format

The class also provides a `getCornerBlock()` function, which generates a visualization block where:

- wall nodes are tagged by a wall ID
- opening edge nodes are tagged by positive opening IDs
- corner nodes are tagged by negative opening IDs

This makes it easy to visually inspect the classification result.

**User-Level Functions**

Several user-level functions are provided to make the extracted data easy to use.

- `getEdge()` returns all edge-node lists
- `getEdge(iOpen)` returns edge nodes for a specific opening in user-defined order
- `getCorner()` returns all corner-node lists
- `getCorner(iOpen)` returns corner nodes for a specific opening
- `getWhole(iOpen)` returns the union of edge and corner nodes for that opening
- `getTagBlock()` returns the tag field for later averaging or post-processing
- `getCornerBlock()` returns a visualization-ready scalar block

Overall, `FluidEdgeFetcher3D` serves as a bridge between the triangulated surface description and the lattice representation of the fluid domain.

### 2. OpenValueFetcher3D

**Purpose:**  
Once the boundary-adjacent fluid nodes have been identified and tagged, `OpenValueFetcher3D` uses those tagged node sets to extract physically meaningful quantities at each opening.

In this implementation, the class is mainly used to compute:

- opening flow rate
- opening-averaged density

---

### Initialization

The constructor of `OpenValueFetcher3D` takes:

- a tagged scalar field `tagBlock`
- the Palabos boundary object
- a user-defined opening sorting direction `holeSort`

Using the boundary object, it retrieves the geometric properties of all lids:

- lid normals
- lid centers
- lid radii

It also stores the opening-tag order so that later queries follow a user-defined physical ordering, rather than the raw internal tag numbering.

This is important because the tag order generated by the mesh may not match the desired anatomical or geometric ordering of the outlets.

---

### Flow Rate Evaluation

The member function

- `getQ(...)`

computes the flow rate at each opening.

The procedure is:

1. Use Palabos to compute the full velocity field
2. Average the velocity over all nodes carrying the tag of the current opening
3. Project the averaged velocity onto the lid normal direction
4. Multiply by the opening area

Mathematically, the flow rate is evaluated as

$$
Q = \left( \bar{\mathbf{u}} \cdot \mathbf{n} \right) A
$$

where:

- $\bar{\mathbf{u}}$ is the average velocity over the tagged opening nodes
- $\mathbf{n}$ is the lid normal
- $A$ is the opening area

This provides an efficient way to estimate opening flow rate directly from the tagged fluid-node set.

---

### Average Density Evaluation

The member function

- `getAvRho(...)`

computes the average density at each opening.

The procedure is simpler:

1. Use Palabos to compute the full density field
2. Extract all nodes belonging to the current opening tag
3. Compute the average density over those tagged nodes

This average density can then be used in later coupling procedures, such as pressure-related outlet treatment.

---

### Why These Utilities Matter

Although these two classes are relatively compact compared with the full solver, they play an important role in the overall framework.

They provide:

- a reusable way to identify opening-related fluid nodes near off-lattice boundaries
- a clean way to distinguish wall, opening, and corner contributions
- a convenient interface for extracting averaged opening quantities
- a modular toolset that can support customized outlet boundary conditions such as Windkessel coupling

In other words, these utilities make it much easier to connect the geometric representation of vascular outlets with the lattice-based field data required for reduced-order boundary modeling.

---

### Summary

Together, `FluidEdgeFetcher3D` and `OpenValueFetcher3D` form a small but important infrastructure layer in this project.

- `FluidEdgeFetcher3D` identifies and classifies fluid nodes near walls and openings
- `OpenValueFetcher3D` uses these tagged nodes to evaluate opening quantities such as flow rate and average density

This design improves both the modularity and the usability of the overall codebase, especially for patient-specific vascular simulations with complex outlet boundary conditions.

---

### 2. Surface Force Evaluation

**File:** `SurfaceForceTools.cpp`

This module provides a modified implementation for computing surface forces based on the Palabos framework.

Compared to the default implementation, this version is designed to:

- improve flexibility for complex geometries  
- integrate more naturally with custom boundary conditions  
- support post-processing of forces on vascular surfaces  

It is especially useful for analyzing hemodynamic quantities such as pressure distribution and wall forces.

---

### 3. Interpolation Utility

**File:** `Interpolation.cpp`

This module implements a standalone interpolation utility in C++.

Unlike Palabos-specific tools, this module is designed to be **fully reusable outside the framework**. It supports:

- interpolation of time-dependent waveforms  
- reconstruction of variables from prescribed signals  
- user-defined order of accuracy  

This is particularly useful for handling inlet/outlet boundary data and time-varying flow conditions.

---

### 4. Three-Element Windkessel Model

**File:** `WindkesselModel.cpp`

This module implements the **three-element Windkessel model (WK3)**.

The model includes:

- proximal resistance $R_c$  
- distal resistance $R_p$  
- compliance $C$  

This implementation is independent of Palabos and can be reused in other CFD solvers or reduced-order modeling frameworks.

Within this project, it is coupled with the pressure boundary condition to provide a physiologically realistic outlet model.

---

### 5. Auxiliary Modules

**Files:**
- `PostProcessingTools.cpp`  
- `SimulationSetup.cpp`  

These modules are introduced to improve code organization and readability.

They are responsible for:

- simulation setup and parameter initialization  
- separation of numerical logic from workflow control  
- post-processing and data handling  

This separation ensures that the main solver remains clean and maintainable, while also making the code easier to extend.

---


<a href="/completed-projects.html">← Back to Completed Projects</a>

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<div style="text-align: center; margin-top: 20px;">
  <span id="busuanzi_container_page_pv">
    👁️ Views: <span id="busuanzi_value_page_pv"></span><br>
    Powered by <a href="https://busuanzi.ibruce.info/" target="_blank" style="color: #007acc; text-decoration: none;">busuanzi</a>
  </span>
</div>