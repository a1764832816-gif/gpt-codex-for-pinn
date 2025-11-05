# Implementation Plan for PINN Reproduction of Fluent Benchmark

This document captures the modeling information encoded in the legacy Fluent case and prescribes how to reproduce it within the physics-informed neural network (PINN) framework. Each subsection must be translated into code or configuration artifacts before model training begins.

## Geometry
- **Domain topology:** Rectangular channel of length *L* and height *H* with a heated obstruction centered at \(x = L/2\).
- **Obstruction:** Cylindrical post with diameter *D* extending the full height of the channel; the Fluent mesh resolves a rounded fillet at the base that must be preserved in any geometry parameterization.
- **Inlet/outlet extensions:** Upstream buffer region of \(2D\) and downstream buffer of \(6D\) to minimize boundary interactions; these lengths must be respected when sampling training points.
- **Coordinate system:** 2D Cartesian \((x, y)\) with the origin at the inlet centerline; non-dimensionalization uses \(D\) for length scaling.

## Governing Equations
- **Fluid flow:** Incompressible Navier–Stokes equations for steady laminar flow, expressed in non-dimensional form with Reynolds number \(Re\) and Prandtl number \(Pr\) specified from the Fluent material card.
  - Continuity: \(\nabla \cdot \mathbf{u} = 0\)
  - Momentum: \(\mathbf{u} \cdot \nabla \mathbf{u} = -\nabla p + \frac{1}{Re} \nabla^2 \mathbf{u}\)
- **Energy transport:** Convection–diffusion equation for temperature with uniform volumetric heat generation inside the obstruction.
  - \(\mathbf{u} \cdot \nabla T = \frac{1}{Re \cdot Pr} \nabla^2 T + S_T\)
- **Material properties:** Constant density, viscosity, and thermal conductivity, matching Fluent’s property table; the PINN must ingest these values from the property data deliverable.

## Boundary Conditions
- **Inlet:** Prescribed parabolic velocity profile with maximum velocity \(U_{max}\); uniform inlet temperature \(T_{in}\).
- **Outlet:** Zero normal stress (do-nothing) condition for velocity and zero-gradient for temperature.
- **Walls (channel walls and obstruction surface):** No-slip velocity and isothermal wall temperature \(T_w\). The obstruction additionally enforces the volumetric heat generation source term.
- **Symmetry:** If symmetry reduction is applied, enforce \(v = 0\) and \(\partial u/\partial y = 0\) along the centerline.
- **Initial condition for training:** Use Fluent’s converged field as a warm-start dataset when available; otherwise initialize with zero velocity and uniform temperature.

## Staged Training Workflow
The training reproduces the staged approach used in Fluent monitoring, ensuring stable convergence of the PINN.

### Phase A – Geometry & Boundary Priming
1. Sample boundary and interior collocation points emphasizing geometry features (e.g., obstruction fillet).
2. Train the PINN with increased loss weights on boundary condition residuals while freezing energy equation terms.
3. Converge until boundary residuals drop below the Fluent mesh convergence threshold.

### Phase B – Coupled Flow Solution
1. Activate full Navier–Stokes residuals with moderate weighting; continue enforcing boundary conditions.
2. Introduce adaptive sampling around recirculation regions observed in the Fluent solution.
3. Monitor drag coefficient and pressure drop metrics, matching Fluent values within 2% before proceeding.

### Phase C – Thermal Coupling and Fine-Tuning
1. Unfreeze the energy equation residual and apply the volumetric heat source inside the obstruction.
2. Balance loss weights across momentum and energy equations using Fluent residual norms as targets.
3. Continue training with curriculum on collocation density until heat flux at the walls matches Fluent post-processing within 3%.
4. Export the converged network weights and inference script as final deliverables.

## Verification Checklist
- Compare velocity, pressure, and temperature profiles at predefined probe lines against Fluent reference data.
- Validate integral quantities (drag, Nusselt number) against Fluent reports.
- Document training hyperparameters, sampling strategies, and checkpoints for reproducibility.
