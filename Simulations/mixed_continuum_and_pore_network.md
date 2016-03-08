# Diffusion in a pore network with continuum nodes

The utility of pore network modeling derives from its ability to simulate transport processes knowing only geometrical properties such as pore size distribution and connectivity between pores.  In some cases however, it is not possible to computationally resolve all the pores.  For example, diffusion may be occurring through a network of large pores on the order of 100's of um, while a reaction occurs in much smaller pores on the order 1 um.  This situation is commonly found when modeling fuel cell electrodes which are made of several sandwiched layers each with drastically different pores sizes.  

This example illustrates how to create a dual-scale pore network, determine transport parameters on the large scale using pore-size information, while ascribing continuum-like transport properties to a smaller scale.  Gas diffusion with reaction will then be simulated on this dual-scale network.  This loosely follows the approach demonstrated in a recent paper in the [Journal of the Electrochemical Society](http://doi.org/10.1149/2.0701605jes) which treated the catalyst layer as a volume averaged porous continua.  

| Highlights |
|------------|
| 1. Generate two distinct Network topologies and stitch them together |
| 2. Assign pore-scale conductance models to some pores based continuum mechanics |
| 3. Add concentration dependent reaction terms to some pores |

## Create Network Topology

Start by creating the network to represent the layer of large pores (~50 um) for the 'gas diffusion layer':

``` python
>>> import OpenPNM as op
>>> GDL = op.Network.Cubic(shape=[10, 10, 10], spacing=0.00006)

```

And next create a second network to represent the catalyst layer, which has 200-500 nm pores (100x smaller than the GDL).  We will not attempt to resolve each pore in the CL, since this would mean 1 million pores for each pore on the GDL (so a total of 40 million pores).  Instead, we'll treat the CL as a network of *nodes* with the properties of a porous continua:

``` python
>>> CL = op.Network.Cubic(shape=[30, 30, 5], spacing=0.00002)

```

This means that each pores in the GDL will interface with 9 pores on the CL.  The `shape` and `spacing` were chosen to make the scaling between the layers straight-forward.

### Stitching Two Networks

OpenPNM includes an assortment of tools for manipulating network topology.  These functions can be found under `OpenPNM.Utilities.topology`.  For this example we'll use the `stitch` method, which as the name suggests 'stitches' a network onto another, in this case `CL` onto `GDL`.

``` python
>>> # Start by assigning labels to each network for identification later
>>> CL['pore.CL'] = True
>>> CL['throat.CL'] = True
>>> GDL['pore.GDL'] = True
>>> GDL['throat.GDL'] = True
>>> # Next manually offset CL one full thickness relative to the GDL
>>> CL['pore.coords'] -= [0, 0, 0.00002*5]
>>> # Finally, send both networks to stitch which will stitch CL onto GDL
>>> pn = op.Utilities.topology.stitch(network=GDL, donor=CL,
                                      P_network=GDL.pores('bottom'),
                                      P_donor=GDL.pores('top'),
                                      len_max=0.00003)

```

Visualizing the network lattice in Paraview shows that indeed the `CL` was offset the correct amount in the correct direction, and that new throats have been added that creat the desired *stitching*.  

![](url)


## Create Pore-Scale Geometry Objects for each Region

For this example, we'll use the pre-defined Toray090 **Geometry** class which is a typical gas diffusion layer material used in fuel cells, but we'll create a custom **Geometry** for the catalyst layer section.  The following uses the `GDL` and `CL` labels we created prior to stitching.  

``` python
>>> geom_GDL = op.Geometry.Toray090(network=GDL, pores=GDL.pores('GDL'),
...                                 throats=GDL.throats('GDL'))
>>> geom_CL = op.Geometry.GenericGeometry(network=GDL, pores=GDL.pores('CL'),
...                                       throats=GDL.throts('GDL'))

```

The **Toray090** class contains all the pore and throat pore-scale models required, but we'll need to add models to `geom_CL`. Also, the *stitch* process created numerous new throat connections between the `CL` and `GDL` that are need some sort of geometrical information.  

``` python
>>> Ts = GDL.find_interface_throats(labels=['GDL', 'CL'])
>>> geom_stitch = op.GenericGeometry(network=GDL, throats=Ts)

```
