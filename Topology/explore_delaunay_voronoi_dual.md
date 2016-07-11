# Generate Random Networks Based on Delaunay Triangulations,  Voronoi Tessellations, of Both

A random network offers several advantages over the traditional Cubic arrangement: the topology is more 'natural' looking, and a wider pore size distribution can be achieved since pores are not constrained by the lattice spacing.  Random networks can be tricky to generate however, since the connectivity between pores is difficult to determine.  One surprisingly simple option is to use Delaunay triangulation to connect pores that are randomly distributed in space.  The Voronoi tessellation is a complementary graph that arises directly from the Delaunay graph.  OpenPNM offers both of these types of network, plus the option to create a network containing *both* including interconnections between Delaunay and Voronoi networks via the ```DelaunayVoronoiDual``` class.

## Generate a Square Network with a DelaunayVoronoiDual Topology

``` python
>>> import OpenPNM as op
>>> pn = op.Network.DelaunayVoronoiDual(num_points=100, domain_size=[1, 1, 1])

```
