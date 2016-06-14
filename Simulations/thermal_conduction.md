# Thermal Conduction

This examples shows 	how OpenPNM can be used to simulate thermal conduction on a generic domain. The result obtained from OpenPNM is compared to the analytical result. 

## Importing OpenPNM

As usual, start by importing OpenPNM, and the SciPy library.

``` python
>>> import scipy as sp
>>> import OpenPNM

```

## Generating the Network object

Next, a **Network** is generated with dimensions of 10x50 elements. The lattice spacing is given by *Lc*. Boundaries are added all around the **Network** object using the *add_boundaries()* method, and the boundary pores on top and bottom of the domain are removed using the *trim()* method. This is done to restrict the problem here to a two dimensional case only, and the top and bottom boundary pores would render this a three dimensional problem. Of course OpenPNM is not limited to two dimensional cases, this is only done in this example for easier explanation.

``` python
>>> # Generate Network and clean up boundaries (delete z-face pores)
>>> divs = [10, 50]
>>> Lc = 0.1  # cm
>>> pn = OpenPNM.Network.Cubic(shape=divs, spacing=Lc)
>>> pn.add_boundaries()
>>> pn.trim(pores=pn.pores(['top_boundary', 'bottom_boundary']))

```

## Generating the Geometry object

After the **Network** is generated, a **Geometry** object is defined for the internal and the boundary pores. Therefore first all relevant pores (either boundary or internal) and throats are obtained from the network object *pn* and assigned to the variables *Ps* and *Ts*, to be used as the pores and throats associated with the **Geometry** object that is generated in the next line.

``` python
>>> # Generate Geometry objects for internal and boundary pores
>>> Ps = pn.pores('internal')
>>> Ts = pn.throats()
>>> geom = OpenPNM.Geometry.GenericGeometry(network=pn,
...                                         pores=Ps,
...                                         throats=Ts)

```

The shape of each individual element, so the pores and throats, is altered in the following section to obtain pores of cubic shape with the diameter equal to the lattice spacing and the pore area equal to the square of the lattice spacing. The throats are set to have the same area, but a very short length, so that the distance between the pores is very low.

``` python
>>> geom['pore.area'] = Lc**2
>>> geom['pore.diameter'] = Lc
>>> geom['throat.length'] = 1e-25
>>> geom['throat.area'] = Lc**2

```

A similar approach is used to define the geometry of the boundary pores **Geometry** object.

``` python
>>> Ps = pn.pores('boundary')
>>> boun = OpenPNM.Geometry.GenericGeometry(network=pn, pores=Ps)
>>> boun['pore.area'] = Lc**2
>>> boun['pore.diameter'] = 1e-25

```

## Generating the Phases and Physics objects

Now a **Phase** object is defined and associated with a corresponding **Physics** object. On the **Phase** object, the thermal conductivity is set equal to one across the whole domain. The **Physics** object is then generated and associated with the pores and throats of *pn* and the **Phase** object that was just defined. Next a model is chosen for the thermal conduction simulation, which is the *series_resistors* model. As a last step of this section, the **Physics** object is regenerated so that the conductance values are updated everywhere across the whole domain.

``` python
>>> # Create Phase object and associate with a Physics object
>>> Cu = OpenPNM.Phases.GenericPhase(network=pn)
>>> Cu['pore.thermal_conductivity'] = 1.0  # W/m.K
>>> phys = OpenPNM.Physics.GenericPhysics(network=pn,
...                                       phase=Cu,
...                                       pores=pn.pores(),
...                                       throats=pn.throats())
>>> mod = OpenPNM.Physics.models.thermal_conductance.series_resistors
>>> phys.add_model(propname='throat.thermal_conductance', model=mod)
>>> phys.regenerate()  # Update the conductance values

```

## Generating the Algorithms objects and running the simulation

The last step in the OpenPNM simulation involves the generation of a **Algorithm** object and running the simulation. The **Algorithm** object is generated on the **Network** object *pn* and the **Phases** object *Cu*. Then the inlet and outlet regions are defined, with the inlets on just one side of the domain, and the outlets on the three remaining sides of the domain. The absolute value of the boundary condition is chosen to be of sinusoidal shape, with a maximum in the middle of the inlet pores. The boundary condition type here is chosen to be of *Dirichlet* form. Assigning the boundary conditions was achieved by using the *set_boundary_conditions()* method, specifying the location in form of the inlet/outlet pores and the respective absolute value of the boundary conditions. The **Algorithm** object is run using *run()* and subsequently *return_results()* yields the results.

``` python
>>> # Setup Algorithm object
>>> Fourier_alg = OpenPNM.Algorithms.FourierConduction(network=pn, phase=Cu)
>>> inlets = pn.pores('back_boundary')
>>> outlets = pn.pores(['front_boundary', 'left_boundary', 'right_boundary'])
>>> T_in = 30*sp.sin(sp.pi*pn['pore.coords'][inlets, 1]/5)+50
>>> Fourier_alg.set_boundary_conditions(bctype='Dirichlet',
...                                     bcvalue=T_in,
...                                     pores=inlets)
>>> Fourier_alg.set_boundary_conditions(bctype='Dirichlet',
...                                     bcvalue=50,
...                                     pores=outlets)
>>> Fourier_alg.run()
>>> Fourier_alg.return_results()

```

This is the last step usually required in a OpenPNM simulation. The algorithm was run, and now the simulation data obtained can be analyzed. For illustrative purposes, the results obtained using OpenPNM shall be compared to an analytical solution of the problem in the following.

## Computing the analytial solution to the problem and comparing to the simulation

The analytical solution is computed in *Python* as well, and the result is assigned to the OpenPNM simulation domain in form of the *pore.analytical_temp* key on the *Cu* dictionary. The variable *sim* holds the simulation result, while *analyt* holds the analytical solution. Both are reshaped to a two dimensional domain and subtracted to yield the difference in both values.

``` python
>>> # Calculate analytical solution over the same domain spacing
>>> Cu['pore.analytical_temp'] = 30*sp.sinh(sp.pi*pn['pore.coords'][:, 0]/5)/sp.sinh(sp.pi/5)*sp.sin(sp.pi*pn['pore.coords'][:, 1]/5) + 50
>>> analyt = Cu['pore.analytical_temp'][pn.pores(geom.name)]
>>> sim = Cu['pore.temperature'][pn.pores(geom.name)]
>>> sim = sp.reshape(sim, (divs[0], divs[1]))
>>> analyt = sp.reshape(analyt, (divs[0], divs[1]))
>>> diff = sim - analyt

``` 

The results can be visualized using Matplotlib. First Matplotlib has to be imported e.g. as ``import matplotlib.pyplot as plt``, after which the results can be plotted using either ``plt.imshow(sim,interpolation='none')`` for the computed simulation results that were saved in the variable *sim* or ``plt.imshow(diff,interpolation='none')`` for the difference between the simulation results and the analytical solution that were saved in the variable *diff*. Calling ``plt.colorbar()`` will give a colorbar.

The resulting images are shown below. The first image shows the simulation results. The second image shows the absolute difference between the simulation and the analytical solution. It can be seen that the relative difference between analytical and simulation is on the order of 10<sup>-4</sup>.

### Simulation result obtained from OpenPNM 

![](http://i.imgur.com/vSu0pIn.png)

### Absolute difference between simulation result from and analytical solution

![](http://i.imgur.com/iEAVlqO.png)




