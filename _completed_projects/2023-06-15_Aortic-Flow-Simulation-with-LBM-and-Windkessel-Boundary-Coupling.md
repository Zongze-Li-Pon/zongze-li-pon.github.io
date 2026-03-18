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

├── myScript.cpp

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

<div align="center"><strong>a. FluidEdgeFetcher3D</strong></div>

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

If a fluid node is associated with an opening but not with the wall, it is stored as an **edge node** of that opening.

These nodes are useful for:

- applying opening-related boundary conditions
- computing opening-averaged quantities

If a fluid node is influenced by both the wall and an opening, it is classified as a **corner node**.

This distinction is useful because corner nodes are geometrically special: they lie near the junction between wall and opening surfaces, and treating them separately can improve clarity in both post-processing and boundary tagging.

**Data Structures**

The class stores the collected nodes in several containers.

`edgeNodes`

This is a vector of vectors:

- `edgeNodes[0]` stores wall edge nodes
- `edgeNodes[1]`, `edgeNodes[2]`, ... store opening edge nodes according to boundary tags

`cornerNodes`

This is also a vector of vectors:

- `cornerNodes[iTag]` stores corner nodes associated with opening `iTag`
- `cornerNodes[0]` is empty in the current definition

`tagBlock`

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

<div align="center"><strong>b. OpenValueFetcher3D</strong></div>

**Purpose:**
Once the boundary-adjacent fluid nodes have been identified and tagged, `OpenValueFetcher3D` uses those tagged node sets to extract physically meaningful quantities at each opening.

In this implementation, the class is mainly used to compute:

- opening flow rate
- opening-averaged density

**Initialization**

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

**Flow Rate Evaluation**

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

**Average Density Evaluation**

The member function

- `getAvRho(...)`

computes the average density at each opening.

The procedure is simpler:

1. Use Palabos to compute the full density field
2. Extract all nodes belonging to the current opening tag
3. Compute the average density over those tagged nodes

This average density can then be used in later coupling procedures, such as pressure-related outlet treatment.

**Why These Utilities Matter**

Although these two classes are relatively compact compared with the full solver, they play an important role in the overall framework.

They provide:

- a reusable way to identify opening-related fluid nodes near off-lattice boundaries
- a clean way to distinguish wall, opening, and corner contributions
- a convenient interface for extracting averaged opening quantities
- a modular toolset that can support customized outlet boundary conditions such as Windkessel coupling

In other words, these utilities make it much easier to connect the geometric representation of vascular outlets with the lattice-based field data required for reduced-order boundary modeling.

## 2. Custom Surface Force Evaluation in SurfaceForceTools.cpp

In patient-specific vascular simulations with **off-lattice boundaries**, evaluating wall quantities such as **surface force**, **pressure**, and **wall shear stress (WSS)** is an important part of the post-processing workflow.

Palabos already provides a built-in method for computing surface force on triangulated boundaries. However, when the geometry is complex and the simulation is performed as an **internal-flow problem**, the interpolation stencil around a boundary vertex may include lattice cells that do not belong to the active fluid region.

This may lead to less physically consistent interpolation near the wall, especially when some of the neighboring cells are outside the usable fluid domain.

To address this, I implemented a customized surface-force evaluation method in `SurfaceForceTools.cpp`. The key idea is to modify the interpolation procedure so that only cells belonging to the current flow region are used in the force reconstruction.

This file mainly contains three parts:

- `ComputeParticleForce3DInnerborder`
- `computeSurfaceForceInnerBorder(...)`
- `writeSurF(...)`

Together, these components provide a custom workflow for evaluating and exporting surface force information based only on **inner-border / usable fluid nodes**.

<div align="center"><strong>a. ComputeParticleForce3DInnerborder</strong></div>

**Purpose:**  
`ComputeParticleForce3DInnerborder` is a customized processing functional that computes surface force at boundary vertices using only the subset of neighboring lattice cells that belong to the current flow region.

Compared with the original Palabos implementation, which may use all eight surrounding nodes in 3D interpolation, this version only keeps cells whose voxel flags are:

- `voxelFlag::bulkFlag(flowType)`
- `voxelFlag::borderFlag(flowType)`

In other words, the interpolation excludes neighboring cells that do not belong to the active fluid region.

This is especially useful for internal-flow simulations, where the geometric neighborhood of a wall vertex may include cells outside the physically meaningful interpolation region.

**Core Idea**

The processor works on three input fields:

- particle field
- lattice field
- voxel flag field

For each particle attached to a mesh vertex, the code reconstructs local macroscopic quantities and computes the force acting on the wall.

The logic can be summarized as follows.

**Step 1 – Create particles on mesh vertices**

The surface-force workflow first creates one visualization particle at each triangulated boundary vertex. These particles serve as carriers of wall-related quantities such as:

- force vector
- pressure
- wall shear stress

Each particle is tagged by its corresponding mesh vertex ID.

**Step 2 – Build interpolation stencil**

For a given boundary vertex, the code evaluates the interpolation coefficients associated with the surrounding lattice cells.

In standard trilinear interpolation, up to 8 neighboring cells may contribute in 3D.

However, not all of these cells are necessarily valid fluid cells for the current flow configuration.

**Step 3 – Keep only usable cells**

Each candidate interpolation cell is checked against the voxel flag field.

Only cells marked as belonging to the current flow region are accepted:

