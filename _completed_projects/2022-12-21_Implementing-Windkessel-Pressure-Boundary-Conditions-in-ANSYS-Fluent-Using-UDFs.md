---
title: "Implementing Windkessel Pressure Boundary Conditions in ANSYS Fluent Using UDFs"
date: 2022-12-21
summary: "Tutorial on implementing patient-specific Windkessel boundary conditions in ANSYS Fluent using UDFs and User-Defined Memory."
tags: [ANSYS Fluent, UDF, Windkessel Model, CFD, Hemodynamics]
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
    Implementing Windkessel Pressure Boundary Conditions in ANSYS Fluent Using UDFs
  </p>
</div>

The pressure boundary conditions provided in the ANSYS Fluent graphical user interface (GUI) primarily support constant values or predefined transient waveforms [1]. However, in many applications, the outlet pressure evolves dynamically in response to the instantaneous flowrate through the cross-section. In this work, we impose a pressure boundary condition whose value varies according to the following equation:

$$
\frac{dP(t)}{dt}
=
\frac{R_c+R_p}{R_p C}Q(t)
+
R_c\frac{dQ(t)}{dt}
$$

From the above equation, it can be seen that the pressure increment at the next time step depends on both the instantaneous flowrate and its temporal derivative, while the Windkessel parameters $R_c$, $R_p$, and $C$ are assumed to be known constants. The flowrate at the current time step can be obtained by integrating the flux over all faces on the outlet cross-section. However, evaluating the flowrate derivative requires the flowrate from the previous time step, which must therefore be stored in User-Defined Memory (UDM). The derivative can then be approximated using the following finite-difference expression:

$$
\frac{dQ(t)}{dt}
=
\frac{Q(t)-Q(t-\Delta t)}{\Delta t}
$$

ANSYS Fluent provides a large number of User-Defined Function (UDF) macros for customizing solver behavior. Readers interested in additional macros and their functionalities are referred to the Fluent UDF Manual [2].

Macros whose names begin with `DEFINE_XXXX` are special macros that expand into function definitions during preprocessing. Therefore, they cannot be nested within other `DEFINE_XXXX` macros [3] (i.e., placing one `DEFINE` macro inside another `DEFINE` macro). In contrast, most non-`DEFINE` macros provided in the manual can be invoked anywhere within a UDF.

In the present implementation, only two `DEFINE` macros are required:

- `DEFINE_PROFILE`
- `DEFINE_EXECUTE_AT_END`

`DEFINE_PROFILE` is used to prescribe boundary-condition profiles, such as velocity or pressure boundary conditions.

Its syntax is

```c
DEFINE_PROFILE(macroName, thread, index)
```

where:

- `macroName`: the name of the UDF. After the UDF is interpreted or compiled in Fluent, this name will appear in the corresponding boundary-condition dialog box, as illustrated in the figure below.
- `thread`: a pointer to the boundary zone (thread) to which the profile is applied.
- `index`: an integer index specifying which variable of the boundary condition is being modified.

In Fluent, a **thread** represents a mesh zone, such as an inlet, outlet, wall, or fluid region. Understanding the concept of threads is essential for writing UDFs because most boundary operations are performed through them.

After the UDF is successfully loaded, the name specified by `macroName` becomes available in the Fluent GUI and can be assigned to the desired boundary condition, like the following screenshot:

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/UDF-dialog-box.png" width="70%">
</div>

`thread` is provided by ANSYS Fluent. For example, when a `DEFINE_PROFILE` UDF is hooked to a boundary condition through the graphical user interface (GUI), Fluent automatically passes a pointer to the corresponding boundary zone (thread) [4]. This thread contains all mesh entities associated with that zone, such as faces and their related information. The user can then access and process these data within the UDF.

`index` identifies the variable to be defined. The value of `index` is determined when the UDF is hooked to a variable in the boundary-condition dialog box. ANSYS Fluent subsequently passes this index to the UDF so that the function knows which variable (e.g., velocity, pressure, temperature, etc.) it should operate on.

For example:

```c
DEFINE_PROFILE(boundary1, thread, index)
{
    face_t f;    // face_t is a Fluent data type representing a single face.

    begin_f_loop(f, thread)    // Loop over all faces in the specified thread.
    {
        F_PROFILE(f, thread, index) = 1.0;
        // Assign a value of 1.0 to the selected boundary variable on this face.
        // F_PROFILE is typically used together with DEFINE_PROFILE.
    }
    end_f_loop(f, thread);    // End of face loop.
}
```

In this example, the UDF loops over every face in the boundary zone represented by `thread` and assigns a constant value of `1.0` to the corresponding boundary variable identified by `index`.

The other `DEFINE` macro used in this case is `DEFINE_EXECUTE_AT_END()`, which is executed at the end of each iteration for steady simulations or at the end of each time step for transient simulations.

