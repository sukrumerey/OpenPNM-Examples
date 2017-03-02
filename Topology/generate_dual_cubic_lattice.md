# Generate a Cubic Lattice with an Interpenetrating Dual Cubic Lattice

(New in version 1.6) OpenPNM now offers two options for generating *dual* networks.  This tutorial will outline the use of the basic *CubicDual* class, while the *DelaunayVoronoiDual* is covered elsewhere.  The main motivation for creating these dual networks was to enable the modeling of transport in the void phase on one network and through the solid phase on the other.  These networks are interpenetrating but not overlapping or coincident so it makes the topology realistic.  Moreover, these networks are interconnected to each other so they can exchange species between them, such as gas-liquid heat transfer.  The tutorial below outlines how to setup a *CubicDual* network object, describes the combined topology, and explains how to use labels to access different parts of the network.

As usual start by importing Scipy and OpenPNM:

``` python
>>> import scipy as sp
>>> import OpenPNM as op
>>> mgr = op.Base.Workspace()  # Initialize a workspace object
>>> mgr.clear()  # Really only necessary for testing purposes

```

Let's create a *CubicDual* and visualize it in Paraview:

``` python
>>> net = op.Network.CubicDual(shape=[6, 6, 6])

```

The resulting network has two sets of pores, labelled as blue and red in the image below.  By default, the main cubic lattice is referred to as the 'primary' network which is colored *blue*, and the interpenetrating dual is referred to as the 'secondary' network shown in *red*.  These names are used to label the pores and throats associated with each network.  These names can be changed by sending ```label_1``` and ```label_2``` arguments during initialization.  The throats connecting the 'primary' and 'secondary' pores are labelled 'interconnect', and they can be seen as the diagonal connections below.

![](https://i.imgur.com/3KRduQh.png)

Inspection of this image shows that the 'primary' pores are located at expected locations for a cubic network including on the faces of the cube, and 'secondary' pores are located at the interstitial locations.  There is one important nuance to note: Some of 'secondary' pores area also on the face, and are offset 1/2 a lattice spacing from the internal 'secondary' pores.  This means that each face of the network is a staggered tiling of 'primary' and 'secondary' pores.  

The 'primary' and 'secondary' pores are connected to themselves in a standard 6-connected lattice, and connected to each other in the diagonal directions.  Unlike a regular *Cubic* network, it is not possible to specify more elaborate connectivity in the *CubicDual* networks since the throats of each network would be conceptually entangled.  The figure below shows the connections in the secondary (left), and primary (middle) networks, as well as the interconnections between them (right).  

![](https://i.imgur.com/mVUhSP5.png)

Using the labels it is possible to query the number of each type of pore and throat on the network:

``` python
>>> net.num_pores('primary')
216
>>> net.num_pores('secondary')
275
>>> net.num_throats('primary')
540
>>> net.num_throats('secondary')
450
>>> net.num_throats('interconnect')
1600

```

Now that this topology is created, the next step would be to create *Geometry* objects for each network, and an additional one for the 'interconnect' throats:

``` python
>>> geo_pri = op.Geometry.GenericGeometry(network=net,
...                                       pores=net.pores('primary'),
...                                       throats=net.throats('primary'))
>>> geo_sec = op.Geometry.GenericGeometry(network=net,
...                                       pores=net.pores('secondary'),
...                                       throats=net.throats('secondary'))
>>> geo_inter = op.Geometry.GenericGeometry(network=net,
...                                         throats=net.throats('interconnect'))

```

The above code block  the *GenericGeometry* class for each geometry, so they will not have any pore or throat size information.  You must add your own pore-scale models from the **Geometry.models** library.  
