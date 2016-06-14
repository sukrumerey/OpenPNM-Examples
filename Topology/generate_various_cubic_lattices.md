# Generate Cubic Lattices of Various Shape, Sizes and Topologies

The Cubic lattice network is easily the most commonly used pore network topology.  When people first learn about pore network modeling they often insist on creating network that are topologically equivalent or representative of the real network (i.e. random networks extracted from tomography images).  In reality, however, a simple cubic network provides a very passable representation of more complex topologies, and provides several additional benefits as well; namely they are much easier to visualize, and applying boundary conditions is easier since the faces of the network are flat.  

The examples below will demonstrate how to create various cubic lattice networks in OpenPNM using the **Cubic** class, as well as illustrating a few topological manipulations that can be performed, such as adding boundary pores, and trimming throats to create a more random-like topology.

## Basic Cubic Lattice

Let's start with the most basic cubic lattice:

``` python
>>> import OpenPNM as op
>>> pn = op.Network.Cubic(shape=[10, 10, 10], spacing=[1, 1, 1])

```

In this case ```pn``` will be a 10 x 10 x 10 *cube* with each pore spaced 1 unit away from it's neighbors in all directions.  Each pore is connected to the 6 neighbors adjacent to each *face* of the cubic lattice site in which it sits.  
