+++ author = "Andrei Pöhlmann"
title = "Graph Algorithms: Union-find"
date = "2021-08-21"
description = "Python 3 implementation of the union-find algorithm with path splitting and union by size"
tags = [
"leetcode",
"python",
]
categories = [
"algorithms",
"graphs"
]
+++

This post presents a Python 3 implementation of the [union-find algorithm](https://en.wikipedia.org/wiki/Disjoint-set_data_structure) with
different optimization techniques: 

1. path compression and union by rank
2. path splitting and union by size.


# What is it good for?

Given a set of objects, of which none, some or all are somehow connected with each other, it helps you to find all its disjoint subsets. 
In graph problems this might look like 

* finding the number of [components](https://www.wikiwand.com/en/Component_(graph_theory))
in a graph, 
* determining if two nodes are connected, 
* or spotting a cycle.

At the bottom of this article there is a list of Leetcode problems asking to solve these kind of problems.

# How does it work?

It consists of two main operations:

* `find`
* and `union`.

We are first going to look at a naive implementation of these to understand the basic concepts. Later we will 
look at some optimizations.

## Find

For a ginve node `n`, `find(n)` returns its `n`'s root node, or "root parent" so to say. That is, it walks up all of `n`'s 
ancestors until it arrives at its root parent. How do we know it's a root parent? If the parent of the node is the node itself.

#### Example

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
And let's assume `roots = [0, 0, 0, 2, 4, 4]`: each index holds the parent of the corresponding node. So in this example
for node `n=3`, `find(3)` would climb up the whole tree until it hits `0`.

#### Implementation
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
Continuing our example, the concrete steps would be `3 -> 2 -> 0`, since the while condition holds at `0` because of `0 == roots[0]`.


## Union

As the name suggests, given an edge between two nodes, `union(node1, node2)` unions them. 

A naive union would be to compare 
`node1`'s and `node2`'s parent and if these 2 parents are not identical, we just update the parent of `node2` to be `node1`.

We will first look at this naive approach. Later we will look at another approach, where we set the parent of `node2` to be 
the (**root**-)parent of `node1`.

#### Example
Suppose we are given a list 
`edges = [[2,3], [0,1], [0,2], [4,5]]`, we can iterate through the list and execute `union(...)` for each item.

We start off with `k=6` individual nodes

```python
0 1 2 3 4 5
```
and start to union according to the `edges`:

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

#### Implementation

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
    node1, node2 = edge  # edge is list with 2 entries
    self.union(node1, node2)
```

# Can we do better?

By now we should have a basic understanding of how union-find constructs the components of a graph given a list of its edges. 

However, the above implementations are somewhat inefficient. Let's analyze why.

## Optimizing `find` 

#### Path compression

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

This is called path compression because we are effectively compressing the graph. I.e. the above graph would be compressed 
to a graph that basically has only 2 levels -- the root level and an additional level with all its children:

```python
        0
 /   /  |  \   \
1   2   3   4   5
```

#### Path splitting

An alternative to the recursive path compression is the so called **path splitting**. It's very similar 
to our first naive approach, we only add one more line during our while loop, where we update the parent
of the node to its grandparent:

```python
def find(self, node):
    parent = node
    
    while parent != self.roots[parent]:
         # first update current node's parent with its grand-parent
        self.roots[parent] = self.roots[self.roots[parent]]  # this is new
        
        # then walk up the ancestors
        parent = self.roots[parent]

    return parent

```


## Optimizing `union`

In our naive algorithm, we arbitrarily merged `node2` into `node1`. It turns out there's a better way to decide on
how to do the merging: merge the shorter tree into the taller/deeper one. 

The rationale behind this is that we want to avoid to construct deep trees because the depth of the tree 
affects the `find` and `union` time complexity. Technically, [it sets an upper bound of O(log n) on the 
worst case cost of every operation](https://stackoverflow.com/a/41689120).  

To visualize the intuiton behind this, note, that if you merge a shorter tree into a taller one, 
you don't change the height of the tree:

```python
    0           4                       0
  /   \         |                     / | \
 1     2        5        ----->      1  2  4
        \                               |   \
         3                              3    5
```
Here, the max height is still 3. That's not the case the other way round, which would result in a max height of 4:

```python
       4
     /   \
    0     5
  /   \    
 1     2   
        \  
         3 
```


#### Union by rank

For this, we need to somehow keep track of the height/depth of the tree. This is done by a so called **rank**: 
the deeper the tree, the higher its rank. This rank is then used in the `union` function to decide on how 
to merge.

As before, we start off with individual nodes so initially
 the rank of each node is `1`. We will keep the ranks in a list called `ranks = [1 for _ in range(k)]`.
 
 ```python
def union(self, node1, node2):
        root1 = self.find(node1)
        root2 = self.find(node2)
        
        if root1 != root2:
            # if rank of root1 is larger, merge 2 into 1
            if self.ranks[root1] > self.ranks[root2]:
                self.roots[root2] = root1
            elif self.ranks[root1] < self.ranks[root2]:
                self.roots[root1] = root2
            else:
                # equal length trees: doesn't matter how we 
                # merge, however the height will increase by 1
                self.roots[root2] = root1
                self.ranks[root2] += 1
``` 

#### Union by size 

This is an alternative approach, where the size is used to determine the new root instead of the height. 
Analogously, we use a list to store information on the size of each node: the more descendants
the node has (i.e. children), the larger its size. 
This size is then used in the `union` function to decide on how 
to merge.

As before, we start off with individual nodes so initially
 the size of each node is `1`. We will keep the sizes in a list called `sizes = [1 for _ in range(k)]`.
 
 ```python
def union(self, node1, node2):
        root1 = self.find(node1)
        root2 = self.find(node2)
        
        if root1 != root2:
            # if size of root1 is larger, merge 2 into 1
            if self.sizes[root1] > self.sizes[root2]:
                self.roots[root2] = root1
                self.sizes[root1] += self.sizes[root2]  # update size
            elif self.sizes[root1] < self.sizes[root2]:
                self.roots[root1] = root2
                self.sizes[root2] += self.sizes[root1]
```

Note, that this is a somewhat lazy implementation since we don't update the size of the smaller tree because
it is technically not needed: we just want to know which one of the two trees is the larger one.


# Bonus material

## Check for connectivity

Once we unioned all the nodes according to our `edges`, we can easily determin if two nodes are
connected. Two nodes are connected if they have the same root parent:

```python
def is_connected(self, node1, node2):
    return self.find(node1) == self.find(node2)
```

## Cycle detection

There's a little detail in the `union` operation we have ignored so far: the `union` is only done
if the roots of the two nodes are different because if they already are there is nothing more
to do.

Let's look at our last example graph and think about what it actually means if we are given a new 
`edge`, call `union` on it, and `find` the same roots during that operation:


```python
       4
     /   \
    0     5
  /   \    
 1     2   
        \  
         3 
```

Suppose we are given a new `edge = [1,3]`. The find operation will tell us `root1 === root2 == 4`, i.e.
these two nodes are already part of the same tree. If we look at the new visualization of the graph including 
this new edge our graph looks acually like this:


```python
       4
     /   \
    0     5
  /   \    
 1     2   
 |      \  
 ------- 3 
``` 

Note the new connection between node 1 and node 3: The graph actually contains a **cycle**! (And technically
speaking, it is no longer a tree [because a tree is a connected acyclic undirected graph.](https://en.wikipedia.org/wiki/Tree_(graph_theory))) 

So basically whenever `root1 == root2` in the `union` operation, we discovered a cycle.

# Summary: union-find with path splitting and union by size

Summarizing, we get the follwoing union-find algorithm optimized with path splitting and union by size:

```python
class Solution:
    def union_find(self, k, edges):
        # k represents the number of nodes
        self.roots = [i for i in range(k)]
        self.sizes = [1 for _ in range(k)]
        
        for node1, node2 in edges:
            self.union(node1, node2)

    def is_connected(self, node1, node2):
        return self.find(node1) == self.find(node2)
                
    def find(self, node):
        parent = node
        while parent != self.roots[parent]:
            # optimization: do path splitting first
            # i.e. update current node's parent with its grand-parent
            self.roots[parent] = self.roots[self.roots[parent]]
            
            # then walk up the ancestors
            parent = self.roots[parent]
            
        return parent
    
    def union(self, node1, node2):
        root1 = self.find(node1)
        root2 = self.find(node2)
        
        
        if root1 == root2:
            # detected a cycle
            # do something if you need this info
            return
        
        
        # optimization: union by size
        if self.sizes[root1] < self.sizes[root2]:
            # if size of root1 is less than size of root2
            # we want to merge root1 into root2, so we swap
            # the roots first 
            root1, root2 = root2, root1

        self.roots[root2] = root1
        self.sizes[root1] += self.sizes[root2]
        
``` 

This is a less verbose version of our previous code: where we replaced the previous `if-else` with a 
simple swap operation.   

# Time complexity

|                | `find`     | `union`   |
| ---------      | ---------- | --------- |
| **worst case** | O(log k)   | O(log k)  |
| **amortized**  | O(⍺(k))    | O(⍺(k))   |

where `k` denotes the number of nodes and ⍺ the inverse [Ackermann-function](https://en.wikipedia.org/wiki/Ackermann_function),
so practically constant time.

On the why of the worst case, see answers to this quesiton on stackoverflow: 
[Why is the time complexity of performing n union find (union by size) operations O(n log n)?](https://stackoverflow.com/q/53149097)


#### But does the optimization really help?
Without any optimization `find` and `union` can lead to trees with height O(k). In such a situation, 
the `find` and `union` operations require O(k) time [[1].](https://en.wikipedia.org/wiki/Disjoint-set_data_structure#Time_complexity)

The Wiki page referenced in [[1]](https://en.wikipedia.org/wiki/Disjoint-set_data_structure#Time_complexity)
contains a nice elaboration on the time complexity of union-find with different optimization combinations.

# Some Leetcode problems

## Medium
* [207. Course Schedule](https://leetcode.com/problems/course-schedule/)
* [547. Number of Provinces](https://leetcode.com/problems/number-of-provinces/)
* [261. Graph Valid Tree](https://leetcode.com/problems/graph-valid-tree/) (premium)

# References

[1] [Wikipedia: Disjoint-set data structure](https://en.wikipedia.org/wiki/Disjoint-set_data_structure#Time_complexity)