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

## Step 2: Run Various Transport Simulations

### Simulate Fickian Diffusion

OpenPNM has a *GenericLinearTransport* class that possesses the main functionality for setting up basic Laplacian transport. This class manages the addition of boundary conditions, building the coefficient matrix, and calling the Scipy matrix inversion functions.  Several sub-classes have been created for each typical transport processes, which basically just have default values set for the various argument names (i.e. ``conductance`` in *FickianDiffusion* defaults to 'throat.diffusive_conductance').  

``` python
>>> fickian = op.Algorithms.FickianDiffusion(network=net, phase=gas)
>>> fickian.set_boundary_conditions(pores=net.pores('top_boundary'),
...                                 bctype='Dirichlet', bcvalue=0.5)
>>> fickian.set_boundary_conditions(pores=net.pores('bottom_boundary'),
...                                 bctype='Dirichlet', bcvalue=0.0)
>>> fickian.setup(conductance='diffusive_conductance', quantity='mole_fraction')

```

The ``setup`` method was called here to demonstrate the ``conductance`` and ``quantity`` arguments.  These values are pre-defined for *FickianDiffusion* but could be customized by users if desired.  For instance, if 'throat.gdg' were used as the ``propname`` when assigning the ``bulk_diffusion`` model to the ``phys_gas``, then this should be explicitly provided in ``setup``.  Also, the result of the *FickianDiffusion* algorithm is to calculate the ``mole_fraction`` in each pore, but this could explicitly set to ``oxygen_conc`` if desired.  

The ``solve`` command is called, which performed the matrix inversion using Scipy's built-in direct solver (See note below), then solves for the unknown *quantity* in each pore.  The result is stored on the *Algorithm* object in the array 'pore.mole_fraction':

``` python
>>> fickian.solve()
>>> 'pore.mole_fraction' in fickian.keys()
True
>>> fickian.return_results()
```

The last line copies the values of 'pore.mole_fraction' from the ``fickian`` object to the ``gas`` object.  This step is done manually to prevent any unwanted overwriting of values on the ``gas`` object.  
> **NOTE**: The default solver in Scipy is SuperLU, which is compatible with the Scipy license but does not offer the best performance. If (scikit-umfpack)[https://pypi.python.org/pypi/scikit-umfpack] is installed Scipy will automatically use it, and *much* better performance will result.  Umfpack is still a direct solver however, so will run into memory issues for large simulations (>200,000 pores).  Scipy offers a range of iterative solvers which can be used by OpenPNM.  This is demonstrated in the final example below.

The concentration gradient can be visualized by exporting the current network data to a *'vtp'* file using ``op.export_data()`` and processing it in Paraview

![](https://i.imgur.com/gJZvTt8m.png)

Although the concentration profile within the network is of some interest, it is usually the flux through the network that is desired.  From this it's possible to calculate the effective diffusivity.  The **GenericLinearTransport** class has a ``rate`` method that calculates the cumulative rate entering or exiting a group of pores.  To determine the flux across the network we can query the rate into the 'top_boundary':

``` python
>>> flux_in = fickian.rate(pores=net.pores('top_boundary'))
>>> flux_out = fickian.rate(pores=net.pores('bottom_boundary'))

```

You can see that ``flux_in`` and ``flux_out`` are equal with opposite signs.  OpenPNM considers mass leaving a group of pores a positive (i.e. production of mass by those pores), and mass entering as negative (i.e. consumption of mass by those pores).  This flux can be used to determine the effective diffusivity of the network since the domain size and boundary conditions are known:

``` python
>>> L = 10 * 0.0001  # Network length in z-direction
>>> A = 20 * 15 * (0.0001)**2  # Network area in x-y direction
>>> c = sp.mean(gas['pore.molar_density'])  # mole/m3 of gas in air
>>> delta_x = 0.5  # Difference between inlet and outlet boundary conditions
>>> Deff = flux_out*L/(A*c)/delta_x  # Solve for effective diffusivity
>>> sp.all(Deff < gas['pore.diffusivity'])  # Ensure Deff is less than open air value

```

### Simulate Stokes flow

The *StokesFlow* class is another subclass of *GenericLinearTransport* suitable for simulating Darcy flow a network.  The following example will mirror that for Fickian diffusion above, but with slightly different boundary conditions.  

``` python
>>> stokes = op.Algorithms.StokesFlow(network=net, phase=liquid)
>>> stokes.set_boundary_conditions(pores=net.pores('top_boundary'),
...                                bctype='Neumann_group', bcvalue=-1e-8)
>>> stokes.set_boundary_conditions(pores=net.pores('bottom_boundary'),
...                                bctype='Dirichlet', bcvalue=101325)
>>> stokes.run()
>>>

```

Note that a *Neumann* boundary condition was specified into the top face, while the outlet was set to atmospheric pressure.   Also, note that instead of calling ``setup`` and ``solve``, the ``run`` method was called.  This is a short-cut for situations where no custom arguments are used and it simply calls ``setup`` and ``solve`` in sequence.  

It can be confirmed that indeed the *rate* of material entering the top face is equal to the specified boundary value:

``` python
>>> flux_in = stokes.rate(pores=net.pores('top_boundary'))
>>> flux_in == -1e-8
True

```

It's also possible to inspect the inlet pressure require to accomplish the specified flow:

``` python
>>> p_in = stokes['pore.pressure'][net.pores('top_boundary')]
>>> sp.mean(p_in) > 101325  # Inlet pressure greater than outlet

```

Finally, the permeability coefficient can be found by invoking Darcy's law:

``` python
>>> L = 10 * 0.0001  # Network length in z-direction
>>> A = 20 * 15 * (0.0001)**2  # Network area in x-y direction
>>> mu = sp.mean(liquid['pore.viscosity'])  # Get average water viscosity
>>> delta_p = sp.mean(p_in) - 101325 # Difference between inlet and outlet pressures
>>> K = flux_out*L*mu/(A)/delta_p  # Solve for permeability coefficient

```

### Simulate Ohmic Conduction


















end
