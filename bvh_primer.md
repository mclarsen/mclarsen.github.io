# BVH Construction Primer

There are two main types of BVH builders:

- Top down builders (think recursive splits)
- Bottom up builders (construction starts at the leafs)

I note that that this brief summary is a broad description of BVH construction methods.
This is a rich area of computer graphics research.

## BVH Quality
Several metrics for measuring the quality of a BVH have been used over the years,
but the main metric is called the [Surface Area Heuristic](https://link.springer.com/article/10.1007/BF01911006) (SAH).
The main intuition behind SAH is based on minimizing the surface area of the bounding boxes contained
the hierarchy. Considering a random distribution of rays, the probability of hitting a node in the BVH
is proportional to its surface area. The cost of traversing down node is also dependent on how many
primitives and other nodes are beneath it in the tree. For a high level explanation, see
[this article](https://medium.com/@bromanz/how-to-create-awesome-accelerators-the-surface-area-heuristic-e14b5dec6160).

Higher quality trees result in a smaller number of axis-aligned bounding box (AABB) tests, and in
typical triangle-based ray tracing, ray-AABB test dominate over ray-triangle intersections. This is
why new NVIDIA cards have hardware specifically for ray-AABB tests.

### Notes on the SAH
In the above referenced article on the SAH, there are several magic values that are left undefined.

*t_intersect* is the *cost* of intersecting a primitive. A primitive could be a triangle, where
the relative cost of intersection is cheap. That said, other primitives could be a higher order
surface such as a bezier surface of arbitrary order. In the bezier surface case, intersection methods
can involve iterative newton solves, which are very expensive when compared to the cost of intersecting
a triangle.

*t_traversal* is the *cost* of intersecting an AABB. Generally this cost is low because there
exist efficient ray-AABB tests, but during the course of a ray traversing a BVH, there generally
are many more ray-AABB than ray-primitive tests.

Hardware architecture is also a factor in determining traversal costs. GPU and CPU architectures are
different, and consequently, the *costs* for different floating-point operations are different.

So how do I determine these costs? One choice is to ignore them completely and give them a value of 1.
Another choice is to empirically determine these values. Often the values in publications are not specified
for a variety of reasons, including that their testing hardware is most likely not the same as yours.
That said, these values effect decisions made during BVH construction and should be carefully considered.

## Top Down Construction

The typical gold standard, in terms of quality, in BVH construction is the [Split BVH](https://www.nvidia.com/docs/IO/77714/sbvh.pdf).
In a nutshell, top down builders look at many different possible splits (along each axis) at each level and they
select the best one based on the SAH. Split BVHs also perform spatial splits of the primitives in order
to maximize the SAH of the tree. A full-sweep method would consider every possible split along each axis.
An optimization of for top down builders is binning. Instead of
considering every possible split, the builder considers every *n* splits, thus reducing the overall amount of
computation. Binned building can often achieve high quality when compared to a builder that considers
all possible splits. One downside to top down builders is the limited amount of parallelism available
at the top of the tree.

## Bottom Up Construction

As the name suggests, bottom up constructors start at the leafs and build upwards.
An exemplar of bottom up construction is the
[Linear BVH](https://research.nvidia.com/sites/default/files/publications/karras2012hpg_paper.pdf)(LBVH).
The main idea here is to build an implicit radix tree based on [Morton codes](https://en.wikipedia.org/wiki/Z-order_curve).
Bottom up construction exposes more parallelism than top down builders and can be constructed very quickly.
That said, the SAH is not used and the resulting trees are of lower quality.

## Hybrid Approaches
There are methods to combine top down and bottom up construction methods. An example of a hybrid method
is [Approximate Agglomerative Clustering](http://graphics.cs.cmu.edu/projects/aac/aac_build.pdf)(AAC).
AAC uses Morton codes and the algorithm consists of top down and bottom up phases.

Another type of construction method is based on optimizing the topology of a cheaply constructed
LBVH. While the initial construction of the LBVH is not based on the SAH, the optimization passes
re-order the topology of sub-trees considering the SAH. The treelet restructuring BVH,
[TRBVH](https://research.nvidia.com/sites/default/files/pubs/2013-07_Fast-Parallel-Construction/karras2013hpg_paper.pdf)
does just that. The TRBVH takes mutliple bottom up passes, and each pass increases the overall quality
of the BVH. The result can achieve about 98% if the quality of a high quality top down builder, while
achieving interactive construction speeds.
