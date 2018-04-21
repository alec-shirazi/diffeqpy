# diffeqpy

[![Build Status](https://travis-ci.org/JuliaDiffEq/diffeqpy.svg?branch=master)](https://travis-ci.org/JuliaDiffEq/diffeqpy)
[![Build status](https://ci.appveyor.com/api/projects/status/4ybwcrgopv6wrno1?svg=true)](https://ci.appveyor.com/project/ChrisRackauckas/diffeqpy)

diffeqpy is a package for solving differential equations in Python. It utilizes
[DifferentialEquations.jl](http://docs.juliadiffeq.org/latest/) for its core routines
to give high performance solving of many different types of differential equations,
including

- Discrete equations (function maps, discrete stochastic (Gillespie/Markov)
  simulations)
- Ordinary differential equations (ODEs)
- Split and Partitioned ODEs (Symplectic integrators, IMEX Methods)
- Stochastic ordinary differential equations (SODEs or SDEs)
- Random differential equations (RODEs or RDEs)
- Differential algebraic equations (DAEs)
- Delay differential equations (DDEs)
- Mixed discrete and continuous equations (Hybrid Equations, Jump Diffusions)

directly in Python.

## Installation

To install diffeqpy, use pip:

```
pip install diffeqpy
```

Using diffeqpy requires that Julia is installed and in the path, along with
DiffEqPy.jl. To install Julia, download a generic binary from
[the JuliaLang site](https://julialang.org/downloads/) and add it to your path.
Then open up Julia and run the commands:

```julia
Pkg.add("DiffEqPy")
```

and you're good! In addition, to improve the performance of your code it is
recommended that you use Numba to JIT compile your derivative functions. To
install Numba, use:

```
pip install numba
```

## General Flow

Import and setup the solvers via the commands:

```py
import diffeqpy
de = diffeqpy.setup()
```

The general flow for using the package is to follow exactly as would be done
in Julia, except add `de.` in front and use `pysolve` instead of `solve`.
Most of the commands will work without any modification. Thus
[the DifferentialEquations.jl documentation](https://github.com/JuliaDiffEq/DifferentialEquations.jl)
and the [DiffEqTutorials](https://github.com/JuliaDiffEq/DiffEqTutorials.jl)
are the main in-depth documentation for this package. Below we will show how to
translate these docs to Python code.

## Ordinary Differential Equation (ODE) Examples

### One-dimensional ODEs

```py
import diffeqpy
de = diffeqpy.setup()

def f(u,p,t):
    return -u

u0 = 0.5
tspan = (0., 1.)
prob = de.ODEProblem(f, u0, tspan)
sol = de.pysolve(prob)
```

The solution object is the same as the one described
[in the DiffEq tutorials](http://docs.juliadiffeq.org/latest/tutorials/ode_example.html#Step-3:-Analyzing-the-Solution-1)
and in the [solution handling documentation](http://docs.juliadiffeq.org/latest/basics/solution.html)
(note: the array interface is missing). Thus for example the solution time points
are saved in `sol.t` and the solution values are saved in `sol.u`. Additionally,
the interpolation `sol(t)` gives a continuous solution.

We can plot the solution values using matplotlib:

```py
import matplotlib.pyplot as plt
plt.plot(sol.t,sol.u)
plt.show()
```

![f1](https://user-images.githubusercontent.com/1814174/39089385-e898e116-457a-11e8-84c6-fb1f4dd82c48.png)

We can utilize the interpolation to get a finer solution:

```py
import numpy
numpy.linspace(0,1,100)
u = sol(t)
plt.plot(t,u)
plt.show()
```

![f2](https://user-images.githubusercontent.com/1814174/39089386-e8affac2-457a-11e8-8c35-d0f039803ff8.png)

### Solve commands

The [common interface arguments](http://docs.juliadiffeq.org/latest/basics/common_solver_opts.html)
can be used to control the solve command. For example, let's use `saveat` to
save the solution at every `t=0.1`, and let's utilize the `Vern9()` 9th order
Runge-Kutta method along with low tolerances `abstol=reltol=1e-10`:

```py
sol = de.pysolve(prob,de.Vern9(),saveat=0.1,abstol=1e-10,reltol=1e-10)
```

The set of algorithms for ODEs is described
[at the ODE solvers page](http://docs.juliadiffeq.org/latest/solvers/ode_solve.html).

### Compilation with Numba

When solving a differential equation, it's pertinent that your derivative
function `f` is fast since it occurs in the inner loop of the solver. We can
utilize Numba to JIT compile our derivative functions to improve the efficiency
of the solver:

```py
import numba
numba_f = numba.jit(f)

prob = de.ODEProblem(numba_f, u0, tspan)
sol = de.pysolve(prob)
```

### Systems of ODEs: Lorenz Equations

To solve systems of ODEs, simply use an array as your initial condition and
define `f` as an array function:

```py
def f(u,p,t):
    x, y, z = u
    sigma, rho, beta = p
    return [sigma * (y - x), x * (rho - z) - y, x * y - beta * z]

u0 = [1.0,0.0,0.0]
tspan = (0., 100.)
p = [10.0,28.0,8/3]
prob = de.ODEProblem(f, u0, tspan, p)
sol = de.pysolve(prob,saveat=0.01)

plt.plot(sol.t,sol.u)
plt.show()
```

![f3](https://user-images.githubusercontent.com/1814174/39089387-e8c5d9d2-457a-11e8-8f77-eecfc955ce27.png)

or we can draw the phase plot:

```py
ut = numpy.transpose(sol.u)
from mpl_toolkits.mplot3d import Axes3D
fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')
ax.plot(ut[0,:],ut[1,:],ut[2,:])
plt.show()
```

![f4](https://user-images.githubusercontent.com/1814174/39089388-e8dae00c-457a-11e8-879f-8b01e0b47178.png)

### In-Place Mutating Form

When dealing with systems of equations, in many cases it's helpful to reduce
memory allocations by using mutating functions. In diffeqpy, the mutating
form adds the mutating vector to the front. Let's make a fast version of the
Lorenz derivative, i.e. mutating and JIT compiled:

```py
def f(du,u,p,t):
    x, y, z = u
    sigma, rho, beta = p
    du[0] = sigma * (y - x)
    du[1] = x * (rho - z) - y
    du[2] = x * y - beta * z

numba_f = numba.jit(f)
u0 = [1.0,0.0,0.0]
tspan = (0., 100.)
p = [10.0,28.0,2.66]
prob = de.ODEProblem(numba_f, u0, tspan, p)
sol = de.pysolve(prob)
```

## Stochastic Differential Equation (SDE) Examples

### One-dimensional SDEs

Solving one-dimensonal SDEs `du = f(u,t)dt + g(u,t)dW_t` is like an ODE except
with an extra function for the diffusion (randomness or noise) term. The steps
follow the [SDE tutorial](http://docs.juliadiffeq.org/latest/tutorials/sde_example.html).

```py
def f(u,p,t):
  return 1.01*u

def g(u,p,t):
  return 0.87*u

u0 = 0.5
tspan = (0.0,1.0)
prob = de.SDEProblem(f,g,u0,tspan)
sol = de.pysolve(prob,reltol=1e-3,abstol=1e-3)

plt.plot(sol.t,sol.u)
plt.show()
```

![f5](https://user-images.githubusercontent.com/1814174/39089389-e8f0343e-457a-11e8-87bb-9ed152caee02.png)

### Systems of SDEs with Diagonal Noise

An SDE with diagonal noise is where a different Wiener process is applied to
every part of the system. This is common for models with phenomenological noise.
Let's add multiplicative noise to the Lorenz equation:

```py
def f(du,u,p,t):
    x, y, z = u
    sigma, rho, beta = p
    du[0] = sigma * (y - x)
    du[1] = x * (rho - z) - y
    du[2] = x * y - beta * z

def g(du,u,p,t):
    du[0] = 0.3*u[0]
    du[1] = 0.3*u[1]
    du[2] = 0.3*u[2]

numba_f = numba.jit(f)
numba_g = numba.jit(g)
u0 = [1.0,0.0,0.0]
tspan = (0., 100.)
p = [10.0,28.0,2.66]
prob = de.SDEProblem(numba_f, numba_g, u0, tspan, p)
sol = de.pysolve(prob)

# Now let's draw a phase plot

ut = numpy.transpose(sol.u)
from mpl_toolkits.mplot3d import Axes3D
fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')
ax.plot(ut[0,:],ut[1,:],ut[2,:])
plt.show()
```

![f6](https://user-images.githubusercontent.com/1814174/39089390-e906c1ea-457a-11e8-8fd2-5cf059e2165a.png)

### Systems of SDEs with Non-Diagonal Noise

In many cases you may want to share noise terms across the system. This is
known as non-diagonal noise. The
[DifferentialEquations.jl SDE Tutorial](http://docs.juliadiffeq.org/latest/tutorials/sde_example.html#Example-4:-Systems-of-SDEs-with-Non-Diagonal-Noise-1)
explains how the matrix form of the diffusion term corresponds to the
summation style of multiple Wiener processes. Essentially, the row corresponds
to which system the term is applied to, and the column is which noise term.
So `du[i,j]` is the amount of noise due to the `j`th Wiener process that's
applied to `u[i]`. We solve the Lorenz system with correlated noise as follows:

```py
import numba
import numpy
import diffeqpy
de = diffeqpy.setup()
def f(du,u,p,t):
  x, y, z = u
  sigma, rho, beta = p
  du[0] = sigma * (y - x)
  du[1] = x * (rho - z) - y
  du[2] = x * y - beta * z

def g(du,u,p,t):
  du[0,0] = 0.3*u[0]
  du[1,0] = 0.6*u[0]
  du[2,0] = 0.2*u[0]
  du[0,1] = 1.2*u[1]
  du[1,1] = 0.2*u[1]
  du[2,1] = 0.3*u[1]


u0 = [1.0,0.0,0.0]
tspan = (0.0,100.0)
p = [10.0,28.0,2.66]
nrp = numpy.zeros((3,2))
numba_f = numba.jit(f)
numba_g = numba.jit(g)
prob = de.SDEProblem(numba_f,numba_g,u0,tspan,p,noise_rate_prototype=nrp)
sol = de.pysolve(prob,saveat=0.005)

# Now let's draw a phase plot

ut = numpy.transpose(sol.u)
from mpl_toolkits.mplot3d import Axes3D
fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')
ax.plot(ut[0,:],ut[1,:],ut[2,:])
plt.show()
```

![f7](https://user-images.githubusercontent.com/1814174/39089391-e91f0494-457a-11e8-860a-865caa26c262.png)

Here you can see that the warping effect of the noise correlations is quite visible!

## Differential-Algebraic Equation (DAE) Examples

```py
def f(du,u,p,t):
  resid1 = - 0.04*u[0]               + 1e4*u[1]*u[2] - du[0]
  resid2 = + 0.04*u[0] - 3e7*u[1]**2 - 1e4*u[1]*u[2] - du[1]
  resid3 = u[0] + u[1] + u[2] - 1.0
  return [resid1,resid2,resid3]

u0 = [1.0, 0, 0]
du0 = [-0.04, 0.04, 0.0]
tspan = (0.0,100000.0)
differential_vars = [True,True,False]
prob = de.DAEProblem(f,du0,u0,tspan,differential_vars=differential_vars)
sol = de.pysolve(prob)
```

![f8](https://user-images.githubusercontent.com/1814174/39089392-e932f012-457a-11e8-9979-c006bcfabf71.png)

and the in-place JIT compiled form:

```py
def f(resid,du,u,p,t):
  resid[0] = - 0.04*u[0]               + 1e4*u[1]*u[2] - du[0]
  resid[1] = + 0.04*u[0] - 3e7*u[1]**2 - 1e4*u[1]*u[2] - du[1]
  resid[2] = u[0] + u[1] + u[2] - 1.0

numba_f = numba.jit(f)
prob = de.DAEProblem(numba_f,du0,u0,tspan,differential_vars=differential_vars)
sol = de.pysolve(prob)
```
