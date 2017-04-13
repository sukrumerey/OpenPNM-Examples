# Loading Networks from CSV Files

CSV stands for "comma separated values".  This file type is as old as computers, and it is easily edited/viewed/managed using any spreadsheet program or text editor.  It's also perfect for dealing with OpenPNM data which is naturally stored in columns, so CSV is the preferred IO format.  This example will explain the file format and show how to read it into OpenPNM.

## The CSV format
There are only a few rules to this format:

1.  The data are all stored in columns, like in a spreadsheet.  In fact using Excel to prepare and edit your data is recommended.  Almost all programming languages can work with this type of data.
2.  The first row of the each column should contain text that will become the property name.  The property names must conform to the OpenPNM convention that all pore data starts with 'pore' and all throat data starts with 'throat'.  Eg. ```'throat.length'``` and ```'pore.diameter'```.
3.  All subsequent rows contain data.  All columns that are labeled as pore data (i.e. ```'pore.foo'```) must have the same number of rows (```Np```), and similarly all columns labeled as throat data must have the same number (```Nt```).  ```Np``` does not have to equal ```Nt```.
3.  Each *comma* in the file indicates the end of a column.  Storing multiple values in a single column, like the pores coordinates [X, Y, Z], is done by omitting commas as in [X Y Z].  Brackets are ignored.
4.  To store labels, you can enter ```T, F, True, False, true, false, 0, and 1```.  All of these are interpreted as Boolean arrays and converted to labels.
5.  **Most importantly**, the file must contain ```'pore.coords'``` and ```'throat.conns'```.  All other data are optional, but without these two column the network cannot be defined.  These two names are not flexible either...OpenPNM is hard coded to use these.

Here is a screenshot of a sample CSV file in MS Excel:

![](http://i.imgur.com/1Pst2Ve.png)

## Using the CSV Import class

The following few lines are sufficient to import a network stored in a CSV file:

``` python
>>> import OpenPNM as op
>>> import os
>>> fname = os.path.join('fixtures', 'example_csv.csv')
>>> pn = op.Utilities.IO.CSV.load(fname)
>>> print(pn.props())
------------------------------------------------------------
1       : pore.area
2       : pore.coords
3       : pore.diameter
4       : pore.index
5       : pore.seed
6       : pore.volume
7       : throat.area
8       : throat.conns
9       : throat.diameter
10      : throat.length
11      : throat.seed
12      : throat.surface_area
13      : throat.volume
------------------------------------------------------------

```

There is one *Gotcha* to be aware of.  All of the import methods dump all the data from the file onto the network (```pn```).  OpenPNM generally likes only topological information on *Network* objects, and geometrical information on *Geometry* objects.  There is a method on all the ```IO``` classes called ```split_geometry``` that accepts the network and removes everything except ```pore.coords```, ```throat.conns``` and all labels, and puts them on a new *Geometry* object, which get returned.  The following illustrates:

``` python
>>> geom = op.Utilities.IO.CSV.split_geometry(pn)
>>> print(net.props())
------------------------------------------------------------
1       : pore.coords
2       : pore.index
3       : throat.conns
------------------------------------------------------------

>>> print(geom.props())
------------------------------------------------------------
1       : pore.area
2       : pore.diameter
3       : pore.index
4       : pore.seed
5       : pore.volume
6       : throat.area
7       : throat.diameter
8       : throat.length
9       : throat.seed
10      : throat.surface_area
11      : throat.volume
------------------------------------------------------------

```
