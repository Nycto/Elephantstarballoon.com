+++
date = 2025-03-21T08:00:00-08:00
title = 'Spacial Indexing: The Adaptive Hash Grid Algorithm'
weight = 2
+++

I've been exploring spatial indexing lately, trying to balance performance and
simplicity in scenarios where objects of varying sizes need to be indexed and
queried quickly. This led me to experiment with what I'm calling an **Adaptive
Hash Grid** (AHGrid). It may not be entirely novel, but I think it's an
interesting approach to tackling a few common challenges. I've combined ideas
from a few other algorithms to make this work.

## The Problem with Traditional Grids

If you've ever used a basic grid to index objects in a 2D space, you've probably
hit a familiar problem: What happens when an object spans multiple cells?

Imagine a grid where each cell is exactly `10 units` wide and `10 units` tall.
If you insert a an object that is 15x15 into that grid, it will overlap multiple
cells, forcing you to register the object with each cell it overlaps. When
querying for nearby objects, you have to check all four cells and deduplicate any
results if they straddle multiple cells.

Quadtrees and Hierarchical Spatial Hash Grids have the same problem as a classical
fixed size grid -- they all require storing an object in all the cells that it overlaps.
This is because the edges between cells are fixed.

I also wanted a system that could represent objects of wildly varying size spread
out across a basically unbound space (`low(int32)..high(int32)`).

## Adaptive Hasing Grid

AHGrid overcomes these limitations in two ways:

1. There are multiple layers in the grid. Each layer is 2x as big as the previous layer.
2. The boundaries between each layer are always at an offset from the layer beneath it.

You can see what that means here:

![ahgrid layers](/ahgrid/scales.svg)

Each layer scales up by 2, and the first index is offset from the layer above.

When inserting an object into the grid, it means we can avoid overlaps by first finding the
smallest layer that can fully contain that object. A simple example of that can be
seen here:

![ahgrid layers](/ahgrid/insert-simple.svg)

In that example, the object is `3 units` wide at offset `3`. We can immediately bump the
width up to the next power of 2, which puts us at a scale of `4`. And because the object
fits entirely within the cell from `2` to `5`, we're done.

If the object overlaps the edge of its cell, we move to the next layer up until we
find a layer that doesn't overlap. This allows us to guarantee that each object is
only ever inserted into exactly one cell. You can see the same object as below, but this
time an offset of `4` puts the right side of the object over the boundary of the cell:

![ahgrid layers](/ahgrid/insert-bumped.svg)

In this case, we can bump the scale up to `8` and the object now fits entirely within
a single cell at the next layer up.

### In code

To see how this works, lets look at the implementation. Imagine we want to insert 2D object
with `x`, `y`, `width` and `height` parameters:

```nim
type
  SpatialObject* = concept obj
    obj.x is int32
    obj.y is int32
    obj.width is int32
    obj.height is int32
```

**First**, we choose the layer that best fits this object. We're calling this
the `scale`. Because each layer is a power of 2, we just need to find the next
power of two based on the maximum width or height:

```nim
var scale = max(obj.width, obj.height).int.nextPowerOfTwo.int32
```

**Second**, we choose the cell index that the object would fall into. This looks
the same for both `x` and `y`.  In a Hierarchical Spatial Hash Grid, this
would just be `coord div scale`. But because we need to offset the root
coordinate for each layer by half the scale, we need to get a bit fancier:

```nim
proc cellIndex(coord, scale: int32): int32 =
  let half = scale div 2
  let adjust = if coord + half >= 0: 0'i32 else: -scale + 1
  return (coord + half + adjust) div scale * scale - half
```

**Third**, we make sure the object we're processing fits entirely within the
cell we've picked. If it doesn't, then we need to increase the `scale` of the
cell until the object fits entirely within it. We call the output of this a
`CellIndex` because it's the value we later use to index all the objects that
are near each other.

```nim
type CellIndex = tuple[x, y, scale: int32]

proc pickCellIndex(grid: AHGrid, x, y, width, height: int32): CellIndex =
  var scale = max(width, height).int.nextPowerOfTwo.int32
  while true:
    let output = (x: x.cellIndex(scale), y: y.cellIndex(scale), scale: scale)
    if x + width < output.x + scale and y + height < output.y + scale:
        return output
    scale = scale * 2
```

Put that all together and we have now picked the cell that we know will fit the entire
object in it.

## Storage and Inserting

As the name of the algorithm would imply, the values being index are stored in a table.
All the values with the same `CellIndex` are stored together, as they, by definition,
are near each other. It looks like this:

```nim
type
  AHGrid*[T: SpatialObject] = object
    maxScale: int32
    cells: Table[CellIndex, seq[T]]
```

