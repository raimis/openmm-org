---
layout: nicepage
title: Alchemical free energy calculations
lead: Computing the free energy of inserting a Lennard-Jones particle in a Lennard-Jones fluid
---

## Files

The code for this example can be downloaded here:
* [`alchemical-example.py`](files/alchemical-example.py)

## Basic concept

OpenMM's custom forces---which allow the programmer to express a potential algebraically, potentially with multiple parameters that can be adjusted on the fly---allow a great deal of flexibility and simplicity in encoding potentials while still achieving high performance on GPUs.
One common use of this facility is to convert standard interactions (such as Lennard-Jones potentials) into alchemically-modified potentials for the purposes of computing free energy differences.
The alchemical free energy code [YANK](http://github.com/choderalab/yank), for example, uses a variety of custom forces to represent alchemically-modified potentials for [protein-ligand alchemical binding free energy calculations](http://dx.doi.org/10.1007/s10822-013-9689-8).

## Defining alchemically-modified potentials

As a simple example of how this is facilitated by custom forces, consider computing the chemical potential of liquid argon by estimating the free energy of alchemically annihilating a Lennard-Jones particle.
First, we create a simple Lennard-Jones fluid to represent liquid argon at 120 K and 80 atm, which can be conveniently done using the `testsystems` module of the conda-installable [`openmmtools`](http://github.com/choderalab/openmmtools) package:
```python
from simtk import openmm, unit

# Create a Lennard-Jones fluid
pressure = 80*unit.atmospheres
temperature = 120*unit.kelvin
collision_rate = 5/unit.picoseconds
timestep = 2.5*unit.femtoseconds
from openmmtools.testsystems import LennardJonesFluid
sigma = 3.4*unit.angstrom; epsilon = 0.238 * unit.kilocalories_per_mole
fluid = LennardJonesFluid(sigma=sigma, epsilon=epsilon)
[system, positions] = [fluid.system, fluid.positions]

# Add a barostat
barostat = openmm.MonteCarloBarostat(pressure, temperature)
system.addForce(barostat)
```
To allow one of the Lennard-Jones particles to be alchemically eliminated, we create a [`CustomNonbondedForce`](http://docs.openmm.org/7.1.0/api-python/generated/simtk.openmm.openmm.CustomNonbondedForce.html#simtk.openmm.openmm.CustomNonbondedForce) that will compute the interactions between the alchemical particle and the remaining chemical particles using a [softcore potential](http://dx.doi.org/10.1016/0009-2614(94)00397-1).
The alchemically-modified particle has its Lennard-Jones well depth (`epsilon` parameter) set to zero in the original `NonbondedForce`, while the `CustomNonbondedForce` is set to evaluate only the interactions between the alchemically-modified particle and the remaining particles using [`addInteractionGroup()`](http://docs.openmm.org/7.1.0/api-python/generated/simtk.openmm.openmm.CustomNonbondedForce.html#simtk.openmm.openmm.CustomNonbondedForce.addInteractionGroup) to specify only interactions between these groups are to be computed.
A global context parameter `lambda` is created to control the coupling of the alchemically-modified particle with the rest of the system during the simulation.
The Lennard-Jones parameters `sigma` and `epsilon` are implemented as per-particle parameters, though this is not strictly necessary in this case since all particles are equivalent.
```python
# Retrieve the NonbondedForce
forces = { force.__class__.__name__ : force for force in system.getForces() }
nbforce = forces['NonbondedForce']

# Add a CustomNonbondedForce to handle only alchemically-modified interactions
alchemical_particles = set([0])
chemical_particles = set(range(system.getNumParticles())) - alchemical_particles
energy_function = 'lambda*4*epsilon*x*(x-1.0); x = (sigma/reff_sterics)^6;'
energy_function += 'reff_sterics = sigma*(0.5*(1.0-lambda) + (r/sigma)^6)^(1/6);'
energy_function += 'sigma = 0.5*(sigma1+sigma2); epsilon = sqrt(epsilon1*epsilon2);'
custom_force = openmm.CustomNonbondedForce(energy_function)
custom_force.addGlobalParameter('lambda', 1.0)
custom_force.addPerParticleParameter('sigma')
custom_force.addPerParticleParameter('epsilon')
for index in range(system.getNumParticles()):
    [charge, sigma, epsilon] = nbforce.getParticleParameters(index)
    custom_force.addParticle([sigma, epsilon])
    if index in alchemical_particles:
        nbforce.setParticleParameters(index, charge*0, sigma, epsilon*0)
custom_force.addInteractionGroup(alchemical_particles, chemical_particles)
system.addForce(custom_force)
```

## Simulating alchemically-modified systems

We then create a [`LangevinIntegrator`](http://docs.openmm.org/7.1.0/api-python/generated/simtk.openmm.openmm.LangevinIntegrator.html#simtk.openmm.openmm.LangevinIntegrator) and [`Context`](http://docs.openmm.org/7.1.0/api-python/generated/simtk.openmm.openmm.Context.html#simtk.openmm.openmm.Context) to run the simulation, and run a series of simulations at different values of `lambda` by using [`context.setParameter()`](http://docs.openmm.org/7.1.0/api-python/generated/simtk.openmm.openmm.Context.html#simtk.openmm.openmm.Context.setParameter) to update the alchemical parameter on the fly.
For each configuration sample that is collected, we can easily scan through the energy at different `lambda` values by simply alternating between `context.setParameter()` to update `lambda` and `context.getState()` to retrieve potential energies at the new alchemical state.
```python
# Create a context
integrator = openmm.LangevinIntegrator(temperature, collision_rate, timestep)
context = openmm.Context(system, integrator)
context.setPositions(positions)

# Minimize energy
print('Minimizing energy...')
openmm.LocalEnergyMinimizer.minimize(context)

# Collect data
nsteps = 2500 # number of steps per sample
niterations = 50 # number of samples to collect per alchemical state
import numpy as np
lambdas = np.linspace(1.0, 0.0, 10) # alchemical lambda schedule
nstates = len(lambdas)
u_kln = np.zeros([nstates,nstates,niterations], np.float64)
kT = unit.AVOGADRO_CONSTANT_NA * unit.BOLTZMANN_CONSTANT_kB * integrator.getTemperature()
for k in range(nstates):
    for iteration in range(niterations):
        print('state %5d iteration %5d / %5d' % (k, iteration, niterations))
        # Set alchemical state
        context.setParameter('lambda', lambdas[k])
        # Run some dynamics
        integrator.step(nsteps)
        # Compute energies at all alchemical states
        for l in range(nstates):
            context.setParameter('lambda', lambdas[l])
            u_kln[k,l,iteration] = context.getState(getEnergy=True).getPotentialEnergy() / kT

```

## Analyzing the data with MBAR

Finally, the [multistate Bennett acceptance ratio (MBAR)](https://dx.doi.org/10.1063%2F1.2978177) is used to estimate the free energy of annihilating the particle using the conda-installable [`pymbar`](http://pymbar.org/) package.
In order to estimate how much data must be discarded to equilibration, we use a scheme for [automated equilibration detection](http://dx.doi.org/10.1021/acs.jctc.5b00784) and subsequent extraction of decorrelated samples found in the [`pymbar.timeseries`](http://github.com/choderalab/pymbar) module.
```python
# Estimate free energy of Lennard-Jones particle insertion
from pymbar import MBAR, timeseries
# Subsample data to extract uncorrelated equilibrium timeseries
N_k = np.zeros([nstates], np.int32) # number of uncorrelated samples
for k in range(nstates):
    [nequil, g, Neff_max] = timeseries.detectEquilibration(u_kln[k,k,:])
    indices = timeseries.subsampleCorrelatedData(u_kln[k,k,:], g=g)
    N_k[k] = len(indices)
    u_kln[k,:,0:N_k[k]] = u_kln[k,:,indices].T
# Compute free energy differences and statistical uncertainties
mbar = MBAR(u_kln, N_k)
[DeltaF_ij, dDeltaF_ij, Theta_ij] = mbar.getFreeEnergyDifferences()
```

## Additional resources

* Full-featured factories for re-encoding standard OpenMM forces as alchemically-modified custom forces is available through the [`openmmtools.alchemy` module](http://openmmtools.readthedocs.io/en/latest/alchemy.html) of the [`openmmtools` package](https://github.com/choderalab/openmmtools).
* The [YANK](http://getyank.org) package from the [Chodera lab](http://choderalab.org) provides a full-featured package for absolute alchemical free energy calculations for small molecule binding and transfer free energies.
* The [Sire](http://siremol.org/) package from the [Michel research group](http://www.julienmichel.net/) provides a full-featured package for relative alchemical free energy calculations using reaction-field for small molecule binding and transfer free energies.