Unlike `DEFINE_PROFILE`, this macro does not take any input arguments and is not automatically provided with any domain or thread information. Therefore, the user must explicitly obtain the required domain and thread pointers within the UDF.

In this work, we use the pressure-based coupled solver for transient flow simulations. The following figure illustrates the execution flow of ANSYS Fluent together with the UDFs used to achieve the Windkessel coupling:

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/excution-flow-of-ANSYS.png" width="80%">
</div>

We do not use the `User-Defined Adjust` or `User-Defined Properties` hooks in this implementation, and therefore these steps can be ignored in the flowchart above.

During the `User-Defined Profile` stage, a `DEFINE_PROFILE` macro is assigned to each outlet boundary condition. At the end of each time step, the `DEFINE_EXECUTE_AT_END` macro is invoked to update the outlet pressure for the next time step, $P(t+\Delta t)$, and store the updated values in User-Defined Memory (UDM) [7].

As an example, consider a case with five outlets. Since one UDM is required to store the pressure history for each outlet, five UDM locations are needed for pressure storage.

Although Fluent provides macros for accessing cell-based quantities such as velocity and density from previous time steps, we did not find an equivalent macro for face-based quantities. Therefore, the flowrate on each face must be explicitly stored in UDM at the current time step and reused as the previous flowrate during the next time step.

Consequently, a total of ten UDM locations are required in this implementation: five for storing outlet pressures and five for storing face-based flow-rate information. The following table summarizes the purpose of each UDM location.

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/aortic-geometry-with-branch-name.png" width="70%">
</div>

| UDM Location | Definition |
|:------------:|:-----------|
| 0 | RSA $Q(t-\Delta t)$ |
| 1 | RCCA $Q(t-\Delta t)$ |
| 2 | DAo $Q(t-\Delta t)$ |
| 3 | LCCA $Q(t-\Delta t)$ |
| 4 | LSA $Q(t-\Delta t)$ |
| 5 | RSA $P(t+\Delta t)$ |
| 6 | RCCA $P(t+\Delta t)$ |
| 7 | DAo $P(t+\Delta t)$ |
| 8 | LCCA $P(t+\Delta t)$ |
| 9 | LSA $P(t+\Delta t)$ |

After the UDF has been implemented (the complete source code is provided in [8]), it can be imported into ANSYS Fluent. UDFs can be loaded using either the **interpreted** or **compiled** approach. Readers interested in the differences between these two methods are referred to the Fluent UDF Manual.

When using the compiled approach, it is recommended to replace `printf()` with `Message()`, especially for parallel simulations, as `Message()` provides more reliable output behavior in Fluent.

The following figures illustrate how to load and hook the UDF into the simulation.

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/use-the-interpreted-approach-to-import-the-UDF.png" width="70%">
</div>

<div style="text-align: center;">
  Use the interpreted approach to import the UDF.
</div>

<br>

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/hook-the-macro.png" width="70%">
</div>

<div style="text-align: center;">
  Hook the `DEFINE_PROFILE` macro to the corresponding pressure outlet boundary condition.
</div>

<br>

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/choose-execute-at-end.png" width="50%">
</div>

<div style="text-align: center;">
  Go to <strong>User-Defined → Function Hooks</strong>, click <strong>Edit...</strong> under <strong>Execute at End</strong>, and select `updateUDMs` (the `DEFINE_EXECUTE_AT_END` macro).
</div>

After the UDF has been successfully hooked, the Windkessel model is fully coupled with the transient CFD simulation.

The source code is provided in [8]. To adapt it to your own case, users may need to modify the following parameters:

- `R1Clinical`
- `R2Clinical`
- `CClinical`
- the outlet names (e.g., `ic`, `cm`, `mg`, etc.)

In addition, the thread IDs associated with the outlet boundaries must be updated to match those in your Fluent case. If your model contains a different number of outlets, corresponding modifications to the `DEFINE_PROFILE` macros and UDM assignments are also required.

Finally, we examine the simulation results. By coupling the Windkessel model with the CFD solver, physiologically realistic flow distributions among the outlets are successfully achieved. The predicted pressure waveform at the ascending aorta (AAo) generally agrees with physiological expectations. However, the AAo pressure waveform exhibits small numerical artifacts during the systolic upstroke. These features are likely related to limited temporal or spatial resolution, although further investigation is required to identify their exact cause.

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/result.png" width="80%%">
</div>






<a href="/completed-projects.html">← Back to Completed Projects</a>

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<div style="text-align: center; margin-top: 20px;">
  <span id="busuanzi_container_page_pv">
    👁️ Views: <span id="busuanzi_value_page_pv"></span><br>
    Powered by <a href="https://busuanzi.ibruce.info/" target="_blank" style="color: #007acc; text-decoration: none;">busuanzi</a>
  </span>
</div>