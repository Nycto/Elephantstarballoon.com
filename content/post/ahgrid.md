+++
date = 2025-03-21T08:00:00-08:00
title = 'Spacial Indexing: The Adaptive Hash Grid Algorithm'
weight = 2
+++

I've been exploring spatial indexing lately, trying to find a balance
performance and simplicity. While looking at the Hierarchical Spacial Grid Hash
algorithm, I had a thought about how to combine its layering system with a flat
spatial hash. I've been searching for prior art to determine if this is a "new"
idea, but haven't found any yet. I'm sharing this approach to collect feedback.

For the purpose of this article, I'm calling this the **Adaptive Hash Grid**
algorithm (`AHGrid`). As I said, it's a twist on a spatial hash that
incorporates the size of the object into the hash key generation. The benefit is
that objects in an `AHGrid` will only ever be stored in a single cell within the
grid, which simplifies the process of queries and updates. It's also capable of
easily storing objects of varying size spread out across a basically unbound
space (`low(int32)..high(int32)`).

## The Problem with Traditional Grids

If you've ever used a basic grid to index objects in a 2D space, you've probably
hit a familiar problem: What happens when an object spans multiple cells?

Imagine a grid where each cell is exactly `10 units` wide and `10 units` tall.
If you insert an object that is 15x15 into that grid, it will overlap multiple
cells, forcing you to register the object with each cell it overlaps.

![ahgrid layers](/ahgrid/grid-overlap.svg)

When querying for nearby objects, you have to check all four cells and deduplicate any
results if they straddle multiple cells. Querying for neighbors can quickly
balloon to searching through dozens of cells.

Quadtrees and Hierarchical Spatial Hash Grids have the same problem as a classical
fixed size grid -- they require storing an object in all the cells that it overlaps.

## Adaptive Hashing Grid

AHGrid is very similar to a spatial hash grid in that it stores objects in a multimap
using a calculated index to colocate objects that are nearby:

```nim
Table[CellIndex, seq[T]]
```

In a regular spacial hash, `CellIndex` above would essentially be:

```nim
type CellIndex = tuple[xBucket, yBucket: int32]
  ## CellIndex for a regular spatial hash grid

proc cellIndexForSpatialHash(x, y: int32): CellIndex =
  return (xBucket: x div cellSize, yBucket: y div cellSize)
```

`AHGrid` also uses an `xBucket` and `yBucket`, but then adds a concept of `scale`:

```nim
type CellIndex = tuple[xBucket, yBucket, scale: int32]
  ## CellIndex for an AHGrid
```

The `scale` property holds the size of the object, rounded up to the next power of two.
The calculation for the scale is exactly what you would expect:

```nim
proc calculateScale(width, height: int32): int32 =
  max(width, height).int.nextPowerOfTwo.int32
```

This create an important property of an `AHGrid`: there are multiple layers
transparently overlaid within the same `Table`. Each new value for `scale`
constitutes a new layer. For each layer, the scale is twice as big as the
previous layer, so it can contain objects that are twice as big. This is very
similar to how a Hierarchical Spatial Hash Grid works in concept, but flattened
into a single storage container.

The next twist is that every layer in an `AHGrid` offsets its root coordinate by
half its size. This leads to a very important property: adjacent layers don't
usually share the same edges between cells. Put another way, the boundaries
between each cell in a layer are always at an offset from the layer beneath it.
In one dimensional space, that looks like this:

![ahgrid layers](/ahgrid/scales.svg)

This offset is what allows us to put an object into exactly one cell -- if an
object overlaps an edge in one layer, we just need to bump up to the next layer.

Being in one cell allows queries and updates to be more efficient, as we don't
have to worry about deduplicating the results. We know that we can blindly emit
all the values within a cell without tracking what has already been returned.

Let's walk through the insertion process to see how this works.

## Insert operation

To insert an object into the grid, we start by finding the smallest layer that
can fully contain that object:

