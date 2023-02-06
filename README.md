# Making the domain decomposition X library highly performing - Google Summer School of Code 2023

In our research group, we have developed a library for performing
numerical simulation of the solvents in computational chemistry and
computational biophysics. The library uses tools from mathematics to
handle in a very efficient way classical electrostatic problems.
The library is released under the LGPL-3.0 license on GitHub:
https://github.com/ddsolvation/ddX.

In this brief page we present a project for the Google Summer School
of Code, together with a bit of context to fully grasp what we are
doing. First we present the modeling problem at the heart,
then the library itself, and only at the end we present the project
ideas.

## Overview

### The modeling problem

The computational modeling of molecular systems is a complex task:
  - accurate models based on **quantum mechanics** have a steeply
  scaling computational cost in the number of atoms in the system: their
  application is only possible on system with **less than a few
  hundred atoms**
  - most of the interesting chemistry and biochemistry happened in
  **condensed phase**, where the interesting molecule is surrounded by
  other molecules which influence its properties: a proper modeling
  need to take into account **more than a few hundred atoms**

### The multiscale solution

One of the strategy to deal with this kind of systems is to use
**focused models**, in which the interesting region is described in great
accuracy and the rest of the environment is treated with cheaper
models. [^1]

In particular, the environment can be described as a continuum medium
characterized only by a few electrostatic properties such as the
**polarizability** of the medium and the **ion concentration** in
it. [^2]

In this case, the modeling problem is turned into a **classical
electrostatic problem** which can be tackled by using the Maxwell
equations.

Depending on the dielectric constant and ion concentration we
distinguish three different models:
  - Infinite dielectric constant: **COSMO** (conductor like screening
  model)
  - Finite dielectric constant, zero ion concentration: **PCM**
  (polarizable continuum model)
  - Finite dielectric constant, nonzero ion concentration: **LPB**
  (linearized Poisson-Boltzmann).

### Domain Decomposition

An innovative idea to solve this kind of electrostatic problems is to
use the **domain decomposition** technique. In general, the equations
involved are quite simple and well known in the mathematical literature,
but at the same time, the domain on which they are defined are rather
complicated thus making the problem difficult.

In the domain decomposition strategy the global problems are recast as
a collection of individual, simpler problems, each of them defined on a
given atom of the solute. These can be easily discretized such that the
original problem is transformed in a linear system in the form

    L x = f

where L is a matrix depending on the geometry of the system, x is
the desired solution and f is a known right hand side which depend on
the geometry and on the kind of atoms that compose the solvated
molecule. [^3]

Once x is known, the effect of the solvent on the solvated molecule is
fully described.

### The ddX library

In the domain decomposition X library (ddX), we have implemented
numerical methods to solve the COSMO, PCM and LPB problems in a
domain decomposition fashion. Furthermore, we have also implemented
all the common framework between the three models as well as APIs
to couple this library with other software packages used in
computational chemistry and biochemistry.

The library is written in Fortran, with particular care for an efficient
implementation. The APIs are exposed to Fortran, to C and to Python.

## Project

The three methods implemented by the ddX library have already proved
to be accurate and fast. In particular the domain decomposition COSMO is
the fastest code in the world able to fully characterize a polarizable
solvent.

Despite the good results, further improvement is still possible and it
is the goal of the present project. Making the library even more
performant would allow simulating huge biological systems, such as
entire viruses, as well as running the calculation on high performance
computer clusters.

The generic goal of optimizing the ddX library can be achieved in
several ways, for this reason we organized the project as three
independent work packages.
This is an advantage, as the project can still be considered a success
even if one work package is not completed. As a consequence, the project
particularly robust against unexpected difficulties.

**Mentor:** [Michele Nottoli](https://github.com/mnottoli) (michele.nottoli@mathematik.uni-stuttgart.de)

**Prerequisites:** interest in scientific computing and high performance
computing. Basic knowledge of Fortran, OpenMP and CUDA can be helpful
but are not required, as we believe that the GSoC can be an opportunity
to learn new things.

### Work package 1: parallelizing the FMM implementation

**Duration:** 100 hours

**Difficulty:** easy

**Context:**
The library requires to compute electrostatic properties from all the
atoms to all the atoms. This operation is in principle quadratically
scaling, however it is possible to use the **fast multipole method**
to compute electrostatic properties in a linear scaling time.

In our library, we have an in house implementation of the fast multipole
method particularly optimized to handle spherical harmonics. Our
implementation however is not yet parallelized to take advantage of
multiple threads.

**Goal:** Restructuring some FMM operations in such a way that
concurrent writes are avoided, and then using **OpenMP directives** to
parallelize the for loops.

### Work package 2: improving the adjoint linear system

**Duration:** 100 hours

**Difficulty:** easy

**Context:** For some operations, which go beyond the most basic usage
of ddX, it is necessary to solve the adjoint linear system

    L^T y = g

During the years most of the optimization effort was spent on the
forward linear system, with the outcome that solving the adjoint
linear system requires more time with respect to the forward one despite
involving roughly the same number of operations.

**Goal:** Rewriting the operations required to perform an adjoint matrix
vector product in a different order such that fully benefits from
vectorization, rewriting the operations in such a way that concurrent
writes are avoid and a clean OpenMP parallelization is possible.

### Work package 3: CUDA for ddX

**Duration:** 100 hours

**Difficulty:** medium

**Context:**
The linear systems of the form

    LX = f

are solved using iterative systems which do not require storing the
matrices. This strategy avoids a quadratically scaling complexity in
memory, but on the other hand, requires fast code which is able to
perform matrix-vector products several times.

The code which performs matrix-vector products is easily parallelizable
as it requires to perform independent operations on different atoms at
the same time. At the moment we have parallelized this code using
OpenMP with good results. However, a parallelization using GPUs has
never been tried.

**Goal:** Writing a module for the ddX library which uses CUDA to
accelerate ddCOSMO. This is achieved through the following steps:
  - identifying the critical arrays which should be stored in the GPU,
  - writing a CUDA kernel code for atom-atom interaction required by the
  ddCOSMO matrix-vector product,
  - writing a CUDA kernel code for the Jacobi/DIIS linear system solver.

[^1]: Arieh Warshel. “Computer Simulations of Enzyme Catalysis: Methods, Progress, and Insights.” Annual Review of Biophysics and Biomolecular Structure 32, no. 1 (June 2003): 425–43. https://doi.org/10.1146/annurev.biophys.32.110601.141807.

[^2]: Jacopo Tomasi, Benedetta Mennucci, and Roberto Cammi. “Quantum Mechanical Continuum Solvation Models.” Chemical Reviews 105, no. 8 (August 2005): 2999–3094. https://doi.org/10.1021/cr9904009.

[^3]: Eric Cancès, Yvon Maday, and Benjamin Stamm. “Domain Decomposition for Implicit Solvation Models.” The Journal of Chemical Physics 139, no. 5 (August 7, 2013): 054111. https://doi.org/10.1063/1.4816767.

