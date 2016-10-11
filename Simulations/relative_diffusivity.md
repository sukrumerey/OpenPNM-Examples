# Relative Diffusivity

## Generating the Network, adding Geometry and creating Phases

This example shows you how to calculate a transport property relative to the saturation of the domain by a particular phase. In this case the property is the diffusivity of air relative to the saturation of water. Start by importing OpenPNM and some other useful packages:

``` python
>>> import OpenPNM
>>> import numpy as np
>>> import matplotlib.pyplot as plt
>>> workspace = OpenPNM.Base.Workspace()
>>> workspace.clear()workspace
>>> workspace.loglevel = 30 # Adjust log level to suppress messages

```
Next create a **Network** object with a cubic topology and lattice spacing of 25 microns and add boundary pores

``` python
>>> pn = OpenPNM.Network.Cubic(shape=[10, 10, 10], spacing=4.0e-5)
>>> pn.add_boundaries()

```

Next create a **Geometry** to manage the pore and throat size information.  A **Geometry** can span over a part of the **Network** only, so we need to specify to which pores and throats this **Geometry** object should apply. For this example, we want one **Geometry** to apply to the internal pores and a different one to the boundary pores:

``` python
>>> Ps = pn.pores('boundary', mode='not')
>>> Ts = pn.find_neighbor_throats(pores=Ps,mode='intersection',flatten=True)
>>> geom = OpenPNM.Geometry.Toray090(network=pn, pores=Ps, throats=Ts)
>>> Ps = pn.pores('boundary')
>>> Ts = pn.find_neighbor_throats(pores=Ps,mode='not_intersection')
>>> boun = OpenPNM.Geometry.Boundary(network=pn, pores=Ps, throats=Ts)

```

The ``Toray090`` **Geometry** is a predefined class that applies Weibull distributed pore and throat sizes to the internal pores and is named after the Toray carbon paper gas diffusion layer commonly used in fuel cells.  The ``Boundary`` class is predefined with properties suitable for boundaries such as 0 volume and length.

We must also create the **Phase** objects, for our purposes the standard ``air`` and ``water`` phase classes provided are fine:

``` python
>>> air = OpenPNM.Phases.Air(network=pn, name='air')
>>> water = OpenPNM.Phases.Water(network=pn, name='water')

```

## Define the Pore-Scale Physics

For this simulation the standard physics object can be used as it contains capillary pressure for use in the ``OrdinaryPercolation`` algorithm and diffusive conductance for use in the ``FickianDiffusion`` algorithm.

``` python
>>> phys_air = OpenPNM.Physics.Standard(network=pn, phase=air, pores=pn.Ps,
...                                     throats=pn.Ts)
>>> phys_water = OpenPNM.Physics.Standard(network=pn, phase=water, pores=pn.Ps,
...                                       throats=pn.Ts)

```

## Set up and run the Ordinary Percolation Algorithm
In order to simulate a partially saturated material we first run an ``OrdinaryPercolation`` **Algorithm** which sequentially invades the network with the ``invading_phase`` based on the capillary pressure of the throats in the network. This allows us to inspect which pores and throats are occupied by which phase at various capillary pressures and this occupancy is used to calculate the multiphase diffusive conductance.

``` python
>>> bounds = [['front', 'back'], ['left', 'right'], ['top', 'bottom']]
>>> npts=101
>>> inv_points = np.linspace(0,10000,npts)
>>> OP_1 = OpenPNM.Algorithms.OrdinaryPercolation(network=pn,
...                                               invading_phase=water,
...                                               defending_phase=air)
>>> inlets = pn.pores('bottom_boundary')
>>> step = 2
>>> used_inlets = [inlets[x] for x in range(0, len(inlets), step)]
>>> OP_1.run(inlets=used_inlets, inv_points=inv_points)
>>> OP_1.return_results()

```

Here we have selected half of the boundary pores at the bottom of the domain as inlets for the percolation **Algorithm**. ``OrdinaryPercolation`` has a helpful plotting function which displays the saturation of the invading phase (volume fraction of the pore space) vs. capillary pressure: ```OP_1.plot_drainage_curve()```. The red line is pore saturation, blue is throat saturation and green is total saturation.

