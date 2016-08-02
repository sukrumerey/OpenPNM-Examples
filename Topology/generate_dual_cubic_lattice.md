# Generate a Cubic Lattice with an Interpenetrating Dual Cubic Lattice

(New in version 1.6) OpenPNM now offers two options for generated dual networks.  This tutorial will outline the use of the basic *CubicDual* class, while the *DelaunayVoronoiDual* is covered elsewhere.  The main motivation for creating these dual networks was to  allow users to model transport in the void phase on one network and through the solid phase on the other.  That these networks are interpenetrating but not overlapping or coincident makes the topology more realistic.  Moreover, these networks are interconnected to each other so exchange of species can occur between the, such as gas-liquid heat transfer.  The tutorial below outlines how to setup a *CubicDual*, describes the topology, and explains how to use the labels to access different parts of the network.

As usual start by importing Scipy and OpenPNM:

``` python
>>> import scipy as sp
>>> import OpenPNM as op
>>> mgr = op.Base.Workspace()
>>> mgr.clear()  # Really only necessary for testing purposes

```

Let's create a *CubicDual* and a basic *Cubic* for comparison:

``` python
>>> dual = op.Network.CubicDual(shape=[5, 5, 5])
>>> cube = op.Network.Cubic(shape=[5, 5, 5])

```

Comparing the number of pores and throats on each network reveals that the *DualCubic* indeed has more of both:

``` python
>>> Np_dual = dual.num_pores()
>>> Nt_dual = dual.num_throats()
>>> Np_cube = cube.num_pores()
>>> Nt_cube = cube.num_throats()
>>> Np_dual > Np_cube
True
>>> Nt_dual > Nt_cube
True

```

In fact,  ```dual``` has nearly 3x more pores than ```cube```, and more than 20x more throats. The reason for the additional pores arise because the dual network that is interpenetrating into the has ```shape + 1``` pores (i.e. [6, 6, 6] in this case).  Thi
