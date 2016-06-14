# Loading Networks Saved in MATLAB

OpenPNM has the ability to load networks generated in MATLAB, saved as a specially formatted \*.mat file.

> **NOTE:** OpenPNM has several more general ways to import networks from outside sources.  The ``MatFile`` class in the **Network** module was created very early in the OpenPNM development, so is mainly kept for legacy reasons.  The preferred format is to import from a CSV file.  Details of how to format the CSV file can be found in the documentation for that class which is found in ``OpenPNM.Utilities.IO.CSV``.  

## MAT File Format

If you have created a pore network in MATLAB and you would like to import it into OpenPNM. Create the following variables:

| Variable Name  | Value        | Description                      |
|----------------|--------------|----------------------------------|
| pcoords        | <Npx3> float | physical coordinates, in meters, of pores to be imported  |
| pdiameter      | <Npx1> float | pore diamters, in meters         |
| pvolume        | <Npx1> float | pore volumes, in cubic meters    |
| pnumbering     | <Npx1> int   | = 0:1:Np-1                       |
| tconnections   | <Ntx2> int   | pore numbers of the two pores that each throat connects    |
| tdiameter      | <Ntx1> float | throat diameters, in meters      |
| tnumbering     | <Ntx1> int   | = 0:1:Nt-1                       |


An example file is part of this repository in the ``fixtures`` directory which you can view in Matlab.

## Importing with MAT

Once you have correctly formatted a \*.mat file, it can be loaded with the following commands, assuming it is stored in a folder called ``fixtures`` in your current working directory:

``` python
>>> import OpenPNM
>>> import os
>>> fname = os.path.join('fixtures', 'example_network.mat')
>>> pn = OpenPNM.Network.MatFile(filename=fname)
>>> # This class also create a geometry object and attaches it to the network
>>> geom_name = pn.geometries()[0]
>>> geom = pn.geometries(geom_name)

```

Of course you store it any folder you wish, so long as you give the correct path in the ``os.path.join`` command.

## Additional Pore and Throat Properties

Additional properties and labels can be added to the network as long as they are arrays of the correct length. For example, if you saved the \*.mat file with the variable `pshape` that was an Npx1 float array, you could include it in your import as follows:

``` python
>>> pn = OpenPNM.Network.MatFile(filename=fname, xtra_pore_data='shape')

```

This will fail unless you actually build a \*.mat file with a variable named "pshape"

## Adding Surfaces and Boundaries to Network with ptype and ttype

There is no add_boundaries() command for this Network class. But, you can add `ptype` and `ttype` variables to the \*.mat file in order to store "surface" and "boundary" information. In OpenPNM, "boundary" pores are zero-volume pores just outside the network domain, that typically gain boundary conditions in simulations. Conventionally, there is one "boundary" pore placed for every "surface" pore in the system, where "surface" pores are pores on the very edge of the domain.

Currently, this class has a built-in, but rather inflexible method that assumes the users knows what they are doing. Follow these instructions carefully:

In order for the network to import boundaries, the pore and throat data `ptype` and `ttype` must be present.

| Variable Name  | Value      | Description                      |
|----------------|------------|----------------------------------|
| ptype          | <Npx1> int | (optional) designates surfaces of pores in network. |
| ttype          | <Ntx1> int | (optional) designates surfaces of throats in network. |

The `type` variables are integers between 0 and 6. All internal pores, including "surface" pores, should have the value 0. The rest should be labelled as follows: 1-top, 2-left, 3-front, 4-back, 5-right, 6-bottom.

Importing a network with boundaries can be done as follows:

``` python
>>> pn = OpenPNM.Network.MatFile(filename=fname,
...                              xtra_pore_data='type',
...                              xtra_throat_data='type')

```

Because the `type` variables are loaded, the importer automatically adds labels for boundaries and surfaces.
