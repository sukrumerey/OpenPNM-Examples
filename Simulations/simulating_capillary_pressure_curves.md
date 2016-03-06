# Drainage Curve on a Cubic

## Generating the Network, adding Geometry and creating Phases

Start by generating a basic **Cubic** network and the other required components:

``` python
>>> import OpenPNM
>>> pn = OpenPNM.Network.Cubic(shape=[10, 10, 10], spacing=0.0001)
>>> pn.add_boundaries()

```

The last call adds a layer of boundary pores on all sides of the network after it is generated. These boundary pores will be used in the following calculations. There is an alternative method call ``add_boundary_pores`` that allows fine grained control of boundary pore addition but it is more cumbersome to use.

Next create a **Geometry** to manage the pore and throat size information.  A **Geometry** can span over a part of the **Network** only, so we need to specify to which pores and throats this **Geometry** object should apply. For this example, we want one **Geometry** to apply to the internal pores and a different one to the boundary pores:

``` python
>>> Ps = pn.pores('*boundary')  # Use wildcard to get all boundary pores
>>> boun = OpenPNM.Geometry.Boundary(network=pn, pores=Ps)
>>> Ps = pn.pores('internal')  # Only non-boundary pores
>>> Ts = pn.throats()  # All throats
>>> geom = OpenPNM.Geometry.Stick_and_Ball(network=pn, pores=Ps, throats=Ts)

```

The ``Stick_and_Ball`` **Geometry** is a predefined class that applies normally distributed pore and throat sizes to the internal pores based on the size of the lattice spacing (which is infers from the network).  The ``Boundary`` class is predefined with properties suitable for boundaries such as 0 volume and length.

We must also create the **Phase** objects:

``` python
>>> Hg = OpenPNM.Phases.Mercury(network=pn)
>>> air = OpenPNM.Phases.Air(network=pn)

```

## Define the Pore-Scale Physics

To perform most algorithms, it is necessary to define the pore scale physics that relates pore/throat geometry, phase properties, and the mechanism to be modeled.  In the case of a drainage curve simulation, it is necessary to define the pressure at which the invading phase can enter each throat.  This is commonly done by assuming the throat is a cylinder and using the so-called 'Washburn' equation.  The OpenPNM **Physics** module has a subfolder for capillary pressure models, including the Washburn model.  To use this model in a simulation, you first create a generic **Physics** object:

``` python
>>> phys = OpenPNM.Physics.GenericPhysics(network=pn, pores=pn.Ps,
...                                       throats=pn.Ts, phase=Hg)

```

Then add the desired models to this object using:

``` python
>>> mod = OpenPNM.Physics.models.capillary_pressure.washburn
>>> phys.add_model(propname='throat.capillary_pressure', model=mod)

```

This means that ``phys`` will now have a model called 'capillary_pressure', that when called will calculate throat entry pressures using the 'washburn' model.  The Washburn model requires that the **Phase** object it's attached  (mercury in this case) has the necessary physical properties of surface tension and contact angle.

    | **Note** :  Both surface tension and contact angle are actually 'phase system' properties, rather than solely water properties.  It is an open problem in OpenPNM to figure out how to treat these sort of properties more rigorously.  For the present time, they must be entered a single phase properties, in this case in ``Hg``.

## Run a Drainage Simulation

At this point, the system is fully defined and ready to perform some simulations.  A typical algorithm used in pore network modeling is to use ordinary percolation to simulate drainage of wetting phase by invasion of a non-wetting phase.  An **Algorithm** object is created as follows:

``` python
>>> MIP = OpenPNM.Algorithms.Drainage(network=pn)

```    

Before performing simulations with this algorithm it is necessary to specify the desired experimental parameters with the ``setup`` command:

``` python
>>> MIP.setup(invading_phase=Hg, defending_phase=air)

```

This step tells the MIP algorithm where to find the required physical properties (i.e on ``Hg`` and the **Physics** associated with it), as well as which **Phase** objects to write the resulting ``'pore.occupancy'`` values.

Next, we specify through which pores mercury enters the network.

``` python
>>> MIP.set_inlets(pores=pn.pores('*boundary'))

```

In MIP experiment this is all pores on the surface which are actually the "boundary" pores we created above.  These are found using the *wildcard* operator with the ``'boundary'`` label.

  | **Advanced Features**: It is also possible to call the ``set_outlets`` method to specify through pores which the defending phase exits the network, but this is not relevant to an MIP simulation since the sample is evacuated of air.  Moreover, it's possible to simulate secondary drainage by using the ``set_residual`` method, but this requires knowing the locations of the residual non-wetting phase from some other simulation.

Now the ``MIP`` algorithm is ready to ``run``:

``` python
>>> MIP.run()

```

It is possible to specify the number of pressure points to test with the ``npts`` argument (default is 25) or which specific points to test by passing a list to the ``inv_pressures`` argument.

This algorithm produces 4 data arrays and stores them on the ``MIP`` object.  ``'pore.inv_seq'`` and ``'throat.inv_seq'`` contain the sequence or step number where each invasion occurred.  These are reported for consistency with the IP algorithms.  ``'pore.inv_Pc'`` and ``'throat.inv_Pc'`` contain the applied pressure at which each pore and throat was invaded at.  Any of these can be used to extract a specific invasion pattern by applying a Boolean mask such as:

``` python
    >>> Ts = MIP['throat.inv_Pc'] < 2000
    >>> Ps = MIP['throat.inv_Pc'] < 2000

```

### Plotting the Capillary Pressure Curve

It is possible using the information stored on the ``MIP`` object to reproduce the capillary pressure curve manually.  Since this is such a common operation, however, the **Drainage** class has methods already available for doing this.  The raw capillary pressure curve data can be obtained using the ``get_drainage_data`` method, which returns a table of data for plotting in an external program of your choice.  Alternatively, a plot can be created directly with ``MIP.plot_drainage_curve()``.

.. image:: http://i.imgur.com/ZxuCict.png
