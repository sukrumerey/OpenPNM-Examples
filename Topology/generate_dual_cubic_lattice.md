# Generate a Cubic Lattice with an Interpenetrating Dual Cubic Lattice

(New in version 1.6) OpenPNM now offers two options for generating *dual* networks.  This tutorial will outline the use of the basic *CubicDual* class, while the *DelaunayVoronoiDual* is covered elsewhere.  The main motivation for creating these dual networks was to enable the modeling of transport in the void phase on one network and through the solid phase on the other.  These networks are interpenetrating but not overlapping or coincident so it makes the topology realistic.  Moreover, these networks are interconnected to each other so they can exchange species between them, such as gas-liquid heat transfer.  The tutorial below outlines how to setup a *CubicDual* network object, describes the combined topology, and explains how to use labels to access different parts of the network.

As usual start by importing Scipy and OpenPNM:

``` python
>>> import scipy as sp
>>> import OpenPNM as op
>>> mgr = op.Base.Workspace()
>>> mgr.clear()  # Really only necessary for testing purposes

```

Let's create a *CubicDual* and visualize it in Paraview:

``` python
>>> net = op.Network.CubicDual(shape=[5, 5, 5])

```

The resulting network has two sets of pores, labelled as blue and red in the image below.  By default, the main cubic lattice is referred to as the 'primary' network which is colored red, and the interpenetrating dual is referred to as the 'secondary' network shown in blue.  These names are used to label the pores and throats associated with each network.  These names can be changed by sending `label_1` and `label_2` arguments during initialization.  The throats connecting the 'primary' and 'secondary' pores are labelled 'interconnect', and they can be seen as the diagonal connections below.

![](https://i.imgur.com/3KRduQh.png)

Inspection of this image shows that the 'primary' pores are located at the center of a unit cell with 'secondary' pores found on each corner.  'primary' and 'secondary' pores are connected to themselves in a standard 6-connected lattice, and connected to each other in the diagonal directions.  The figure below shows the connections in the primary (left), and secondary (right) networks, as well as the interconnections between them.

![](https://i.imgur.com/mVUhSP5.png)

Using the labels is possible to query the number of each type of pore and throat on the network:

``` python
>>> net.num_pores('primary')
275
>>> net.num_pores('secondary')
216
>>> net.num_throats('primary')
450
>>> net.num_throats('secondary')
540
>>> net.num_throats('interconnect')
1600

```

There is one important nuance regarding the topology.  A standard cubic network with a shape of [5, 5, 5] has 125 pores. In this case, however, a [5, 5] layer of pores has been added to each face offset away from the domain by 1/2 a lattice spacing.  This puts them on the same plane as the 'secondary' network pores, and creates a flat surface at the boundary of the domain.  These pores are accordingly labelled 'surface':

``` python
>>> net.num_pores(['primary', 'surface'], mode='intersection')
150

```
