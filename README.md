# bvh ![Build Status](https://github.com/madmann91/bvh/workflows/build-and-test/badge.svg)

> Note: This is the `v2` branch of this library, and it is in an highly unstable, unfinished state,
> so please stick to the `master` branch, or expect constant breakage and inconsistent results.

This is a modern C++17 header-only BVH library optimized for ray-tracing. Traversal and
construction routines support different primitive types. The design is such that the
BVH only holds nodes, no primitive data. There is no hardware- or platform-specific
intrinsic used. Parallelization is done using OpenMP. There is no dependency
except the C++ standard library.

![Example rendering generated by a path tracer using this library](render.jpg)
(Scene by Blend Swap user MaTTeSr, available [here](https://www.blendswap.com/blend/18762), distributed under CC-BY 3.0)

## Performance

Here is a comparison of this library with other alternatives ([Embree](https://github.com/embree/embree), [Fast-BVH](https://github.com/brandonpelfrey/Fast-BVH), [nanort](https://github.com/lighttransport/nanort)):

![Comparison of this library vs. other alternatives](chart.png)

These numbers were obtained when rendering the image above, with a path tracing renderer
using single-ray traversal, on an AMD Ryzen Threadripper 2950X. They show that this library
can get very close to Embree, and be orders of magnitude faster than alternatives, all while
being portable and not relying on SIMD intrinsics.

## Detailed Description

Since there are various algorithms for BVH traversal and construction, this library provides
several options that can be used to target real-time, interactive, or offline rendering.

### Construction Algorithms

This library contains several construction algorithms, all parallelized using OpenMP.

 - `bvh::BinnedSahBuilder`: Top-down builder using binning to compute the SAH (see
   _On fast Construction of SAH-based Bounding Volume Hierarchies_, by I. Wald). Relatively fast
   and produces medium- to high-quality trees. Can be configured by setting the number of bins.
 - `bvh::SweepSahBuilder`: Top-down builder that sorts primitives on all axes and sweeps them
   to find the split with the lowest SAH cost. Relatively slow but produces high-quality trees.
 - `bvh::LocallyOrderedClusteringBuilder`: Bottom-up builder that produces trees by sorting
   primitives on a Morton curve and performing a local search to merge them into nodes (see
   _Parallel Locally-Ordered Clustering for Bounding Volume Hierarchy Construction_,
   by D. Meister and J. Bittner). Very fast algorithm that produces medium- to high-quality
   trees (if `bvh::LeafCollapser` is used). The search radius can be configured to change the
   quality/speed ratio. Additionally, the integer type used to compute Morton codes can also be
   changed when more precision is required.
 - `bvh::LinearBvhBuilder`: Bottom-up LBVH builder. Very fast and produces low-quality trees.
   Not substantially faster than `bvh::LocallyOrderedClusteringBuilder`, particularly when the
   search radius of that algorithm is small. Since the BVHs built by this algorithm are significantly
   worse than those produced by other algorithms, do not use this builder unless you absolutely
   need the fastest construction algorithm.
 - `bvh::SpatialSplitBvhBuilder`: Top-down SBVH builder that splits primitives to minimize overlap
   between nodes. Based on the article _Spatial Splits in Bounding Volume Hierarchies_, by M. Stich et al.
   Produces very high-quality trees, at the cost of very slow BVH builds. Use for offline rendering.
   Although it is possible to disable spatial splits and only perform object splits with this builder,
   prefer `bvh::SweepSahBuilder` for this as, unlike this builder, it does not need to sort references
   at every step.

Those algorithms only require a bounding box and center for each primitive, except for
`bvh::SpatialSplitBvhBuilder`, which needs a way to split individual primitives.

### Optimization/Splitting/Refitting Algorithms

Additionally, the BVH structure can be further improved by running post-build optimizations,
or pre-build triangle splitting.

 - `bvh::ParallelReinsertionOptimizer`: An optimization that tries to re-insert BVH nodes
   in a way that minimizes the SAH (see _Parallel Reinsertion for Bounding Volume Hierarchy Optimization_,
   by D. Meister and J. Bittner). This can lead up to a 20% improvement in trace performance,
   at the cost of longer build times.
 - `bvh::HeuristicPrimitiveSplitter`: A pre-splitting algorithm that splits primitives at regular
   positions, inspired by _Fast Parallel Construction of High-Quality Bounding Volume Hierarchies_,
   by T. Karras and T. Aila. Works well in combination with `bvh::LinearBvhBuilder`, but may
   decrease performance for other builders on some scenes. Takes a budget of primitives to split,
   and distributes it to each primitive based on a priority heuristic.
 - `bvh::NodeLayoutOptimizer`: A cheap post-build optimization that reorders the nodes of the BVH in
   memory in such a way that nodes that have a high probability of being hit are likely to already be
   in the cache. This optimization is only really helpful when traversing incoherent rays, but can in
   that case increase performance by up to 10%.
 - `bvh::LeafCollapser`: A post-build optimization that collapses the leaves of the BVH when this is
   beneficial to the SAH cost of the tree. This optimization is only relevant for bottom-up builders
   like `bvh::LinearBvhBuilder` or `bvh::LocallyOrderedClusteringBuilder`, because top-down builders
   already have a stopping criterion that prevents creating leaves that would be detrimental to the
   SAH cost of the tree.
 - `bvh::HierarchyRefitter`: A parallel bottom-up BVH refitting algorithm, to use after changing the
   geometry inside the leaves of the BVH (for example, with animated models). This algorithm will
   simply update the bounding boxes of each node, and propagate the result up to the root. The topology
   of the BVH is left untouched during the operation. Useful when building another BVH is too expensive,
   or when the changes are small enough. Note that BVH quality may decrease as a result.

### Traversal Algorithms

This library provides the following traversal algorithms:

 - `bvh::SingleRayTraverser`: A traversal algorithm optimized for single rays.
    Rays are classified by octant, to make the ray-box test more efficient. The
    traversal order is such that the closest node is taken first. The ray-box
    test does not use divisions, and uses FMA instructions when possible. The
    traversal can also be configured to operate in "robust" mode, in which the
    ray-node intersection does not use FMA instructions, and makes the necessary
    corrections to avoid false-misses (see _Robust BVH Ray Traversal_, by T. Ize).
    
Traversal algorithms can work in two modes: closest intersection,
or any intersection (for shadow rays, usually around 20% faster).
They only require an intersector to compute primitive-ray intersections.

### Intersectors

Intersectors contained in the libary support linear collections of primitives (i.e. arrays/vectors),
and allow permuting the primitive data in such a way that no indirection is done during traversal
(the helper function `bvh::shuffle_primitives` reorders primitives for this exact purpose).
Custom intersectors can be introduced when the primitive data is in another form.

## Building

There is no need to build anything, since this library is header-only.
To build the tests, type:

    mkdir build
    cd build
    cmake ..
    cmake --build .

## Installing

In order to use this library in an existing project using CMake, you can clone, add as submodule, or
even copy the source tree into some subdirectory of your project, say `my-project/contrib/bvh`.
Then, instruct CMake to visit this directory and link against the `bvh` target:

    add_subdirectory(contrib/bvh)
    target_link_libraries(my-project PUBLIC bvh)

That's all that you need to do. Dependencies will be automatically added (e.g. OpenMP, if available).

## Usage

For a basic example of how to use the API, see [this simple example](test/simple_example.cpp).
If you need to library to read your primitive data from another source (if you want to avoid
copying existing primitive data, for instance), take a look at [this example](test/custom_intersector.cpp).
Finally, if triangles are not enough for you, refer to [this file](test/custom_primitive.cpp)
in order to understand how to implement your own primitive type so that it is compatible
with the rest of the API.

## License

This library is distributed under the [MIT](LICENSE.txt) license.
