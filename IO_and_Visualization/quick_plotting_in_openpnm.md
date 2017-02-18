# Producing Quick and Easy Plots of Topology within OpenPNM

The main way to visualize OpenPNM networks is Paraview, but this can be a bit a hassle when building a new network topology that needs quick feedback for troubleshooting.  Starting in V1.6, OpenPNM offers two plotting functions for showing pore locations and the connections between them: ```OpenPNM.Network.tools.plot_coordinates``` and ```OpenPNM.Network.tools.plot_connections```. This example demonstrates how to use these two methods.

Start by initializing OpenPNM and creating a network.  For easier visualization we'll use a 2D network:

``` python
>>> import OpenPNM as op
>>> op.Base.Workspace().clear() # Clear workspace (mostly for testing purposes)
>>> net = op.Network.Cubic(shape=[5, 5, 1])

```

Next we'll add boundary pores to two sides of the network, to better illustrate these plot commands:

``` python
>>> net.add_boundaries(['left', 'right'])

```

Now let's use ```plot_coordinates``` to plot the pore centers in a 3D plot, starting with the internal pores:

``` python
>>> Ps = net.pores('internal')
>>> fig = op.Network.tools.plot_coordinates(network=net, pores=Ps, c='b')

```

Note that the above call to ```plot_coordinates``` returns a figure handle ```fig```.  This can be passed into subsequent plotting methods to overlay points.

``` python
>>> Ps = net.pores('*boundary')
>>> fig = op.Network.tools.plot_coordinates(network=net, pores=Ps, fig=fig, c='r')

```

![](https://i.imgur.com/5RfcOBb.png)

Next, let's add lines to the above plot indicating the throat connections. Again, by reusing the ```fig``` object we can overlay more information:

``` python
>>> Ts = net.throats('boundary', mode='not')
>>> fig = op.Network.tools.plot_connections(network=net, pores=Ts, fig=fig, c='b')
>>> Ts = net.throats('*boundary')
>>> fig = op.Network.tools.plot_connections(network=net, pores=Ts, fig=fig, c='r')

```

![](https://i.imgur.com/cgAIw6O.png)

These two methods are meant for quick and rough visualizations.  If you required pretty and fancy 3D images, you should use Paraview!

![](https://i.imgur.com/uSBVFi9.png)