![](http://imgur.com/o6zfY8p.png)

> **Note** :  OpenPNM's Postprocessing module also has the ability to plot drainage curves and is suitable for use with the ``InvasionPercolation`` algorithm too.

## Run a Fickian Diffusion Algorithm for each step of the invasion process

We now need to model how the presence of the phases affects the diffusive conductivity of the network. Currently the **Physics** objects have a property called ``throat.diffusive_conductance`` but this model does not account for the occupancy of each phase and assumes that the phase occupies every pore-throat-pore conduit. OpenPNM has a number of multiphase models including a conduit conductance that multiplies the single phase conductance by a factor (default 0.000001) when the phase associated with the physics object is not present. The model has a mode which defaults to 'strict' which applies the conductivity reduction if any one of the connected pores or connecting throat is unoccupied.

``` python
>>> import OpenPNM.Physics.models as pm
>>> phys_air.add_model(model=pm.multiphase.conduit_conductance,
...                    propname='throat.conduit_diffusive_conductance',
...                    throat_conductance='throat.diffusive_conductance')
>>> phys_water.add_model(model=pm.multiphase.conduit_conductance,
...                      propname='throat.conduit_diffusive_conductance',
...                      throat_conductance='throat.diffusive_conductance')

```    

Now we create some variables to store our data in for each principle direction (x, y, z). The boundary planes at each side of the domain are used as boundary pores for the Diffusion algorithm.

``` python
>>> bounds = [['front', 'back'], ['left', 'right'], ['top', 'bottom']]
>>> diff_air = {'0': [], '1': [], '2': []}
>>> diff_water = {'0': [], '1': [], '2': []}
>>> sat=[]
>>> tot_vol = np.sum(pn["pore.volume"]) + np.sum(pn["throat.volume"])

```

Now for each invasion step we cycle through the principle directions and create ``FickianDiffusion`` objects for each phase and calculate the effective diffusivity.


``` python
>>> for Pc in inv_points:
...     OP_1.return_results(Pc=Pc)
...     phys_air.regenerate()
...     phys_water.regenerate()
...     this_sat = 0
...     this_sat += np.sum(pn["pore.volume"][water["pore.occupancy"] == 1])
...     this_sat += np.sum(pn["throat.volume"][water["throat.occupancy"] == 1])
...     sat.append(this_sat)
...     print("Capillary Pressure: "+str(Pc)+", Saturation: "+str(this_sat/tot_vol))
...     for bound_increment in range(len(bounds)):
...         BC1_pores = pn.pores(labels=bounds[bound_increment][0]+'_boundary')
...         BC2_pores = pn.pores(labels=bounds[bound_increment][1]+'_boundary')
...         FD_1 = OpenPNM.Algorithms.FickianDiffusion(network=pn, phase=air)
...         FD_1.set_boundary_conditions(bctype='Dirichlet', bcvalue=0.6,
...                                      pores=BC1_pores)
...         FD_1.set_boundary_conditions(bctype='Dirichlet', bcvalue=0.2,
...                                      pores=BC2_pores)
...         FD_1.run(conductance='conduit_diffusive_conductance')
...         eff_diff = FD_1.calc_eff_diffusivity()
...         diff_air[str(bound_increment)].append(eff_diff)
...         FD_2 = OpenPNM.Algorithms.FickianDiffusion(network=pn, phase=water)
...         FD_2.set_boundary_conditions(bctype='Dirichlet', bcvalue=0.6,
...                                      pores=BC1_pores)
...         FD_2.set_boundary_conditions(bctype='Dirichlet', bcvalue=0.2,
...                                      pores=BC2_pores)
...         FD_2.run(conductance='conduit_diffusive_conductance')
...         eff_diff = FD_2.calc_eff_diffusivity()
...         diff_water[str(bound_increment)].append(eff_diff)
...         workspace.purge_object(FD_1)
...         workspace.purge_object(FD_2)

```

The ```return_results``` method updates the two **Phase** objects with the occupancy at the given capillary pressure (Pc). The **Physics** objects are then regenerated to re-calculate the ```conduit_diffusive_conductance``` property.

> **Note** :  Six Diffusion algorithm objects could have been created outside the loop and then run over and over with the updated conductance values and this would possibly save some computational time but is not as nice to code. Instead the objects are purged and redefined with updated boundary pores inside another for loop.

## Plot the Relative Diffusivity Curves for each direction and Phase

Now tidy up the data converting them into Numpy arrays for easy plotting and manipulation and normalize the results by the single phase values:

``` python
>>> sat=np.asarray(sat)
>>> sat /= tot_vol
>>> rel_diff_air_x=np.asarray(diff_air['0'])
>>> rel_diff_air_x /= rel_diff_air_x[0]
>>> rel_diff_air_y=np.asarray(diff_air['1'])
>>> rel_diff_air_y /= rel_diff_air_y[0]
>>> rel_diff_air_z=np.asarray(diff_air['2'])
>>> rel_diff_air_z /= rel_diff_air_z[0]
>>> rel_diff_water_x=np.asarray(diff_water['0'])
>>> rel_diff_water_x /= rel_diff_water_x[-1]
>>> rel_diff_water_y=np.asarray(diff_water['1'])
>>> rel_diff_water_y /= rel_diff_water_y[-1]
>>> rel_diff_water_z=np.asarray(diff_water['2'])
>>> rel_diff_water_z /= rel_diff_water_z[-1]

```

Finally plot the results:

``` python
>>> fig = plt.figure()
>>> ax = fig.gca()
>>> plots=[]
>>> plots.append(plt.plot(sat,rel_diff_air_x,'^-r',label='Dra_x'))
>>> plots.append(plt.plot(sat,rel_diff_air_y,'^-g',label='Dra_y'))
>>> plots.append(plt.plot(sat,rel_diff_air_z,'^-b',label='Dra_z'))
>>> plots.append(plt.plot(sat,rel_diff_water_x,'*-r',label='Drw_x'))
>>> plots.append(plt.plot(sat,rel_diff_water_y,'*-g',label='Drw_y'))
>>> plots.append(plt.plot(sat,rel_diff_water_z,'*-b',label='Drw_z'))
>>> plt.xlabel('Liquid Water Saturation')
>>> plt.ylabel('Relative Diffusivity')
>>> handles, labels = ax.get_legend_handles_labels()
>>> ax.legend(handles, labels,loc=1)
>>> box = ax.get_position()
>>> ax.set_position([box.x0, box.y0, box.width * 0.9, box.height])
>>> ax.legend(loc='center left', bbox_to_anchor=(1, 0.5))

```
![](http://imgur.com/hEGXVyp.png)
