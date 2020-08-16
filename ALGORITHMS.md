
# Top-k densest overlapping subgraphs  
  
This document provides an overview of the algorithms used to implement the "Top-k overlapping densest subgraphs" method introduced in the homonym paper by Esther Galbrun, Aristides Gionis and Nikolaj Tatti.  
  
## General overview  
  
The goal of this method is to iteratively extract the densest subgraph from a graph, allowing overlaps.   
  
### Naming conventions  
  
$G = (V,E)$ is the original graph
$V$ is the set of nodes in $G$  
$E$ is the set of edges in $G$  
$n = |V|$ is the number of nodes in $G$  
$W$ is the set of all already extracted subgraphs. At the end of the execution, this will be the result of the algorithm
$\lambda$ is a parameter $>0$ for this algorithm, chosen by the caller

### Definitions
The adjusted degree of a node $v \in A$, where $A$ is a subgraph of $G$ is
$$adjustedDegree(v,A) = deg(v; A) - 4\lambda\sum_{w \in W \mid v \in w}\frac{|A \cap w|}{|w|}$$

The density of a graph $U = (V_u, E_u)$ is
$$dens(U) = \frac{|E_u|}{|V_u|}$$

The distance $D$ between two graph $X=(V_X, E)$ and $Y=(V_Y,E)$ is
$$D(X,Y) = 2 - \frac{|V_X \cap V_Y| ^ 2}{|V_X||V_Y|} \quad \text{if} \ V_X \neq V_Y,\quad 0\ \text{otherwise}$$

The marginal gain of a graph $U$ is
$$marginalGain(U) = \frac{1}{2}dens(U)+\lambda\sum_{w \in W}D(U,W)$$

### Extraction procedure
The procedure to extract the subgraph from the original graph $G$ is the following:  
1. $V_n = V$  
2. For $i = n,...,2$:  
   a. Find the node $v \in V_i$ that minimizes $adjustedDegree(v,V_i)$
   b. $V_{i-1} = V_i \setminus{\{v\}}$  
3. Among $V_n, .., V_2$, the subraph to extract is the one that maximizes the marginal gain

It is possible to repeat these steps until the necessary number of subgraphs are extracted.

## Algorithms
Each step of the procedure declared in the previous section requires particular care in order to be done efficiently. 

### Removing nodes from the subgraph
In the step `2.b` we need to declare a new subgraph, removing a node from the "previous" one.
After considering these observations, it is possible to execute this step in `O(1)` time:
- It can be shown that it is not necessary to keep all the $V_n,...,V_2$ graphs, but it is possible to keep only one working copy
- The modification only requires to remove a node
- The subgraph is initially equal to the whole graph
- It is possible to assign an index $0,...,n-1$ to each node in the graph 

The subgraph can be represented with a boolean array `node_mask` that represents whether a node is in the graph or not. 
```
class Subgraph {
	nodeMask = BooleanArray(n, true)

	fun remove(node) {
		nodeMask[node.index] = false
	}
}
```
Then, the step $V_{i-1} = V_i \setminus{\{v\}}$  can become:
```
subgraph.remove(v)
```


### Choosing the subgraph that maximises the marginal gain
In the step `3`, the subgraph that maximizes the marginal gain must be selected.
It has been implemented by keeping track of the marginal gain while the nodes are extracted, by doing this after each extraction: 
```
marginalGain = subgraph.marginalGain()
if(marginalGain > maxMarginalGain){
	maxMarginalGain = marginalGain 
	maxSubgraph = subgraph.copy()
}
```
This will cost $O(|W|)$ time assuming that `subgraph.marginalGain()` takes $O(|W|)$ time and `subgraph.copy()` takes $O(1)$ time.

#### Copying the subgraph in $O(1)$
In order to copy the graph in $O(1)$ it's necessary to change the representation of the subgraph. In particular, it is necessary to keep track of node removals:
```
class Subgraph {
	nodeMask = BooleanArray(n, true)
	removals = List<Nodes>()

	fun remove(node) {
		removals.add(node)
		nodeMask[node.index] = false
	}
}
``` 
By observing that the only operation that changes the subgraph is the removal of a node, it is possible to copy the graph by simply saving the number of removals happened so far.

It must be pointed out that the copied subgraph can't provide any functionality in a computationally efficient way, but that's not needed during the iteration. After that, it has to be converted to a different representation.
That costs $O(|V|)$, but only needs to be performed once. This is done by reapplying the list of removals to the initial graph $G$.

#### Computing the marginal gain in $O(|W|)$
In order to compute the marginal gain in $O(|W|)$ time, it's necessary to use some tricks that reuse as much information as possible during the extraction of nodes.
To compute the marginal gain of `subgraph`, the following information are required:
1. The number of nodes in `subgraph`
2. The number of edges in `subgraph`
3. The size of the intersection between `subgraph` and every other already extracted subgraph.

