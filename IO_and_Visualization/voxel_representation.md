# Creating a Voxel Image of a Pore Network

It is often useful to have a 3D image of your pore network.  It can be used for visualization (although VTK is better for that), but also for performing direct numerical simulations such a estimating permeability using the Lattice-Boltzmann method, or simulating MIP drainage curves using morphological image opening.  

``` python
>>> import scipy as sp
>>> import OpenPNM as op

```

## Import Statoil 'dat' Files
