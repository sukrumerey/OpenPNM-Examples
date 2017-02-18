## Create Network Topology

OpenPNM includes numerous tools for manipulating and altering the topology.  Most of these are found unter **Network.tools**.  This example will illustrate who to join or 'stitch' two distinct networks, even if they have different lattice spacing.  In this example we'll create a coarse and a fine network then stitch them together to make a network with two distinct layers.  

Start by creating a network with a large lattice spacing:

``` python
>>> # Import the usual suspects
>>> import scipy as sp
>>> import matplotlib.pyplot as pyplot
>>> # Import OpenPNM and initialize the workspace manager
>>> import OpenPNM as op
>>> sim = op.workspace()
>>> sim.clear()  # Clear the existing workspace (mostly for testing purposes)
>>> coarse_net = op.Network.Cubic(shape=[10, 10, 10], spacing=0.00005)

```

The ```coarse_net``` network has 1000 pores in a cubic lattice with a spacing of 50 um for a total size of 500 um per size.  Next, we'll make another network with smaller spacing between pores, but with the same total size.

``` python
>>> fine_net = op.Network.Cubic(shape=[25, 25, 5], spacing=0.00002)

```

These two networks are totally independent of each other, and actually both spatially overlap each other since the network generator places the pores at the [0, 0, 0] origin.  Combining these networks into a single network is possible using the ```stitch``` function, but first we must make some adjustments.  For starters, let's shift the ```fine_net``` along the z-axis so it is beside the ```coarse_net``` to give the layered effect:

``` python
>>> fine_net['pore.coords'] += sp.array([0, 0, 0.0005])

```

Before proceeding, let's quickly check that the two networks are indeed spatially separated now:

``` python
>>> fig = op.Network.tools.plot_connections(coarse_net)
>>> fig = op.Network.tools.plot_connections(fine_net, fig=fig)

```

As can be seen below, ```fine_net``` (orange) has been repositioned above the ```coarse_net``` (blue).

![](https://i.imgur.com/8nEBCpf.png)

Now it's time stitch the networks together by adding throats between the pores on the top of the coarse network and those on the bottom of the fine network. The ```stitch``` function uses Euclidean distance to determine which pore is each face is nearest each other, and connects them.  

``` python
>>> op.Network.tools.stitch(network=fine_net, donor=coarse_net,
...                         P_network=fine_net.pores('bottom'),
...                         P_donor=coarse_net.pores('top'),
...                         len_max=0.00004)

```

And we can quickly visualize the result using OpenPNM's plotting tools:

``` python
>>> fig = op.Network.tools.plot_connections(fine_net)

```

The diagonal throats between the two networks have been added by the stitch process:

![](https://i.imgur.com/6oLq9wv.png)

The next step would be to assign different geometry objects to each network, with different pore sizes and such.