By remembering that the only change of `subgraph` is the removal of a node, and that `subgraph` is initially the whole original graph, it's easy to show that we can keep track of information `1` and `2` in $O(1)$ time, and `3` in $O(|W|)$ time:
```
class Subgraph {
	nodeMask = BooleanArray(n, true)
	removals = List<Nodes>()
	nodesCount = G.nodesCount
	edgesCount = G.edgesCount
	intersections = W.asssociateWith { it.size } // The size of the intersection is initially just the size of the other subgraphs, since this is initially the whole original graph

	fun remove(node) {
		removals.add(node)
		nodeMask[node.index] = false
		nodesCount -= 1
		edgesCount -= node.degree
		for(w in W){
			if(node in w){
				intersections[w] -= 1 // If the node was in w, the intersection with w is getting smaller
			}
		}
	}
}
```
With this information, it's easy to show that the marginal gain can indeed be computed in $O(|W|)$ time.

### Extracting the node
Following the step `2.a` of the main procedure, it is required to extract the node that minimizes the adjusted degree.
In order to do so efficiently, it's necessary to rethink the adjusted degree as the differece of two quantities: 
$$adjustedDegree(v,A) = deg(v; A) - 4\lambda\sum_{\{w \in W \mid v \in w\}}\frac{|A \cap w|}{|w|} = deg(v; A) - f(\{w \in W \mid v \in w\})$$

#### Computing $f(\{w \in W \mid v \in w\})$
It can be observed that the quantity $\{w \in W \mid v \in w\}$ will be the same for a lot of nodes. 

That said, the set of nodes $V$ can partitioned according to the quantity $\{w \in W \mid v \in w\}$.  Let's call the partitions $P$, and the number of partitions $|P|$.  
For each partiton, $f(\{w \in W \mid v \in w\}) = 4\lambda\sum_{\{w \in W \mid v \in w\}}\frac{|A \cap w|}{|w|}$ has to be computed.  
It can be shown that keeping track of this quantity costs $O(|W|)$ by doing something similar to what has been done for the marginal gain.

It's important to point out that the number of partition $|P|$ can grow up to $min(|V|, 2^{|W|})$, however in practice this number will be much smaller, as shown in the original paper. 

#### Extracting the node
Let's momentarely forget about $f(\{w \in W \mid v \in w\})$ by assuming that it is equal among all nodes.  
Than extracting the node with the minimum adjusted degree will be equivalent to extracting the node with the minimum degree.  
To do so, we can have all the nodes in a min heap. This however won't be efficient, since each extraction will take $O(|V|)$ time.

By observing that many nodes will share the same degree, we can introduce the concept of a "bucket" as a set of nodes that share the same degree.  
The heap can now be a priority queue of buckets.
Extracting a node will take only $O(1)$[^1], since it's only a matter of removing a node from the bucket at the head of the queue.  
Changing the degree of a node will take $O(1)$[^1] as well, since it's only a matter of moving the node from a bucket to anoter.  
[^1]: When a bucket will become empty, or a new bucket has to be created, some additional work has to be done. In these cases, the complexity will increase to $O(|B|)$, where $B$ is the set of all buckets. In practice this happens so infrequently that this can be ignored. Further work needs to be done to formalize this claim.  

It's possible to further optimize this by observing that nodes with an high degree will never be extracted. By keeping those nodes in a separate bucket that is outside of the heap, it's possible to speed up the system even more. Right now this "high degree" is hardcoded to `100`. Further work is necessary to choose the threshold dinamically.

---
Let's remember that this structure only works for nodes with the same $f(\{w \in W \mid v \in w\})$. It's in fact necessary to replicate this structure for each partition of $V$, as in the previous section. 
To extract the node with the minimum adjusted degree it's necessary to look at the node with the minimum degree for each partition, and choose the minimum among those. 

## Final Complexity
This table summarises the complexity of each operation discussed so far
|Operation|Cost|Number of calls |
| - | - | - |
| Initialization 													| $O(\vert V \vert + \vert W \vert)$ | $1$
| Keeping the marginal gain updated 								| $O(\vert W \vert)$ | $O(\vert V \vert)$
| Copying the subgraph												| $O(1)$ | $O(\vert V \vert)$
| Converting the copyed subgraph to a standard representation		| $O(\vert V \vert)$ | $1$
| Extracting the node with minimum adjusted degree					| $O(\vert P \vert)$ | $O(\vert V \vert)$
| Updating the degrees of a neighbor of the extracted node			| $O(1)$ | $O(\vert E \vert)$
| Keeping the adjusted degree updated for each partition			| $O(\vert P \vert * \vert W \vert)$ | $O(\vert V \vert)$

The final complexity for extracting a subgraph is $O(|P|*|W|*|V| + |E|)$. 
This means that for extracting $k$ subgraphs, the total complexity is $O(k^2*|P|*|V| + k|E|)$
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYyNTY0NjIwMCwzMzg2MzQzNjZdfQ==
-->