Notice that we also need to store the `maxScale`. That's in there so that queries know
how many layers they need to query before stopping.

With the type definition above, inserting an object can be done in linear time:

```nim
proc insert*[T](grid: var AHGrid[T], obj: T) =
  let key = obj.pickCellIndex(grid)
  grid.maxScale = max(grid.maxScale, key.scale)
  grid.cells.mgetOrPut(key, newSeq[T]()).add(obj)
```

## Querying at a point

To now find all the objects that are near a given point, we can implement a query by
calculating the `CellIndex` for each `scale` within the grid. Visually, if you wanted
to query for anything that exists at the 1d coordinate `8`, it would look like this:

![query at point](/ahgrid/query-point.svg)

In code, that looks like:

```nim
iterator eachScale(grid: AHGrid): int32 =
  var scale = 0
  while scale <= grid.maxScale:
    yield scale
    scale *= 2

iterator find*[T](grid: AHGrid[T]; x, y: int32): T =
  for scale in grid.eachScale:
    let key = pickCellIndex(x, y, scale)
    for obj in grid.cells.getOrDefault(keys):
      yield obj
```

This dynamic scaling adapts smoothly to the size and spread of the data, minimizing unnecessary checks.

## Querying with a radius

We can further expand the querying capability by adding in a radius for our query.
This requires doing a search for every `CellIndex` that is within the radius of
the search point:

![query at point](/ahgrid/query-range.svg)

In code, that looks like:

```nim
iterator eachCellIndex(x, y, radius, scale: int32): CellIndex =
  let xRange = cellIndex(x - radius, scale)..cellIndex(x + radius, scale)
  let yRange = cellIndex(y - radius, scale)..cellIndex(y + radius, scale)

  for x in countup(xRange.a, xRange.b, scale):
    for y in countup(yRange.a, yRange.b, scale):
      yield (x, y, scale)

iterator find*[T](grid: AHGrid[T]; x, y, radius: int32): T =
  for scale in grid.eachScale:
    for key in eachCellIndex(x, y, radius, scale):
      for obj in grid.cells.getOrDefault(key):
        yield obj
```

## Optimizing usage

To make the most of the AHGrid approach, there are a few optimization strategies to consider:

* Initial Sizing for Hash Tables and Lists

  When initializing your hash tables and nested lists, it’s helpful to estimate
  the number of cells and objects you expect to store. By pre-allocating space
  in your hash table and object lists, you can avoid costly reallocations as the
  data grows. If you expect a large, sparse distribution of data, you may want
  to use a more aggressive initial capacity for the hash table.

* Adding a Minimum Scale (`minScale`)

  Another critical parameter that can be added to this implementation is the
  notion of a "minimum scale", or `minScale`. This value sets the smallest cell
  size used by the grid, acting as a lower bound for the `scale` factor. If the
  `minScale` is too small, queries will need to search through many cells,
  increasing the cost of each lookup. If it’s too large, you lose the
  granularity and benefits of a spatial index.

## Alternate Approaches and Comparisons

While the Adaptive Hash Grid has proven effective for my needs, it's helpful to understand how it compares to other spatial hashing techniques. Here are a few common approaches:

* Fixed-Size Grids: These are simple to implement but suffer from overlapping issues when objects span multiple cells. They also struggle with varying object sizes or non-uniformly distributed data.
* Quadtrees: Quadtrees dynamically divide space, making them suitable for varied object sizes. However, they can become unbalanced with irregular data distributions and often require complicated balancing or depth-limited optimizations.
* Hierarchical Spatial Hash Grids: These are more flexible than fixed grids, leveraging multiple levels of granularity. However, objects still overlap cells across levels, necessitating deduplication during querying.

The Adaptive Hash Grid strikes a balance between these approaches. By adjusting cell sizes dynamically while maintaining a flat hash table, AHGrid reduces overlaps, minimizes query time, and simplifies implementation.

## Strengths of this approach

- **Simplicity:** Uses a flat hash table — no deep tree traversals, no rebalancing. This
  algorithm is _really_ easy to implement
- **Adaptability:** Supports objects of any size within a single cell.
- **Efficient Queries:** No deduplication needed due to unique cell assignment.
- **Infinite Space:** Because we're using a hash table to store the data, we can store
  two objects that are at opposite ends of a grid as efficiently as storing two that
  are close together

## Conclusion

I've been using this for Game development, where I need to quickly and dynamically
index a large number of game entities -- it has worked well so far!

If you're interested in seeing all this code together, I've got a working implementation
over on Github: https://github.com/Nycto/AHGrid