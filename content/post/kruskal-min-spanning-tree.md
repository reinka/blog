+++ author = "Andrei Pöhlmann"
title = "Graph algorithms: sorting edges + union-find = Kruskal's minimum spanning tree"
date = "2021-09-30"
description = "Minimum spanning tree"
tags = [
"leetcode",
"Python",
]
categories = [
"algorithms",
"graphs",
]
+++

This post presents a Python 3 implementation of Kruskal's Minimum Spanning Tree algorithm.

# What is a minimum spanning tree?

According to Wikipedia [[1]](https://en.wikipedia.org/wiki/Minimum_spanning_tree):

>A minimum spanning tree (MST) or minimum weight spanning tree is a subset of the edges of a [connected](https://en.wikipedia.org/wiki/Connected_graph), 
> edge-weighted undirected graph that connects all the vertices together, without any cycles and with 
> the minimum possible total edge weight. That is, it is a [spanning tree](https://en.wikipedia.org/wiki/Spanning_tree) 
> whose sum of edge weights is 
> as small as possible. More generally, any edge-weighted undirected graph (not necessarily connected) 
> has a minimum spanning forest, which is a union of the minimum spanning trees for its [connected 
> components](https://en.wikipedia.org/wiki/Component_(graph_theory)).

## What is it good for?

Standard applications are network designs: given a list of locations (e.g. offices), you want to connect them with 
leased lines (e.g. internet/phone lines) which are charged different amounts of money to connect different pairs of
locations. Find the set of lines that connects all locations and that minimizes the total cost.

The solution is the minimum spanning tree, because -- [by definiton of trees](https://en.wikipedia.org/wiki/Tree_(graph_theory)) --
if the network is not a tree you can always remove some edges and reduce the cost.

# How does it work? 

One commonly used algorithm is [Kruskal's algorithm](https://en.wikipedia.org/wiki/Kruskal%27s_algorithm). In a nutshell, 
it sorts the edges in ascending order of their weights and applies Union-Find in a second step to pick the smallest edge
that does not lead to a cycle:

1. Sort all the edges in ascending order of their weight.
2. Pick the (next) smallest edge.
3. Use Union-Find to check if it forms a cycle with the spanning tree formed so far. If so, discard it.
   Else add it to the tree.
3. Repeat step 2. & 3. until there are `V-1` edges in the spanning tree, where `V` corresponds to the number of nodes (also called *vertices*) 
   in the graph.

If you're not familiar with Union-Find you can check this earlier post: [GRAPH ALGORITHMS: UNION-FIND](https://andrei.poehlmann.dev/post/union-find/)

# Runtime Complexity

Let `E` denote the number of edges and `V` the number of vertices in the graph. Sorting the edges by weight using a comparison sort 
runs in `O(E log(E))` time. This allows the step to pick the next smallest edge to operate in constant time `O(1)`.

Regarding the union-find operations: in the worst-case we must iterate through all E edges and run two `find` and one `union` operation.
The worst-case runtime for the `union` and `find` operations is `O(log(V))` for each (see **Time complexity** 
[in this article](https://andrei.poehlmann.dev/post/union-find/)). So step 3. of the algorithm boils down to `O(E log(V))`.

So summing up, in total this would lead to 

```Python 
O(E log(E)) + O(E log(V))
``` 

However, note that the number of edges `E` is at most
`V*V` (i.e. a graph where each vertex is connected with all the other vertices), so  

```Python
E <= V^2
```

Together with the logarithm this leads to

```Python 
log(E) <= 2*log(V).
```  
Applying this to our runtime complexity we get 

```Python
O(E log(E)) + O(E log(V)) <= O(E * 2*log(V)) + O(E log(V)) 
                           = O(3 * E *log(V))
``` 

And because constants can be ignored in the big-O notation this can be further simplified to: 

```Python
O(E log(V))
```


# Code

Let's look at an example, Leetcode's [1584. Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/):

> You are given an array points representing integer coordinates of some points on a 2D-plane, where points `[i] = [xi, yi]`.
> 
> The cost of connecting two points `[xi, yi]` and `[xj, yj]` is the manhattan distance between them: `|xi - xj| + |yi - yj|`, where `|val|`denotes 
> the absolute value of `val`.
>
> Return the minimum cost to make all points connected. All points are connected if there is exactly one simple path between any two points.
>
> Example:
>
> Input: `points = [[0,0],[2,2],[3,10],[5,2],[7,0]]`
>
> Output: `20`
```python3

class Solution:
    def minCostConnectPoints(self, points: List[List[int]]) -> int:
        n = len(points)
        self.roots = [i for i in range(n)]
        self.ranks = [1 for _ in range(n)]
        
        # construct an adjacency map where each key
        # represents and edge and its value the corresponding cost
        costs = self.computeCosts(points)
        # sort edges in ascending order of their cost
        sorted_costs = sorted(costs, key=costs.get)
        
        unions = 0
        mincost = 0
        # for each edge
        for node1, node2 in sorted_costs:
            # get its cost
            curr_cost = costs[(node1, node2)]
            
            # add it to the tree if it will not
            # lead to a cycle and update the mincost
            if self.union(node1, node2):
                unions += 1
                mincost += curr_cost
                
                # stop the union once we added n-1 edges
                if unions == n - 1:
                    break
                    
        return mincost
                    
        
        
    def find(self, node):
        res = node
        while res != self.roots[res]:
            self.roots[res] = self.roots[self.roots[res]]
            res = self.roots[res]
            
        return res
    
    def union(self, node1, node2):
        root1 = self.find(node1)
        root2 = self.find(node2)
        
        if root1 != root2:
            if self.ranks[root1] > self.ranks[root2]:
                self.roots[root2] = root1
                self.ranks[root1] += self.ranks[root2]
            else:
                self.roots[root1] = root2
                self.ranks[root2] += self.ranks[root1]
                
            return True
        
        return False
        
        
    def computeCosts(self, points):
        n = len(points)
        # costs represents an adjecency map 
        # where each key represents an edge
        # between two points and the value of that key
        # to its corresponding cost
        # This will be basically our graph
        costs = {}  
        for i, point1 in enumerate(points):
            for j, point2 in enumerate(points[i+1:], 1):
                cost = abs(point1[0] - point2[0]) + abs(point1[1] - point2[1])
                # add the connection to the adjacency map 
                costs[(i, i+j)] = cost
                
        return costs
```

# References 

* [1] [Wikipedia: Minimum spanning tree](https://en.wikipedia.org/wiki/Minimum_spanning_tree)
* [2] [https://www.ics.uci.edu/~eppstein/161/960206.html](https://www.ics.uci.edu/~eppstein/161/960206.html)
* [3] [https://www.geeksforgeeks.org/applications-of-minimum-spanning-tree/](https://www.geeksforgeeks.org/applications-of-minimum-spanning-tree/)