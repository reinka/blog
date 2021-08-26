+++ author = "Andrei Pöhlmann"
title = "How-to: Visiting all neighbors in a grid"
date = "2021-08-26"
description = "Visiting all neighbors in a MxN grid"
tags = [
"leetcode",
"python",
]
categories = [
"algorithms",
"how-to"
]
+++

This post presents a simpole Python3 code snippet that visits all neighbors of a cell in a MxN grid.

# What is it good for?

Graph problems are sometimes formulated as matrix / 2d array problems that require you to do some
kind of search in the matrix. 

That is, given a 2d array `grid` of size `MxN` and starting at some index `grid[i][j]`, you need to 
make a decision how to move forward based on the adjacent neighbors of `grid[i][j]`. 

Sometimes you're only interested in the 4 adjacent neighbors: top, right, bottom, left:

```python
|    |  top |     |
|left|   x  |right|
|    |bottom|     |
```

Other times, you might be asked to check **all** 8 adjacent neighbours, including the diagonal ones left
out above.

Some Leetcode examples would be:

* [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/)
* [200. Number of Islands](https://leetcode.com/problems/number-of-islands/)
* [1091. Shortest Path in Binary Matrix](https://leetcode.com/problems/shortest-path-in-binary-matrix/)

[1091. Shortest Path in Binary Matrix](https://leetcode.com/problems/shortest-path-in-binary-matrix/) 
is an an example where you need to visit all 8 neighbors.

# How to do it: the naive way?

When I started with these kind of problems, I had some naive approach where for each index, I computed its
neighbors, e.g. in the 4-neighbor case:

```pythhon
top, right, bottom, left = row - 1, col + 1, row + 1, col - 1
```

Since you need to make sure you don't pass the boundaries of the grid, I simply added 
four if-clauses:

```python
top, right, bottom, left = row - 1, col + 1, row + 1, col - 1
        
if top >= 0 and grid[top][col] == `some-val`:
        # do something

if right < self.n and grid[row][right] == `some-val`:
        # do something

if bottom < self.m and grid[bottom][col] == `some-val`:
        # do something
    
if left >= 0 and grid[row][left] == `some-val`:
        # do something
```
Needless to say this is hard to read and increases the likelihood of a bug because it's easy to make a
mistake when indexing into the grid in each `if`-clause.

# How to do it: the concise way

There's a very neat way to do this in an iterative way. 

1. Define your directions, e.g.: `[(-1, 0), (0, 1), (1, 0), (0, -1)]`
2. Iterate over each direction and compute the new neighbor via: `neighbor_row, neighbor_col = row + d[0], col + d[1]`
3. use a single `if`-clause to check your boundaries: `if ROWS > neighbor_row >= 0 and COLS > neighbor_col >= 0:`

Glueing this together, an example could look like so:

```Python
ROWS, COLS = len(grid), len(grid[0])
# some more code ...

# top, right, bottom, left
directions = [(-1, 0), (0, 1), (1, 0), (0, -1)]

while queue:
    row, col = queue.popleft()
    for d in directions:
        neighbor_row, neighbor_col = row + d[0], col + d[1]
        if ROWS > neighbor_row >= 0 and COLS > neighbor_col >= 0:
            # do something, e.g. BFS ...
```

The improvement in readability is huge, isn't it? And the `if`-clause looks also trivial compared to the 
ones above.

# Bonus: what about all 8 directions?

You have to change only a single line: 
```Python 
# order:
# top, top right, right, bottom right, bottom, bottom left, left, top left
directions = [(-1, 0), (-1, 1), (0, 1), (1,1), (1, 0), (1,-1), (0, -1), (-1,-1)]
```

That's it. 