![ahgrid layers](/ahgrid/insert-simple.svg)

Above, the object is `3 units` wide at an offset of `3`. We can immediately choose an
initial `scale` by bumping the size up to the next power of 2, which puts us at a scale
of `4`. And because the object fits entirely within the cell from `2` to `5`, we're done.

However, if we do the above operation and the object overlaps the edge of its
destination cell, we need to keep working. To ensure it fits in exactly one
cell, we move up to the next layer until we find a cell where the object doesn't
overlap the boundaries.

![ahgrid layers](/ahgrid/insert-bumped.svg)

In the image above, you can see the same object as before, but this time it has
an offset of `4`. That puts the right edge over the right side boundary of the
cell. In this case, we can bump the scale up to `8` and the object now fits
entirely within a single cell.

### In code

Lets look at the code for this. Imagine we want to insert a 2D object with `x`,
`y`, `width` and `height` parameters:

```nim
type
  SpatialObject* = concept obj
    obj.x is int32
    obj.y is int32
    obj.width is int32
    obj.height is int32
```

**First**, we choose the smallest layer that best fits this object. Because the cell size for
each layer is a power of 2, we just need to find the next power of two based on the maximum
width or height. That's the function we already showed above:

```nim
var scale = calculateScale(obj.width, obj.height)
```

**Second**, we examine the `x` and `y` coordinates and independently choose a
cell index for each. This is basic bucketing -- and it the same for both `x` and `y`, so we
can create a general function that works for both.

In a Hierarchical Spatial Hash Grid, this would just be `coord div scale`. But
because we need to offset the root coordinate for each layer by half the scale,
we need to get a bit fancier:

```nim
proc chooseBucket(coord, scale: int32): int32 =
  let half = scale div 2
  # We need to specifically adjust the index to handle negative coordinates. This
  # looks funky because we also have to deal with the shifting root coordinates
  let adjust = if coord + half >= 0: 0'i32 else: -scale + 1
  return (coord + half + adjust) div scale * scale - half
```

**Third**, we make sure the object we're processing fits entirely within the
cell we've picked. If it doesn't, then we need to increase the `scale` of the
cell until the object can be fully contained by a single cell.

```nim
proc pickCellIndex(grid: AHGrid, x, y, width, height: int32): CellIndex =
  var scale = calculateScale(width, height)
  while true:
    let output = (xBucket: x.chooseBucket(scale), yBucket: y.chooseBucket(scale), scale: scale)

    # Ensure the object fits entirely within the cell
    if x + width < output.xBucket + scale and y + height < output.yBucket + scale:
        return output

    # If it doesn't, try fitting the object into the next layer up
    scale = scale * 2
```

Put that all together and we have now picked a single cell that we know will fit
the entire object.

## Storage and Inserting

As already mentioned, the object being indexed are stored in a table of lists. All the
values with the same `CellIndex` are stored together, as they -- by definition --
are near each other. It looks like this:

```nim
type
  AHGrid*[T: SpatialObject] = object
    maxScale: int32
    cells: Table[CellIndex, seq[T]]
```

Notice that we also need to store the `maxScale`. That's in there so that queries know
how many layers they need to query before stopping. Any time we insert an object, we need
to update that `maxScale`

With the type definition above, inserting an object can be done in linear time:

```nim
proc insert*[T](grid: var AHGrid[T], obj: T) =
  let key = obj.pickCellIndex(grid)
  grid.maxScale = max(grid.maxScale, key.scale)
  grid.cells.mgetOrPut(key, newSeq[T]()).add(obj)
```

## Querying at a point

To find all the objects that are near a given point, we can implement a query by
calculating the `CellIndex` for each `scale` within the grid at the point of
interest. Visually, if you wanted to query for anything that exists at the 1d
coordinate `8`, it would look like this:

![query at point](/ahgrid/query-point.svg)

