---
layout: post
title:  "Properties of Dijkstra's Algorithm"
date:   2020-10-26 18:24:26 +0800
categories: algorithm
---

If you look around the internet, or peek at books on introductory algorithms, trying to find a proof of dijkstra's algorithm, you'll probably find something like this:  
[https://math.stackexchange.com/questions/835871/dijkstra-algorithm-proof](https://math.stackexchange.com/questions/835871/dijkstra-algorithm-proof)

![Dijkstra proof?]({{ "/assets/dijkstra-proof.png" | absolute_url }})

However this proof is too succinct, in that it does not capture the gist of the values we update for each nodes in every loop. In its raw shape, it is also based an implementation that keeps all unvisited nodes in a large heap, which begs the question why the most common min-heap based implementation is correct  

I'll base my proof on the heap based implementation

suppose we have a graph G(V, E), with nonnegative edge length, starting from s in V, we want to find the distance(shortest path) from s to any node in V

we'll maintain two data structures:  
a set of nodes whose distance from s has been found, call it FOUND  
a min-heap, call it HEAP, of nodes that are at the frontier of our search, the value associated with nodes in this heap is the "temporary distance", we denote it by d[x] for node x, which we will prove to be the distance of a partial graph  
for convinience we'll call the set of nodes not in FOUND/HEAP OTHERS 

the algorithm is follows:  
  * create empty set FOUND, and min heap HEAP
  * add s to HEAP, with d[s] := 0
  * while s is not empty, pull min value x out of it, add x to FOUND, and for 
  each of x's neighbor, let's say y, 
    * if y is in OTHERS, add y to HEAP with value d[y] = d[x] + len(x,y)
    * if y is in HEAP and if d[x] + len(x,y) < d[y], then update d[y] to this smaller value



we'll prove 3 invariants that are rather intuitive once they are spilled out

claim: before each iteration of the while loop, we maintain:  
  1. frontier property: for any node x in FOUND, and any node y not in FOUND, any path(with no loop) from x to y has to go through some node in HEAP
  2. partial shortest path property: for each node x in HEAP, the value d[x] associated with x is length the shortest path from s to x, that only passes through nodes in FOUND  
  3. shortest path property: when node x is put into FOUND, the value d[x] is the distance from s to x

we'll prove them of course by induction, on iterations of the while loop

base case:   
claim 1: FOUND is empty so this is automatically true

claim 2: only node in HEAP is s, d[s] = 0 is the length of shortest path, since we require every edge to have nonnegative length

claim 3: we remove s and add it to FOUND with d[s] = 0

introductory step:

claim 1: suppose after we pull out x from HEAP and add it to FOUND, there exists some path from u in FOUND to v in OTHERS that does not go past HEAP, WLOG suppose u~v has the smallest # of edges  
if u is not x, it was previously in FOUND, then u~v has to go pass some node y in HEAP, if y is not x, then y is still in HEAP, contradiction   
if y is x, then x~v has smaller # of edges, x is now in FOUND, contradiction  
so u has to be x, consider the next node in path x~v, call it next  
if it were in FOUND, then it still is, and next~v is a smaller path. if it were not in FOUND, then it was either in HEAP, or is added to HEAP after the operation, both are contradictions.

claim 2:  
This part is actually similar to the original proof  
we prove with 2 parts  
1, for x in HEAP, d[x] >= partial dist(s, x). this is trivial since d[x] has always been the length of some partial path  
2, for x in HEAP, d[x] <= partial dist(s, x).  
after we pull out x from HEAP and add it to FOUND, examine y in HEAP:  
denote a~b be a shortest path from a to b in FOUND, and let right arrow -> denote a direct edge between any two nodes，
in the partial shortest path from s to y, let u be the predecessor of y, then the part of the path from s to u must be the shortest path of s~u，so partial dist(s, y) = d[u] + len(u,y)  
When u was added to FOUND, d[y] was updated to be <= d[u] + len(u,y) = partial dist(s, y), and d[y] would not ever increase to a larger value, so d[y] <= partial dist(s, y) maintains


claim 3：
when we pull out x from HEAP, we claim that the partial shortest path from s to x is the shortest path in the whole graph  
By claim 1，the shortest path，has to go through some node y in HEAP first，and the partial shortest path to y is no smaller than that of x because x is the min node of the HEAP, adding some potential additional distance from y to x if y is not x, the total length of path is no smaller than the partial shortest path from s to x