- bulk fluid cells
- border fluid cells

All other cells are discarded from the interpolation.

This is the key modification compared with the original implementation.

**Step 4 – Shift inward if necessary**

If no usable cells are found at the initial boundary-vertex position, the code slightly shifts the evaluation point inward along the surface normal and retries the interpolation.

This procedure is repeated for a few trials.

The purpose of this step is to improve robustness near difficult geometric locations where the original vertex position does not immediately yield a valid interpolation stencil.

**Step 5 – Reconstruct the local lattice state**

Once a non-empty subset of usable cells is found, the code constructs an interpolated cell state at the vertex.

Instead of using all 8 neighboring cells, the interpolated distribution is reconstructed only from the usable subset, and the weights are normalized accordingly.

This ensures that the reconstructed state is based only on fluid cells relevant to the current domain.

**Step 6 – Compute macroscopic quantities**

After the interpolated cell state is obtained, the code computes:

- density-related quantity
- momentum
- non-equilibrium stress tensor

Using the local surface normal, it then evaluates the traction contribution associated with the vertex.

From this, the following quantities are extracted:

- surface force vector
- pressure
- wall shear stress (WSS)

The wall force is obtained by applying Newton’s third law to the force acting on the fluid.

**Fallback Handling**

If no usable interpolation cells can be found even after several inward shifts, the code assigns fallback values:

- zero wall force
- default pressure
- zero wall shear stress

This prevents the program from failing at isolated problematic points and keeps the output field well-defined.

**Why This Modification Matters**

The main motivation of this customized implementation is that, in internal-flow simulations, the original interpolation stencil may contain cells outside the physically meaningful fluid region.

By restricting interpolation to:

- `bulkFlag(flowType)`
- `borderFlag(flowType)`

the reconstructed wall quantities become more consistent with the actual active flow domain.

This is especially helpful for:

- narrow vascular branches
- highly curved wall regions
- complex patient-specific outlet geometries

where geometric interpolation near the wall can otherwise be sensitive to stencil contamination.

<div align="center"><strong>b. computeSurfaceForceInnerBorder</strong></div>

**Purpose:**  
The function `computeSurfaceForceInnerBorder(...)` provides a higher-level interface for applying the customized surface-force processor to the full triangulated boundary.

Its role is to generate a particle field carrying surface-force information computed by the inner-border-only method.

Two overloads are provided:

- one for the full computational domain
- one for a user-specified sub-domain

This makes the routine flexible for both global output and localized evaluation.

**Workflow**

The function performs the following steps:

1. Select the appropriate boundary mesh  
   - static open mesh
   - dynamic open mesh

2. Create a particle field associated with the lattice multi-block structure

3. Generate one particle per boundary vertex using:
   - `CreateParticleFromVertex3D`

4. Apply the customized processor:
   - `ComputeParticleForce3DInnerborder`

5. Restore the original boundary selection

The final output is a particle field in which each boundary vertex carries:

- force vector
- pressure
- wall shear stress

This particle-based representation is then ready for VTK export or further analysis.

**Why a Separate Wrapper is Useful**

Instead of directly embedding all logic into one function, this wrapper separates:

- particle generation
- processor application
- mesh selection management

This makes the code cleaner and easier to reuse in different post-processing workflows.

<div align="center"><strong>c. writeSurF</strong></div>

**Purpose:**  
The function `writeSurF(...)` writes surface-force results into VTK files for visualization and comparison.

An important feature of this function is that it outputs **two versions** of the surface-force result:

1. the standard Palabos surface-force evaluation  
2. the customized inner-border-only evaluation

This side-by-side output is very useful for debugging, verification, and assessing the effect of the modified interpolation strategy.

**Output Quantities**

The exported fields include:

- `pressure`
- `wss`
- `force(Pa)`

Appropriate scaling factors are also applied using the unit-conversion object `GetParam<T>` so that the written quantities are expressed in physical units.

**Workflow**

The function first determines whether it is called during iteration or as a standalone post-processing step, and prints corresponding status information.

It then writes:

- one VTK file using the standard Palabos `computeSurfaceForce(...)`
- one VTK file using the custom `computeSurfaceForceInnerBorder(...)`

The custom result is written with an `"IB"` suffix in the filename, indicating the **inner-border-only** version.

This naming convention makes it easy to compare the two outputs during post-processing.

**Why This Output Function Matters**

This function is not only a convenience wrapper for file writing. It also reflects an important design decision in the project:

the customized implementation is intended to be compared directly with the standard Palabos result, rather than replacing it blindly.

This provides a transparent way to:

- validate the modified force evaluation
- inspect differences between the two methods
- choose the more appropriate output for specific hemodynamic analyses

**Summary**

`SurfaceForceTools.cpp` extends the original Palabos surface-force evaluation by introducing a more selective interpolation strategy near off-lattice boundaries.

The three key components serve different roles:

- `ComputeParticleForce3DInnerborder`  
  performs the customized force reconstruction using only valid fluid-side cells

- `computeSurfaceForceInnerBorder(...)`  
  builds the corresponding particle-based surface-force field

- `writeSurF(...)`  
  exports both the standard and custom results for visualization and comparison

Overall, this file improves the robustness and physical consistency of wall-quantity evaluation in complex internal-flow simulations, particularly for patient-specific vascular geometries.

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