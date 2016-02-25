# Statoil Import Example

``` python
import scipy as sp
import OpenPNM as op
assert op.__version__ == '1.4.0'
```

## Import Statoil 'dat' Files

The following assumes that the folder containing the 'dat' files is in the same directory as this script:

``` python
path = r"ICL-Sandstone(Berea)"
pn = op.Utilities.IO.Statoil.load(path=path, prefix='Berea')
pn.name = 'berea'
```

This import class extracts all the information contained in the 'Statoil' files produced by the Maximal Ball network extraction code developed by Martin Blunt's group at Imperial College London.  The code is available from him upon request, but they offer a small library of pre-extracted networks on their [website] (https://www.imperial.ac.uk/engineering/departments/earth-science/research/research-groups/perm/research/pore-scale-modelling/micro-ct-images-and-networks/).

At this point, the network can be visualized in Paraview.  A suitable '.vtp' file can be created with:

``` python
op.export_data(network=pn, filename='imported_statoil')
```

## Adding Boundary Pores

Upon importing these networks, OpenPNM does perform a 'optimizations' to make the network compatible.  The main problem is that the original network contains a large number of throats connecting actual internal pores to fictitious 'reservoir' pores.  OpenPNM strips away all these throats since 'headless throats' break the graph theory representation, but labels the real internal pores as either 'inlet' or 'outlet' if there were connected to one of these fictitious reservoirs.

### Reading Reservoir Pores

Identify inlet pores by their label

``` python
Pin = pn.pores('inlets')
```

For inlet reservoir, we must create a new pore to act as the reservoir, connect it to the pores on the inlet face:

``` python
XYZ = sp.mean(pn['pore.coords'][Pin], axis=0)
pn.extend(pore_coords=XYZ, labels='inlet_reservoir')
```

Now we want to connect this reservoir pore to the pores on the inlet face of the network.  First we need to identify the pore number of the new reservoir pore just created:

``` python
N = pn.pores('inlet_reservoir')
```

Next we need to create a list of pore-to-pore connections between the reservoir pore and the inlet pores.  The details of OpenPNM topological representation are given elsewhere, but basically we indicate that pores 'a' and 'b' are connected by adding a row to the 'throat.conns' array containing [a, b].  The addition of new throats is a bit more involved that that, so a method is available on Network objects to handle this for us.  We only need to send in the list of new pore-to-pore connections:

``` python
conns = sp.vstack([Pin, N*sp.ones_like(Pin)]).T
pn.extend(throat_conns=conns, labels='to_inlet_reservoir')
```

Finally, we want to offset the inlet reservoir pore away from the internal network pores, but we don't necessarily know which way to move it or by how much.  The following code checks the x, y and z coordinates of the inlet pores and detects which dimension has the least spread.

``` python
extents = sp.ptp(pn['pore.coords'][Pin], axis=0)
offset_dim = sp.argmin(extents)
pn['pore.coords'][-1, offset_dim] = pn['pore.coords'][-1, offset_dim] - \
                                    extents[offset_dim]
```

Now repeat for the outlet reservoir pore:

``` python
Pout = pn.pores('outlets')
XYZ = sp.mean(pn['pore.coords'][Pout], axis=0)
pn.extend(pore_coords=XYZ, labels='outlet_reservoir')
N = pn.pores('outlet_reservoir')
conns = sp.vstack([Pout, N*sp.ones_like(Pout)]).T
pn.extend(throat_conns=conns, labels='to_outlet_reservoir')
extents = sp.ptp(pn['pore.coords'][Pout], axis=0)
offset_dim = sp.argmin(extents)
pn['pore.coords'][-1, offset_dim] = pn['pore.coords'][-1, offset_dim] + \
                                    extents[offset_dim]
```

Since we've added two new pores, the network is now incomplete because the reservoir pores and the throats connecting them have no physical properties. This can be observed by printing the network:

``` python
print(pn)

#------------------------------------------------------------
OpenPNM.Network.GenericNetwork: 	berea
#------------------------------------------------------------
#     Properties                          Valid Values
#------------------------------------------------------------
1     pore.coords                          6300 / 6300
2     pore.radius                          6298 / 6300
3     pore.shape_factor                    6298 / 6300
4     pore.volume                          6298 / 6300
5     throat.conns                        12545 / 12545
6     throat.length                       12098 / 12545
7     throat.radius                       12098 / 12545
8     throat.shape_factor                 12098 / 12545
9     throat.total_length                 12098 / 12545
10    throat.volume                       12098 / 12545
#------------------------------------------------------------
#     Labels                              Assigned Locations
#------------------------------------------------------------
1     pore.all                            6300
2     pore.clay_volume                    0
3     pore.inlet_reservoir                1
4     pore.inlets                         201
5     pore.outlet_reservoir               1
6     pore.outlets                        246
7     throat.all                          12545
8     throat.clay_volume                  0
#------------------------------------------------------------
```

The new reservoir pores can now be seen in Paraview, by exporting a 'vtp' file:

``` python
op.export_data(network=pn, filename='imported_statoil_with_reservoirs')
```

### Adding OpenPNM-Style Inlet and Outlet Boundary Pores
