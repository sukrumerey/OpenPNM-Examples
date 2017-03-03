# Creating a Voxel Image of a Pore Network

It is often useful to have a 3D image of your pore network.  It can be used for visualization (although VTK is better for that), but also for performing direct numerical simulations such a estimating permeability using the Lattice-Boltzmann method, or simulating MIP drainage curves using morphological image opening, or opening ImageJ and analyzing the pore space using your favorite plugins or custom macros.  Perhaps you want to find the 2-point correlation function, or the chord-length distribution.  Whatever the reason, OpenPNM provides an option to output the pore space as a network as a numerical array that can be saved as a tiff stack, and subjected to the same work flow as a tomography image.  

## Create Cubic *Stick and Ball* Network

To illustrate, let's create a small basic cubic network:

``` python
>>> import scipy as sp
>>> import OpenPNM as op
>>> net = op.Network.Cubic(shape=[5, 6, 7])
>>> geom = op.Geometry.Stick_and_Ball(network=net, pores=net.Ps, throats=net.Ts)

```

## Output Voxel Image

The voxel generator function is found in ```OpenPNM.Utilities.misc``` for lack of a better place to put it.  This function accepts a *Network* object as an argument, allow with a few other options to be explained below.

``` python
>>> im = op.Utilities.misc.generate_voxel_image(net)

```

The above function returns a Numpy ND-array of 2's to represent pore bodies, 1's for throats, and 0's for the solid phase.  This can be saved as 3D tiff image using the ```mimsave``` function from the excellent [imageio](https://imageio.github.io/) package.  It can then be opened in Paraview directly as a tiff stack to yield the following:

![](https://i.imgur.com/obDUqEx.png)

Sometimes it's not possible to resolve all the features in some networks, especially when the pore size distribution is wide.  For instance, there is a small pore in the top left corner that is only a few voxels, and it's throats, which are even smaller, are not visible since they are too small to show up at the given resolution.

### Changing Image Resolution

To avoid the issue of missing features due to lack of resolution, its easy to specify a high resolution, meaning more voxels per unit volume. This function accepts an argument called ```maxdim``` which specifies the size, in voxels, of the largest dimension of the image.  The function then inspects the dimensions of the network and scales the output image so that the longest dimension is scaled to ```maxdim``` voxels in the output image.  This is clearly a convoluted way to specify resolution, but it offers a key advantage: It's very easy to know how large the resultant image will be.  This avoid the issue of accidentally creating an image with 1000's of voxels in each direction.  The following code and image show the same image as above, but with 500 voxels in the z-direction, while the default is 200.

``` python
>>> im = op.Utilities.misc.generate_voxel_image(net, maxdim=500)

```

![](http://i.imgur.com/ZDtPvwz.png)

### Changing Pore and Throat Shape

The function accepts two arguments to control the shape of the pores and throats.  They can either spheres and cylinders (the default) or you can specify cubes and cuboids.  This is done as follows:

``` python
>>> im = op.Utilities.misc.generate_voxel_image(net, pshape='cube', tshape='cuboid')

```

![](https://i.imgur.com/hAMWicU.png)