In the chart above, we would visit every cell at every layer that intersects with
the dashed line. In code, that looks like:

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

Within game development, it's common to need to know everything that is within
a specific radius of a given point. For example, finding all the enemies that might
be impacted by an explosion.

To handle this use case, we can add a radius to our query.  This requires doing
a search for every `CellIndex` that is within the radius of the search point:

![query at point](/ahgrid/query-range.svg)

Again, we would need to visit every cell that intersects with or is between the
two dashed lines above. In code, that looks like:

```nim
iterator eachCellIndex(x, y, radius, scale: int32): CellIndex =
  ## Yields each cell key within a given radius of a point at the given scale
  let (xLow, xHigh) = (chooseBucket(x - radius, scale), chooseBucket(x + radius, scale))
  let (yLow, yHigh)= (chooseBucket(y - radius, scale), chooseBucket(y + radius, scale))

  for x in countup(xLow, xHigh, scale):
    for y in countup(yLow, yHigh, scale):
      yield (x, y, scale)

iterator find*[T](grid: AHGrid[T]; x, y, radius: int32): T =
  for scale in grid.eachScale:
    for key in eachCellIndex(x, y, radius, scale):
      for obj in grid.cells.getOrDefault(key):
        yield obj
```

## Further Optimizations

Beyond what is described here, there are a few optimization strategies that need
further exploration:

* Initial Sizing for Hash Tables and Lists

  When initializing your hash tables and nested lists, it’s helpful to estimate
  the number of cells and objects you expect to store. By pre-allocating space
  in the hash table and object lists, you can avoid costly reallocations as the
  data grows.

* Skip the hash table entirely

  Technically, we don't even need a `Table`. We could use an array of arrays
  and do the index management ourselves. This would let us guarantee that we never
  have to deal with table resizing.

* Adding a Minimum Scale (`minScale`)

  Another critical parameter that can be added to this implementation is the
  notion of a "minimum scale", or `minScale`. This value sets the smallest cell
  size used by the grid, acting as a lower bound for the `scale` factor. If the
  `minScale` is too small, queries will need to search through many cells,
  increasing the cost of each lookup. If it’s too large, you lose the
  granularity and benefits of a spatial index.

* Use a `HashSet` instead of a `seq`

  In the example code, I'm using a `Table` that contains a `seq`. One possible
  optimization would be using a `HashSet` instead. Though I didn't discuss the
  `remove` or `update` operations in this article, using a set would potentially
  make those operations faster.

## Strengths

- **Simplicity:** Uses a flat hash table — no deep tree traversals, no rebalancing. This
  algorithm is _really_ easy to implement
- **Adaptability:** Supports objects of any size at any position
- **Efficient Queries:** No deduplication needed due to unique cell assignments

## Weaknesses

* **Cache Locality:** Because we're using a hash table, memory locality could
  suffer. This is especially noticeable if the grid is large and queries need to
  traverse multiple levels of scaling. Compared to a linear or tree-based
  structure, the cache efficiency might be lower.

* **Large Query Radii:** A large query radius is going to cause a lot of cells
  to be visited, reducing efficiency. This could be a bigger issue if large
  queries are frequent

* **Integer only:** This implementation relies on a lot of integer based math and
  doesn't really work for floats. Any usage would first need to convert a float to
  an int. This is trivial with a fixed point system, though.

* **Scaling Factor Sensitivity:** Choosing the right `minScale` is crucial. If the scaling
  is off, it can lead to either too many grid levels (increasing overhead) or too few (leading
  to dense cells that degrade query performance).

## Conclusion

There are certainly some improvements and extensions that can be made to this implementation.
For example, being able to remove objects or update them in place. Supporting 3D objects, too.
Those should be fairly straight forward.

I've been using this for Game development, where I need to quickly and dynamically
index a large number of game entities -- it has worked well so far!

If you're interested in seeing all this code together, I've got a working implementation
over on Github: https://github.com/Nycto/AHGrid