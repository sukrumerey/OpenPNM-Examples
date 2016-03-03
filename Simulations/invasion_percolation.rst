.. _IP-example:

===============================================================================
Predicting Fluid Configurations using Invasion Percolation
===============================================================================

Invasion percolation (IP) describes the pore-by-pore invasion of a non-wetting phase into a porous material.  It is physically analogous to an experiment where fluid is injected at a constant rate using a syringe pump, but can be used to describe physical processes where the detailed sequence of pore filling is important.  The original application of IP to porous materials was presented by `Wilkinson and Willemsen <http://dx.doi.org/10.1088/0305-4470/16/14/028>`_ and a detailed analysis of IP in terms of general percolation theory was given by `Sheppard et al <http://doi.org/10.1088/0305-4470/32/49/101>`_.

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Generating the Network, adding Geometry and creating Phases
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Start by generating a basic cubic **Network** and the other required components.  We'll use a 2D network to illustrate the algorithm since it's easier for visualization.

.. code-block:: python

    >>> import OpenPNM
    >>> pn = OpenPNM.Network.Cubic(shape=[20, 20, 1], spacing=0.0001)

Next we need to create a **Geometry** object to manage the pore and throat size information, and a **Phase** object to manage the thermophysical properties of the invading fluid.

.. code-block:: python

    >>> geom = OpenPNM.Geometry.Toray090(network=pn, pores=pn.Ps, throats=pn.Ts)
    >>> water = OpenPNM.Phases.Water(network=pn)

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Define the Pore-scale Physics
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

To perform most algorithms, it is necessary to define the pore scale physics that relates pore/throat geometry, phase properties, and the mechanism to be modeled.  In the case of an IP simulation, it is necessary to define the pressure at which the invading phase can enter each throat.  This is commonly done by assuming the throat is a cylinder and using the so-called "Washburn" equation.  The OpenPNM **Physics** module has a folder for capillary pressure methods, including the "Washburn" model.  To use it in a simulation, you first create a generic **Physics** object, then add the model as follows:

.. code-block:: python

    >>> phys = OpenPNM.Physics.GenericPhysics(network=pn, phase=water,
		...                                       geometry=geom)
    >>> phys.add_model(propname='throat.capillary_pressure',
    ...                model=OpenPNM.Physics.models.capillary_pressure.washburn)

This means that ``phys`` will now have a model called ``'throat.capillary_pressure'``, that when called will calculate throat entry pressures using the ``'washburn'`` model.  The Washburn model requires that the **Phase** object (``water`` in this case) has the necessary physical properties of surface tension and contact angle.

The predefined **Water** class is assigned a contact angle of 110 degrees by default (water on Teflon). This can be confirmed by printing the first element of the ``'pore.contact_angle'`` array:

.. code-block:: python

    >>> print(water['pore.contact_angle'][0])
    110.0

To change this value, the ``'pore.contact_angle'`` property of ``water`` can be set to a new constant:

.. code-block:: python

   >>> water['pore.contact_angle'] = 140.0

However, this change will *not* be reflected in the ``'throat.capillary_pressure'`` array until the ``phys`` object's models are ``regenerated``:

.. code-block:: python

    >>> phys.models.regenerate()

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Run an Invasion Simulation
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

At this point, the system is fully defined and ready to perform some simulations.  An IP algorithm object is instantiated as follows:

.. code-block:: python

    >>> IP = OpenPNM.Algorithms.InvasionPercolation(network=pn)
    >>> IP.setup(phase=water)

Before running the algorithm it is necessary to specify the inlet sites from where the invading fluid enters the network:

.. code-block:: python

    >>> IP.set_inlets(pores=pn.pores('left'))

The final step is to invaded the network.  This is accomplished with the ``run`` method of the IP object.

.. code-block:: python

    >>> IP.run()

This method produces arrays called ``'pore.invaded'`` and ``'throat.invaded'`` on the IP object that contain the invasion sequence of each pore and throat, respectively.  It is also possible to visualize the partial invasion in Paraview starting by exporting the data to a 'VTK' file:

.. code-block:: python

    >>> IP.return_results()
    >>> OpenPNM.export_data(network=pn, filename='IP', fileformat='VTK')

The top image in the figure below shows the invasion pattern in the network with each pore (sphere) colored according to the order it was invaded, with blue invaded early and red invaded last.  You can see that the smaller pores are colored red since these are likely to be connected to small throats.  In the bottom image at *Threshold* filter has been applied in Paraview to show only pores invaded in the first 200 steps, so a specific invasion pattern can be clearly seen.

.. image:: http://i.imgur.com/tFftRVA.png

To obtain a specific invading fluid configuration at some intermediate invasion state in OpenPNM (for instance the first 200 invasions) for use in a subsequent simulations such as relative permeability, it is simply a matter of applying a Boolean operator to the ``'pore.invaded'`` and ``'throat.invaded'`` arrays such as:

.. code-block:: python

    >>> Pinv = IP['pore.invaded'] < 200
    >>> Tinv = IP['throat.invaded'] < 200

More control of the invasion sequence is also possible.  The ``run`` command takes an option argument of ``n_steps``, which if given performs a partial invasion of the network.  This approach is required if you wish to perform more complex invasions such as 100 steps from the 'left', then 100 from the 'right'.  In this case the invasion pattern will not be the same as if the invasion had proceeded entirely from the 'left'.
