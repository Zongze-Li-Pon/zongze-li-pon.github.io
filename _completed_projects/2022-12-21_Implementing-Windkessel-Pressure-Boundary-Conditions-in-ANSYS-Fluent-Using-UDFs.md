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

In this work, we use the pressure-based coupled solver for transient flow simulations. The following figure illustrates the execution flow of ANSYS Fluent together with the UDFs used to achieve the Windkessel coupling.

<div style="text-align: center;">
  <img src="/completed-projects/2022-12-21_Implementing-Windkessel-Pressure-Boundary-Conditions-in-ANSYS-Fluent-Using-UDFs/excution-flow-of-ANSYS.png" width="80%">
</div>

We do not use the `User-Defined Adjust` or `User-Defined Properties` hooks in this implementation, and therefore these steps can be ignored in the flowchart above.

During the `User-Defined Profile` stage, a `DEFINE_PROFILE` macro is assigned to each outlet boundary condition. At the end of each time step, the `DEFINE_EXECUTE_AT_END` macro is invoked to update the outlet pressure for the next time step, $P(t+\Delta t)$, and store the updated values in User-Defined Memory (UDM) [7].

As an example, consider a case with five outlets. Since one UDM is required to store the pressure history for each outlet, five UDM locations are needed for pressure storage.

Although Fluent provides macros for accessing cell-based quantities such as velocity and density from previous time steps, we did not find an equivalent macro for face-based quantities. Therefore, the flow rate on each face must be explicitly stored in UDM at the current time step and reused as the previous flow rate during the next time step.

Consequently, a total of ten UDM locations are required in this implementation: five for storing outlet pressures and five for storing face-based flow-rate information. The following table summarizes the purpose of each UDM location.














<a href="/completed-projects.html">← Back to Completed Projects</a>

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<div style="text-align: center; margin-top: 20px;">
  <span id="busuanzi_container_page_pv">
    👁️ Views: <span id="busuanzi_value_page_pv"></span><br>
    Powered by <a href="https://busuanzi.ibruce.info/" target="_blank" style="color: #007acc; text-decoration: none;">busuanzi</a>
  </span>
</div>