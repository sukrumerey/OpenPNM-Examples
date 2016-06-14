# Generate Cubic Lattices of Various Shape, Sizes and Topologies

The Cubic lattice network is easily the most commonly used pore network topology.  When people first learn about pore network modeling they often insist on creating network that are topologically equivalent or representative of the real network (i.e. random networks extracted from tomography images).  In reality, however, a simple cubic network provides a very passable representation of more complex topologies, and provides several additional benefits as well; namely they are much easier to visualize, and applying boundary conditions is easier since the faces of the network are flat.  

The examples below will demonstrate how to create various cubic lattice networks in OpenPNM using the **Cubic** class, as well as illustrating a few topological manipulations that can be performed, such as adding boundary pores, and trimming throats to create a more random-like topology.

## Basic Cubic Lattice with Different Connectivity

Let's start with the most basic cubic lattice:

``` python
>>> import OpenPNM as op
>>> pn = op.Network.Cubic(shape=[10, 10, 10], spacing=[1, 1, 1])

```

In this case ```pn``` will be a 10 x 10 x 10 *cube* with each pore spaced 1 *unit* away from it's neighbors in all directions.  Each pore is connected to the 6 neighbors adjacent to each *face* of the cubic lattice site in which it sits.  The image below illustrates the resulting network with pores shown as white spheres, along with a zoomed in view of the internals, showing the connectivity of the pores.

![](http://i.imgur.com/JTUodGy.png)

The **Cubic** network generator applies 6-connectivity by default, but different values can be specified.  In a cubic lattice, each pore can have up to 26 neighbors: 6 on each face, 8 on each corner, and 12 on each edge.  This is illustrated in the image below.  

![](http://i.imgur.com/ACiQFtJ.png)

Cubic networks can have any combination of corners, edges, and faces, which is controlled with the ```connectivity``` argument by specifying the total number of neighbors (6, 8, 12, 14, 18, 20, or 26):

``` python
>>> pn = op.Network.Cubic(shape=[10, 10, 10], spacing=[1, 1, 1], connectivity=26)

```

This yields the following network, which clearly has a LOT of connections!

![](http://i.imgur.com/PS6d7CO.png)

## Trimming Random Throats to Adjust Coordination Number

Often it is desired to create a distribution of coordination numbers on each pore, such that some pores have 2 neighbors and other have 8, while the overall average may be around 5.  It is computationally very challenging to specify a specific distribution, so OpenPNM does not offer this feature (yet); however it can be approximated manually by creating a highly connected network, and then trimming random throats to reduced the coordination numbers.  The following code block randomly selects 500 throats, then trims them from the network:

``` python
>>> import scipy as sp
>>> pn = op.Network.Cubic(shape=[10, 10, 10], spacing=[1, 1, 1], connectivity=26)
>>> pn.num_throats()
10476
>>> throats_to_trim = sp.random.randint(low=0, high=pn.Nt-1, size=500)
>>> pn.trim(throats=throats_to_trim)
>>> pn.num_throats()
9976
>>>

```

The following image shows histogram of the pore connectivity before and after trimming.  Before trimming the coordination numbers fall into 4 distinct bins depending on where the pores lies (internal, face, edge or corner), while after trimming the coordination numbers show some distribution around their original values.  If the trimming is too aggressive, OpenPNM might report an error message saying that isolated pores exist, which means that some regions of the network are now disconnected from the main network due to a lack of connected throats.

![](http://i.imgur.com/Z4HgMYC.png)

## Creating Domains with More Interesting Shapes

### Rectangular Domains with Non-Uniform Spacing

The ```shape``` and ```spacing``` arguments can of course be adjusted to create domains other than simple cubes:

``` python
>>> pn = op.Network.Cubic(shape=[10, 20, 20], spacing=[0.003, 0.02, 0.01])

```

### Spherical and Other Arbitrary Domains

The **Cubic** class can generate networks of arbitrary shapes (i.e. spheres), but still with *cubic* connectivity.  
