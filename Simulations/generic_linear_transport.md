# Applying Generic Linear Transport Equations

Simulating basic transport processes (diffusion, viscous flow) through the pore network is one of the main applications of OpenPNM.  This tutorial will outline how to estimate the effective transport properties of a cubic network.

## Step 1: Define Core Objects Necessary for the Simulation

### Create Network with Boundary Pores
Start by creating a cubic network with 100 um between pore centers:

``` python
>>> import scipy as sp
>>> import OpenPNM as op
>>> net = op.Network.Cubic(shape=[20, 15, 10], spacing=0.0001, name='net')

```
The spacing was chosen to be slightly different in each direction to aid the demonstration of the domain size calculations below.  Next, we'll add boundary pores to the top and bottom faces of the network:

``` python
>>> net.add_boundaries(labels=['top', 'bottom'])

```

The ``add_boundaries`` method clones all pores labeled ``'top'`` and ``'bottom'``, then shifts them to the nearest face of the Network.  These new boundary pores can be assigned special geometrical properties such as zero volume or zero length so they don't impact the transport behavior of the actual network.  Let's create 2 *Geometry* objects to handle the ``'internal'`` and ``'boundary'`` pores separately, using pre-defined *Geometry* classes:

### Create Geometry Objects Using Pre-Defined Classes

``` python
>>> Ps = net.pores('internal')
>>> Ts = net.throats('all')
>>> geom_internal = op.Geometry.Stick_and_Ball(network=net, pores=Ps, throats=Ts)
>>> Ps = net.pores('*boundary')  # Use wildcard to find all boundary pores
>>> geom_boundary = op.Geometry.Boundary(network=net, pores=Ps)

```

The pre-defined *STick_and_Ball* class contains pore-scale models that assign random pore diameters to each pore without any overlap, then assigns throat diameters equal to the larger of the two neighboring pores.  All the subsequent calculations uses the geometry of spheres and cylinders.  The *Boundary* class assigns pore and throat sizes to assure no transport resistance, such as zero volume pores, zero length throats, infinite cross-sectional area, etc.

### Create Phase Objects for Thermophysical Properties

In this tutorial we'll demonstrate diffusion of oxygen in air, viscous flow of water, and thermal conduction through the solid phase, so we need to define a Phase object for each:

``` python
>>> gas = op.Phases.Air(network=net, name='air')
>>> liquid = op.Phases.Water(network=net, name='water')
>>> solid = op.Phases.GenericPhase(network=net, name='matrix')
>>> solid['pore.electrical_conductivity'] = 100  # S/m
```

The ``gas`` and ``liquid`` objects used the pre-defined *Air* and *Water* classes which include all the necessary thermophysical properties and models.  The ``solid`` object used the *GenericPhase* class which includes no properties, so they must be assigned or calculated.

### Create Physics Objects and Define Pore-Scale Models

The final setup step is to define the pore-scale physics models.  Each *Phase* requires its own pore-scale physics to be defined, so we must create a *Physics* object for each.  Technically it is also possible to define a unique *Physics* object for each *Geometry*.  This possibility was to allow for geometrically distinct regions to also have distinct physics (i.e. bulk vs Knudsen diffusion).  In the present case this is not necessary so one *Physics* per *Phase* will suffice:

``` python
>>> phys_gas = op.Physics.GenericPhysics(network=net, phase=gas, pores=net.Ps, throats=net.Ts)
>>> phys_liq = op.Physics.GenericPhysics(network=net, phase=liquid, pores=net.Ps, throats=net.Ts)
>>> phys_sol = op.Physics.GenericPhysics(network=net, phase=solid, pores=net.Ps, throats=net.Ts)

```
The above definitions of Phase objects mean that each will look into their respective phases for thermophysical properties, and will get geometrical properties from the entire Network, not just a limited subset defined by one of the Geometries.

These Physics objects are all *GenericPhysics* which means that no pore-scale models have been selected yet.  The simulations to be run dictate which pore-scale physics to assign.  Each sub-module in OpenPNM contains a library of common models that can be used as follows:

``` python
>>> mod = op.Physics.models.diffusive_conductance.bulk_diffusion
>>> phys_gas.models.add(propname='throat.diffusive_conductance', model=mod)
>>> mod = op.Physics.models.hydraulic_conductance.hagen_poiseuille
>>> phys_liq.models.add(propname='throat.hydraulic_conductance', model=mod)
>>> mod = op.Physics.models.electrical_conductance.series_resistors
>>> phys_sol.models.add(propname='throat.electrical_conductance', model=mod)

```

These models all require certain geometrical parameters such as throat length, cross-sectional area, and so forth, which were defined by the *Geometry* objects above.  They also require specific thermophysical properties which were defined by the *Phase* objects above.  

## Step 2: Run Various Transport simulations

### Setup Fickian diffusion

OpenPNM has a *GenericLinearTransport* class that possesses the main functionality for setting up basic Laplacian transport. This class manages the addition of boundary conditions, building the coefficient matrix, and calling the Scipy matrix inversion functions.  Several sub-classes have been created for each typical transport processes, which basically just have default values set for the various argument names (i.e. ``conductance`` in *FickianDiffusion* defaults to 'throat.diffusive_conductance').  
