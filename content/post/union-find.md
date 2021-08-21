+++ author = "Andrei Pöhlmann"
title = "Union-find"
date = "2021-08-21"
description = "Python 3 implementation of the union-find algorithm."
tags = [
"leetcode"
"Python",
]
categories = [
"algorithms"
]
+++

This post presents a Python 3 implementation of the union-find algorithm with rank and path compression optimization.


# What is it good for?

Given a set of objects, of which none, some or all are somehow connected with each other, it helps you to find all its disjoint subsets. 
In graph problems this might look like finding the number of [components](https://www.wikiwand.com/en/Component_(graph_theory\))
in a graph. Or determining if two nodes are connected.

# How does it work?

It consists of two main operations:

* find
* and union.

We are first going to look at a naive implementation of these to understand their concepts. Later we will look at their 
optimizations.

## Find

For a ginve node `n`, `find(n)` returns its `n`'s root node, or "root parent" so to say. That is, it walks up all of `n`'s 
ancestors until it arrives at its root parent. How do we know it's a root parent? If the parent of the node is the node itself.

### Example

Let's assume we have `k` nodes and a list called `roots` which for each node `n` contains the parent of that node.
More specifically, it stores the parent of node `n` at index `n`: so the current parent of `n` would be `roots[n]`. 

Let's consider the following example with 2 components and `k=6`:
```python
    0           4
  /   \         |
 1     2        5
        \
         3
```
And let's assume `roots = [0, 0, 0, 2, 4, 4]`: each index holds the parent of the corresponding node. So in this example,
for node `n=3`, `find` would climb up the whole tree until it hits `0`.

### Implementation
A Python implementation of `find` can look like so:

```python
def find(self, node):
    parent = node
    
    # loop through all the parents
    # until we arrive at the root parent:
    # i.e. when the parent of the node is
    # identical to the node itself
    while parent != self.roots[parent]:
        parent = self.roots[parent]

    return parent
```
Continuing our example, the concrete steps would be `3 -> 2 -> 0`, since the while condition holds only at `0` because of `0 == roots[0]`.


## Union

As the name suggests, given an edge between two nodes, `union(node1, node2)` unions them. 

A naive union would be to compare 
`node1`'s and `node2`'s parent and if these 2 parents are not identical, we just update the parent of `node2` to be `node1`.

We will first look at this naive approach. Later we will look at another approach, where we set the parent of `node2` to be 
the (**root**-)parent of `node1`.

### Example
Suppose we are given a list 
`edges = [[2,3], [0,1], [0,2], [4,5]]`, we can iterate through the list and execute `union(...)` for each item.

We start off with `k=6` individual nodes

```python
0 1 2 3 4 5
```
and start to union according to `edges`:

* First we `union(2,3)` leading to 

```python
2
|
3
```

* followed by `union(0,1)`

```python
0     2
|     |
1     3
```

* followed by `union(0,2)`
```python
    0           
  /   \         
 1     2        
        \
         3
```

* and finally `union(4, 5)` leads to the graph we saw in the `find` section:

```python
    0           4
  /   \         |
 1     2        5
        \
         3
```

### Implementation

An implementation could look like so:

```python
def union(self, node1, node2):
        parent1 = self.find(node1)
        parent2 = self.find(node2)
        
        if parent1 != parent2:
            self.roots[parent2] = parent1
```

Remember we start off with `k=6` individual nodes at first. Unlike in our find example, we don't assume anymore that the 
`roots` list is given. Instead, we start with a `roots` list where each parent is initialized
to the node itself, i.e. `roots = [0, 1, 2, 3, 4, 5]` 

So running `union(2,3)` for our first edge in `edges`:
* `parent1 = self.find(2)` results in  `parent1 == 2`
* `parent2 = self.find(3)` results in  `parent2 == 3`
* Because `parent1 != parent2`, we update `self.roots[parent2] = parent1`, that is: `self.roots[2] = 1` and we end up with
`roots = [0, 1, 1, 3, 4, 5]`

We repeat this for every edge, i.e.

```python
for edge in edges:
    node1, node2 = edge  # edge is list with 2 items
    self.union(node1, node2)
```

# Optimization

By now we should have a basic understanding of how union-find constructs the components of a graph given a list of its edges. 

Now the above implementations are somewhat inefficient. Let's analyze why.

## Optimizing `find` via path compression

Imagine we have a graph like the following:

```python
0
|
1
0
|
2
|
3
|
4
| 
5
```

The way our find algorithm is implemented it would traverse the graph every time all-over again even if we already visited all the
parents once. 

For instance, suppose you do `find(5)`. This would result in the traversal of `5 -> 4 -> ... -> 0`. 

Now let's say we do `find(4)`. This would traverse again the whole graph starting at `4 -> ... -> 0`. 

One way to avoid this is to update all the parents during the first `find(5)` via recursion:

```python
def find(self, node):
    if node != self.roots[node]:
        self.roots[node] = self.find(self.roots[node])
        
    return self.roots[node]
```

This is called path compression because we are effectively compressing the graph. I.e. the above graph would be compressed to:

```python
        0
 /   /  |  \   \
1   2   3   4   5
```

