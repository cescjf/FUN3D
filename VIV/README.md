## Transient Fluid-Structure Analysis with FUN3D
In this tutorial, I looked at the Vortex Induced Vibration (VIV) of a cylinder in cross flow. In fluid dynamics, vortex induced vibrations are motions induced on bodies interacting with an external fluid flow, produced by – or the motion producing – periodical irregularities on this flow. This tutorial is my first attempt on solving the coupled fluid-structure interaction problem by coupled FUN3D with an external FEA solver.

## Finite Element Solver
I am using a simple truss-based FEA solver to get structural displacement for this problem. The time integration is done using the [Crank-Nicholson](https://en.wikipedia.org/wiki/Crank%E2%80%93Nicolson_method) method. FEA solver uses the same time step as the CFD solver; therefore, it cannot handle structures with high natural frequencies at this time. Please note that this is just a prove of concept. This example is not intended to be run for wide variety of structural properties. The structural model is shown in the following Figure.

<p align="center">
  <img src="https://github.com/kooroshg1/FUN3D/blob/master/VIV/figure/viv_physical_problem.jpg", height="300.0">
</p>

## Fluid-Structure Interaction Coupling
Fluid–structure interaction (FSI) is the interaction of some movable or deformable structure with an internal or surrounding fluid flow. Fluid–structure interactions can be stable or oscillatory. Here, I am using the following steps to couple the CFD and FEA solvers:

1. Solve and advance the fluid's equation one time step. Now we have the fluid's solution (pressure and velocity) at times `n` and `n+1`.
2. Integrate pressure on the cylinder surface to calculate the aerodynamic load. For this problem, I am only transferring lift force to the structural solver.
3. Calculate the position of the structure at time step `n+1` based on position at time `n` and loads at `n` and `n+1`. I am using Crank-Nicholson for this step.
4. Update the location of the structure and advance fluid's solution one time step.

## Specifying Aerodynamic Surfaces
The first step is to identify the surfaces that contribute to the aerodynamic load calculation. For the case of flow over cylinder, the cylinder surface is used for calculating aerodynamic loads.

I also need to write the aerodynamic surface to a text file. This file contains the coordinate and the `id` that fun3d assisgns to the said point. The aerodynamic surface file is moved at each iteration of the FSI solution based on the FEA solution. This updated surface location is used by fun3d solver as well. The aerodynamic surface is in `massoud` file format. I generate this file by adding the following lines to `fun3d.nml` file. Please note that `3` is the id of cylinder surface in `cylinder.mapbc` file.
```
&massoud_output
   n_bodies = 1
   nbndry(1) = 1
   boundary_list(1) = '3'
/
```
The massoud file is written by running the flow solver `nodet` using `--write_massoud_file` as command line option.

## FUN3D Coupled Fluid-Structure Interaction Solution
I coupled FUN3D with and external FEA solver by using the command line option (CLO) `--aeroelastic_external`. To make sure that `nodet` solver reads the updated aerodynamic surface position at each time step, I used `--read_surface_from_file` as well. `nodet` solver requires a shell script called `get_displacements_from_csd` to update the location of the aerodynamic surfaces. You need to provide this script.

The solution starts by solving the aerodynamic problem first. `nodet` solver will write a file named `new_airloads_are_ready` and waits before going to the next time step. At this point, the `get_displacements_from_csd` script is called. You can see that I am calling a Python FEA solver in my `get_displacements_from_csd` script. My Python FEA solver reads the aerodynamic loads from the `cylinder_ddfdrive_body1.dat` file generated by fun3d, calculate the new location of the structure, and updates the `cylinder_body1.dat` file. Since I am using `--read_surface_from_file` as CLO, fun3d will read this new surface location in the next time step. Please note that I am using the same step size for my FEA and CFD solvers. I am saving the solution history in `fsi_solution_history.txt` file. This file is later used for post processing.

## Running Coupled FSI Simulation
To run the FSI simulation you need to run the bash scipt `run-fsi`. You may need to change the access permission to this file using the following syntex
```
sudo chmod u+x run-fsi
```
`run-fsi` script will clean the directory and resets the cylinder location. It will also generate the `fsi_solution_history.txt` file for saving the solution history and initialize it. It will also assign the initial condition (initial displacement and velocity) that is used by the structural solver. `run-fsi` will then run the `nodet` solver with `--aeroelastic_external` and  `--read_surface_from_file` command line options.