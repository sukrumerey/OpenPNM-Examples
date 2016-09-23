# Statoil Import Example

This example explains how to use the OpenPNM.Utilies.IO.Statoil class to import a network produced by the Maximal Ball network extraction code developed by Martin Blunt's group at Imperial College London.  The code is available from him upon request, but they offer a small library of pre-extracted networks on their [website](https://www.imperial.ac.uk/engineering/departments/earth-science/research/research-groups/perm/research/pore-scale-modelling/micro-ct-images-and-networks/).

``` python
>>> import scipy as sp
>>> import OpenPNM as op

```

##Import Statoil 'dat' Files

The following assumes that the folder containing the 'dat' files is in the same directory as this script:

``` python
>>> import os
>>> path = os.path.join('fixtures', 'ICL-Sandstone(Berea)')
>>> pn = op.Utilities.IO.Statoil.load(path=path, prefix='Berea')
>>> pn.name = 'berea'

```

This import class extracts all the information contained in the 'Statoil' files, such as sizes, locations and connectivity. At this point, the network can be visualized in Paraview.  A suitable '.vtp' file can be created with:

``` python
>>> op.export_data(network=pn, filename='imported_statoil')

```

The resulting network is shown below:

![](http://i.imgur.com/771T36M.png)


### Clean up network topology

Although it's not clear in the network image, there are a number of isolated and disconnected pores in the network.  These are either naturally part of the sandstone, or artifacts of the Maximal Ball algorithm.  In any event, these must be removed before proceeding since they cause problems for the matrix solver.  The easiest way to find these is to use the ```find_clusters2``` method, which returns an Np-long array of cluster numbers given a list of active throats.  If we merely send in the entire list of throats as the active list the disconnected clusters will be found.  We can then use this list to delete the isolated pores using the ```trim``` method:

``` python
>>> clusters = pn.find_clusters2(mask=pn.Ts >= 0)
>>> pn.trim(pores=clusters > 0)

```

## Calculating Permeability of the Core

> **Note:** Upon importing these networks, OpenPNM performs 'optimizations' to make the network compatible.  The main problem is that the original network contains a large number of throats connecting actual internal pores to fictitious 'reservoir' pores.  OpenPNM strips away all these throats since 'headless throats' break the graph theory representation.  OpenPNM then labels the real internal pores as either 'inlet' or 'outlet' if there were connected to one of these fictitious reservoirs.  It is possible to add a new pore to each end of the domain and stitch it to the internal pores labelled 'inlet' and 'outlet', but this is a bit complex. For the sake of this example it is acceptable to apply boundary conditions directly to the 'inlet' and 'outlet' pores.  

### Setup Necessary Geometry, Phase, and Physics Objects

In order to conduct a permeability simulation we must define a **Phase** object to manage the fluid properties, and **Physics** object to manage the pore-scale transport models (i.e. Hagan-Poiseuille equation), and **Geometry** model to calculate the necessary size information.

``` python
>>> geom = op.Geometry.GenericGeometry(network=pn, pores=pn.Ps, throats=pn.Ts)
>>> water = op.Phases.Water(network=pn)
>>> phys = op.Physics.GenericPhysics(network=pn, phase=water, geometry=geom)

```

Before proceeding, we must address a flaw in the way OpenPNM imports (soon to be addressed).  Upon importing a file, ALL the data are stored on the **Network** object, including geometrical information that ought to be stored on a **Geometry**.  It's very easy to move these values to ```geom```:

``` python
>>> for item in pn.props():
...    if item not in ['throat.conns', 'pore.coords']:
...        geom.update({item: pn.pop(item)})

```

Next we must add the additional pore-scale geometry models that will be required by the Hagan-Poiseuille model, namely the pore and throat cross-sectional area, which use diameter instead of radius:

``` python
>>> geom['pore.diameter'] = 2*geom['pore.radius']
>>> geom.models.add(propname='pore.area',
...                 model=op.Geometry.models.pore_area.spherical)
>>> geom['throat.diameter'] = 2*geom['throat.radius']
>>> geom.models.add(propname='throat.area',
...                 model=op.Geometry.models.throat_area.cylinder)

```

### Run StokesFlow Algorithm

Now we can add the Hagan-Poiseuille model to calculate hydraulic conductivity to ```phys```:

``` python
>>> phys.models.add(propname='throat.hydraulic_conductance',
...                 model=op.Physics.models.hydraulic_conductance.hagen_poiseuille)

```

Finally, we can create a **StokesFlow** object to run some fluid flow simulations:

``` python
>>> flow = op.Algorithms.StokesFlow(network=pn, phase=water)
>>> flow.set_boundary_conditions(pores=pn.pores('inlets'), bctype='Dirichlet', bcvalue=200000)
>>> flow.set_boundary_conditions(pores=pn.pores('outlets'), bctype='Dirichlet', bcvalue=100000)

```

The resulting pressure field can be visualized in Paraview, giving the following:

![](https://i.imgur.com/AIK6FbJ.png)

### Determination of Permeability Coefficient

There are two ways to find K, the easy and the hard way.  The easy way is to use the ``find_effective_permeability`` method already implemented on the **StokesFlow** algorithm:

``` python
>>> K = flow.calc_eff_permeability()

```
This gives a value of 4400 mD, which compares reasonably well with the value of 1200 mD reported in the 'Summary Results Berea.txt' file included with the extracted network file.  Presumably this reported K value was determined experimentally, in which case a difference of a few mD is quite good.  

The hard way to calculate K is the determine each of the values in Darcy's law manually and solve for K, such that K = Q*mu*L/(delta_P*A).

``` python
>>> mu = sp.mean(water['pore.viscosity'])
>>> delta_P = 100000
>>> # Using the rate method of the StokesFlow algorithm
>>> Q = sp.absolute(flow.rate(pores=pn.pores('inlets')))
>>> # Because we know the inlets and outlets are at x=0 and x=X
>>> Lx = sp.amax(pn['pore.coords'][:, 0]) - sp.amin(pn['pore.coords'][:, 0])
>>> A = Lx*Lx  # Since the network is cubic Lx = Ly = Lz
>>> K = Q*mu*Lx/(delta_P*A)

```

Using this approach the K values is 4.7 mD, which is essentially identical to the value found using ```calc_eff_permeability```.
