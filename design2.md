# In this writeup I will go over all the major design problems that I encountered, and I will explain the decision that was made.

I've thought a lot about the best way to describe the design of this data structure, and really, the design I ended up with is the result of making a decision at important crossroads. So what better way to talk about the design than to talk about all the crossroads and the path I decided to go down.


# Tree space partitioning vs grid 

I wanted to make a collision system that could be used in the general case, and did not need to be fine-tuned. Grid based collision systems suffer from the teapot-in-a-stadium problem. They also degenerate more rapidly as objects get more clumped up. If, however, you have a system where you have strict rules with how evenly distributed objects will be among the entire space you're checking collisions against, then a grid system can be better. But I think these systems are few and far in between. I think in most systems, for example, its perfectly possible for all the objects to exist entirely on one half of the space being collision checked leaving the other half empty. In such a case, half of the data structure of the grid system is not being used to partition anything. A tree space partitioning system, on the other hand, would be able to exploit that.


# (Sweep and Prune) vs (Kd Tree) vs (KdTree + Sweep and Prune)

Sweep and prune is a simple AABB collision finding system, but it degenerates as there are more and more "false-positives" (objects that intersect on one axis, but not both). Kd Trees are great, but objects that can't be inserted into children, but be left in the parent node, and those objects must be collision checked with everybody else naivley. The best solution is to use both. #TODO talk more

# KD tree vs Quad Tree

The cost of building a kd tree is often overstated. People say use a quad tree for dynamic systems, and a kd tree for static systems. But in reality, the gains you get from building a kd tree, offset the added cost of building it- even in dynamic systems. In systems where you know there cannot be that many collisions maybe, but in most systems it is entirely possible for very very dense clumps. In those cases, you want all the help you can get, in which case you will want the tree that has partitioned the space the best.

KD trees are also great in a multithreaded setting. With a kd tree, you are guarenteed that for any parent, there are an equal number of objects if you recurse the left side and the right side since you specifically chose the divider to be the median. This means that both the left and right are jobs of equal side and can be handled in parallel.  With a quad tree you don't have this property since the dividers chosen were all chose statically.

# Level of indirection vs none for elements in the tree.

It is important to note that what we are trying to speed up is the collision finding. This is the potentially n^2 computation. It is worth it shifting everything around in memory if it allows us to remove one level of indirection inside of the computationally expensive n^2 computation. The main benefit that comes from removing the level of indirection is cache coherency. If we just have pointers to all the objects in the tree, Then for every collision check we do against two objects, its possible that is a cache miss. This becomes a problem for large n.


# Copy vs NoCopy

This is the classic tradeoff: Memory vs Computation. Because we are not using a level of indirection, we have to reordering objects to match how they 
show up in the tree. You can reorder an array of objects in place, but it is a fairly expensive operation. You also have to do this twice (once to order them into the tree, once to order them back to the original). A less compuationally expensive way it to just use an auxilary array, but this requires twice the amount of memory. So in certain cases, you might not have the space for an auxiliary array if we are talking millions of objects.

Here, I left the api flexible enough to use either option, we Copy being the suggested api.


# Exploiting Temporal Locality vs Not.

If you are simulating moving elements, it might seem slow to rebuild the tree every iteration. But from benching, most of the time querying is the cause of the slowdown. Rebuilding is always a constant load, but the load of the query can very wildly depending on how many elements are overlapping.

Rebuilding the first level of the tree does take some time, but it is still just a fraction of the entire building algorithm in most cases. 

The main reason against it is that adding any kind of "memory" to the tree where you save the positins of the dividers to use as good heuristic positions for next iterations will come at a cost of a possibly sub optimal tree layout which will hurt the query algorithm. Our goal is to make the query algorithm as fast as possible since that is what dominates.


# Tree structure data seperate from elements in memory, vs intertwined
There is a certain appeal to storing the tree elements and tree data in the same piece of contiguous memory. Cache coherency is improved since there is very little change that the tree data, and the elements that belong to that node are not in the same cache block. But there is added complexity. Because the elements that are inserted into the tree are generic, the alignment of the objects may not match that of the tree data. This would lead to wasted padded space that must be inserted into the tree to accomodate this. Additionally, while this data structure would be faster at querying, it would be slower if the user just wants to iterate through all the elements in the tree. The cache misses doesnt happen that often since the change of a cache miss only happens on a per node basis, instead of per element. Moreover, I think the tree data structue itself is very small and has a good chance of being completely in a cache block.

# Leaves as unique type, vs single node type.
The leaf elements of the tree data structure don't need as much information as their parent nodes. They don't have dividers. So half of the nodes of the data structure has some fields that store no information. It can be tempting to give the leaf nodes a seperate type to improve memory usage, but the divider size is small, and the number of nodes in general is relaively small compared to the number of bots. Alignment can also be an issue. 


# Dfs in-order vs Dfs pre-order vs Bfs order.

Dfs in order uses more memory during construction from during recursion, but it gives you a memory layout that more matches how it is logically. Dfs order has the nice property of the following: If a node is to the left of another node in space, then that node is to the left of that node in memmory. 

In reality, it doesnt really matter because all this does is reduce cache misses when handling collision between two nodes which doesnt happen as often as an object to object basis. 

# AABB vs Point + radius

Point+radius pros:
less memory (just 3 floating point values)
cons:
can only represent a circle (not an oval)
have to do more floating point calculations durying querying

AABB pros:
no floating point calculations needed during querying.
can represent any rectangle
cons:
more memory (4 floating point values)

Note, if the size of the elements is the same then the Point+radius only takes up 2 floating point values, so that might be better in certain cases. But even in this case, I think the cost of having to do floating point calculations when comparing every bot with every other bot in the query part of the algorithm is too much.


# Performance Themes

Now lets go over some things to note about the design that wern't exactly making a decisive choise about go down one path versus another.

# Only pick ODD height trees.

In order to assure that the leaf nodes all map to a relatively square piece of space, we force the tree to always have odd height.

# Multithreading

# Memory Locality

# Knowing the axis at compile time




#Other ramblings/ Questions

# Liquid vs Particle

Simulating liquid is 

