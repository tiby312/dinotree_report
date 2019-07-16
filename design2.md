## For the reader

I've thought a lot about the best way to describe the design of this data structure, and really, the design I ended up with is the result of making a decision at important crossroads. So what better way to talk about the design than to talk about all the crossroads and the path I decided to go down. So in this writeup I will go over all the major design problems that I encountered, and I will explain the decision that was made.

I've written this assuming the reader knows what a [kdtree](https://en.wikipedia.org/wiki/K-d_tree),[sweep and prune](https://en.wikipedia.org/wiki/Sweep_and_prune) ,and [quad tree](https://en.wikipedia.org/wiki/Quadtree) are. 


## Tree space partitioning vs grid 

I wanted to make a collision system that could be used in the general case and did not need to be fine-tuned. Grid based collision systems suffer from the teapot-in-a-stadium problem. They also degenerate more rapidly as objects get more clumped up. If, however, you have a system where you have strict rules with how evenly distributed objects will be among the entire space you're checking collisions against, then I think a grid system can be better. But I think these systems are few and far in between. I think in most systems, for example, its perfectly possible for all the objects to exist entirely on one half of the space being collision checked leaving the other half empty. In such a case, half of the data structure of the grid system is not being used to partition anything. 


## (Sweep and Prune) vs (Kd Tree) vs (KdTree + Sweep and Prune)

Sweep and prune is a simple AABB collision finding system, but it degenerates as there are more and more "false-positives" (objects that intersect on one axis, but not both). Kd Trees are great, but objects that can't be inserted into children are left in the parent node and those objects must be collision checked with everybody else naivley. The tree height might also end up very large to satisfy the requirement that the leaf has only one element. A better solution is to use both. 

The basic idea is that you use a tree up until a specific tree height, and then switch to sweep and prune, and then additionally use sweep and prune for elements stuck in higher up tree nodes. The sweep and prune algorithm is a good candidate to use since it uses very little memory (just a stack that can be reused as you handle decendant nodes). But the real reason why it is good is the fact that the bots that belong to a non-leaf node in a kd tree are likely to be stewn across the divider in a line. Sweep and prune degenerates when the active list that it must maintain has many bots that end up not intersecting. This isnt likely to happen for the bots that belong to a node. The bots that belong to a non leaf node are guarenteed to touch the divider. If the divider partitions bots based off their x value, then the bots that belong to that node will all have x values that are roughly close together (they must intersect divider), but they y values can be vastly different (all the bots will be scattered up and down the dividing line). So when we do sweep and prune, it is important that we sweep and prune along axis that is different from the axis along which the divider is partitioning.


## KD tree vs Quad Tree

The cost of building a kd tree is high. Quad trees are easy to build so its suggested to be better for dynamic systems, and a kd tree good for static systems. But in some systems, the gains you get from building a kd tree, offset the added cost of building it- even in dynamic systems. In systems where you know there cannot be that many collisions a quad-tree or a grid based system could be better, but in most systems it is entirely possible for very very dense clumps. In those cases, you want all the help you can get, in which case you will want the tree that has partitioned the space the best.

KD trees are also great in a multithreaded setting. With a kd tree, you are guarenteed that for any parent, there are an equal number of objects if you recurse the left side and the right side since you specifically chose the divider to be the median. This means that both the left and right are jobs of either side and can be handled in parallel.  With a quad tree you don't have this property since the dividers chosen were all chosen statically.


## Exploiting Temporal Locality vs Not.

If you are simulating moving elements, it might seem slow to rebuild the tree every iteration. But from benching, most of the time querying is the cause of the slowdown. Rebuilding is always a constant load, but the load of the query can vary wildly depending on how many elements are overlapping.

For example, in a bench where inside of the collision call-back function I do a reasonable collision response with 80_000 bots, if there are 0.8 times (or 65_000 ) collisions or more, querying takes longer than rebuilding. For your system, it might be impossible for there to even be 0.8 * n collisions, in which case building the tree will always be the slower part. For many systems, 0.8 * n collisions can happen pretty often. For example if you were to simulate a 2d ball-pit, every ball could be touching 6 other balls (https://en.wikipedia.org/wiki/Circle_packing), and that is without soft-body physics. So in that system, there are 0.9 * n collisions. So in that case, querying is the bottle neck. With liquid or soft-body physics, the number can be every higher. up to n * n.

Rebuilding the first level of the tree does take some time, but it is still just a fraction of the entire building algorithm in most cases, provided that it was able to partition almost all the bots into two planes. 

The main reason against exploiting temporal locality is that adding any kind of "memory" to the tree where you save the positions of the dividers to use as good heuristic positions for next iterations will come at a cost of a possibly sub optimal tree layout which will hurt the query algorithm. Our goal is to make the query algorithm as fast as possible since that is what can dominate.

The construction of the tree may seem expensive, but it is still less than the possible cost of the querying. The querying could dominate very easily depending on how many bots intersect. That is why the cost of sorting the bots in each node is worth it because our goal is to make this algorithm the fastest it possibly can be. The load of the rebalancing of the tree doesnt very as much as the load of this algorithm. 

Additionally, we have been assuming that once we build the tree, we are just finding all the colliding pairs of the elements. In reality, there might be many different queries we want to do on the same tree. So this is another reason we want the tree to be built to make querying as fast as possible, because we don't know how many queries the user might want to do on it. In addition to finding all colliding pairs, its quite reasonable the user might want to do some k_nearest querying, some rectangle area querying, or some raycasting.


## Tree structure data seperate from elements in memory, vs intertwined

There is a certain appeal to storing the tree elements and tree data intertwined in the same piece of contiguous memory. Cache coherency should be improved since there is very little chance that the tree data, and the elements that belong to that node are not in the same cache block. But there is added complexity. Because the elements that are inserted into the tree are generic, the alignment of the objects may not match that of the tree data object. This would lead to wasted padded space that must be inserted into the tree to accomodate this. Additionally, while this data structure could be faster at querying, it would be slower if the user just wants to iterate through all the elements in the tree. The cache misses don't happen that often since the chance of a cache miss only happens on a per node basis, instead of per element. Moreover, I think the tree data structue itself is very small and has a good chance of being completely in a cache block.

When rebalancing, it is much easier to do it with two seperate arrays intsead of a heterogeneous array. The heterogenous array laid out in dfs in order made up of node types and bot types would give you better memory locality as you decended the tree, but comes at much complication with memory alignment. Also, since the size of the bots are variable based on the number type used for the bounding box, its possible that there doesnt exist a good alignment to allow the bots and nodes to be placed compactle interspersed next to each other. The result of that would be a lot of dead memory space inbetween the elements of the array, which would ironically make it less cache friendly.

## Tree structure as portable in memory, vs not.

A portable data structure has some benefits. The main benefit is that serialization is a no-op. Its already serialized and can be placed directly into binary file. The problem is that memory alignment is a platform specific thing and this binary file format would be expected to not be platform specific. So I didn't bother with this. The tree data is in its own heap allocated array, and the tree elements are in its own heap allocated array, and both arrays could be in completely different parts of the memory. Another alternative would be to be extremely liberal with the padding between the two arrays so that any platform could handle it, but this leaves a bad taste in my mouth since you still ultimately need to make an assumption about all platforms. The best alternative is you just put the two arrays next to each other in the file, and upon serialization/deserialization you account of padding. The problem with this is that serialization is not a no-op anymore. But to that I say, why is that a bad thing. You want different things at different times, so why not have a function to transform the data between the two even if it is a little slow? When the data structure is in meory you want it to be as fast as possible, so you should 100% take advantage of platform specific alignment properties. WHen the data structure is stored in a file, you want to optimizate platform independance and file size instead.

## Level of indirection vs None for elements in the tree.

It is important to note that what we are trying to speed up is the collision finding. This is the potentially n^2 computation. It is worth it shifting everything around in memory if it allows us to remove one level of indirection inside of the computationally expensive n^2 computation. The main benefit that comes from removing the level of indirection is cache coherency. If we just have pointers to all the objects in the tree, Then for every collision check we do against two objects, its possible that is a cache miss. This becomes a problem for large n.

If we were inserting references into the tree, then the original order of the bots is preserved during construction/destruction of the tree. However, we are inserting the actual bots to remove this layer of indirection. So when are done using the tree, we want to return the bots to the user is the same order that they were put in. This way the user can rely on indicies for other algorithms to uniquely identify a bot. To do this, during tree construction, we also build up a Vec of offsets to be used to return the bots to their original position. We keep this as a seperate data structure as it will only be used on destruction of the tree. If we were to put the offset data into the tree itself, it would be wasted space and would hurt the memory locality of the tree query algorithms. We only need to use these offsets once, during destruction. It shouldnt be the case that all querying algorithms that might be performed on the tree suffer performance for this.


## Copy vs NoCopy

This is the classic tradeoff: Memory vs Computation. Because we are not using a level of indirection, we have to reorder objects to match how they show up in the tree. You can reorder an array of objects in place, but it is a fairly expensive operation. You also have to do this twice (once to order them into the tree, once to order them back to the original). A less compuationally expensive way it to just use an auxilary array, but this requires twice the amount of memory. So in certain cases, you might not have the space for an auxiliary array if we are talking millions of objects. But even in those cases, probably the OS taking advantage of virtual memory would still be faster than reordering elements without an auxilliary array, but then you have to guarentee you have the hard drive space.

Here, I left the api flexible enough to use either option, with Copy being the suggested api.


## Leaves as unique type, vs single node type.

The leaf elements of the tree data structure don't need as much information as their parent nodes. They don't have dividers. So half of the nodes of the data structure has some fields that store no information. It can be tempting to give the leaf nodes a seperate type to improve memory usage, but the divider size is small, and the number of nodes in general is relaively small compared to the number of bots. Alignment can also be an issue. 

The leaves do not have dividers. This means that if we use the same type for both nonleaves, and leaves we have wasted space in our leaf objects. This wouldnt be so bad if it wernt for two things. First, this is a complete binary tree, so literally half the nodes are leaves. Second, before the tree is laid out in dfs in order order in memory, this means that the distance between the root node and its children is effected by the empty space all of the leaves between them of which a quarter of the leaves will be. Our goal is to make this as compact in memory as possible, so to avoid this, we simply have two different types. 


## Dfs in-order vs Dfs pre-order vs Bfs order.

Dfs in order uses more memory during construction from during recursion, but it gives you a memory layout that more matches how it is logically. Dfs order has the nice property of the following: If a node is to the left of another node in space, then that node is to the left of that node in memmory. 

In reality, it doesnt really matter because all this does is reduce cache misses when handling collision between two nodes which doesnt happen as often as an object to object basis. 

So that should give you an idea of how we achieve great memory compactness, but what about memory locality? Well lets think about what kind of pattern algorithms that work on trees tend to work. The main primitive that they use is accessing the two children nodes from a parent node. Fundamentally, if I have a node, I would hope that the two children nodes are somewhat nearby to where the node I am currently at is. More importantly, the further down the tree I go, I would hope this is more and more so the case! For example, the closeness of the children nodes to the root isnt that important since there is only one of those. On the other hand, the closeness of the children nodes for the nodes at the 5th level of the tree are more important since that are 32 of them. 

So how can we layout the nodes in memory to achieve this? Well, putting them in memory in breadth first order doesnt cut it. This achieves the exact opposite. For example, the children of the root are literally right next to it. On the other hand the children of the most left node on the 5th level only show up after you look over all the other nodes at the 5th level. It turns out in-order depth first search gives us the properties that we want. With this ordering, all the parents of leaf nodes are literally right next to them in memory. The children of the root node are potentially extremely far apart, but that is okay since there is only one of them.

## AABB vs Point + radius

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

Note, if the size of the elements is the same then the Point+radius only takes up 2 floating point values, so that might be better in certain cases. But even in this case, I think the cost of having to do floating point calculations when comparing every bot with every other bot in the query part of the algorithm is too much. With a AABB system, absolutely no floating point calculations need to be done to find all colliding pairs.


# Performance Themes

Now lets go over some things to note about the design that wern't exactly making a decisive choise about go down one path versus another.

## Only pick ODD height trees.

In order to assure that the leaf nodes all map to a relatively square piece of space, we force the tree to always have odd height. Picking an even height would be that one axis would be divided one ore time than the other axis.

## Multithreading

Evenly dividing up work into chuncks for up to N cores is the name of the game here. You don't want to make assumptions about how many cores the user has. Why set up your system to only take up advantage of 2 cores if 16 are available. So here, using a thread pool library like rayon is key. 

In big complicated system, it can be tempting to keep sub-systems sequential and then just allow multiple sub-systems to happen in parallel. But this is not fully taking advantage of multithreading. You want each subsystem to itself be parallizable. So this collision detection system is parallizable.

## Memory Locality



## Knowing the axis at compile time

A problem with using recursion on an kd tree is that every time you recurse, you have to access a different axis, so you might have branches in your code. A branch predictor might have problems seeing the pattern that the axis alternate with each call. One way to avoid this would be to handle the nodes in bfs order. Then you only have to alternate the axis once every level. But this way you lose the nice divide and conquer aspect of splitting the problem into two and handling those two problems concurrently. So to avoid, this, the axis of a particular recursive call is known at compile time. Each recursive call, will call the next with the next axis type. This way all branching based off of the axis is known at compile time. A downside to this is that the starting axis of the tree
must be chosen at compile time. It is certainly possible to create a wrapper around two specialized versions of the tree, one for each axis, but this would leads to alot of generated code, for benefit. Not much time is spent handling the root node anyway, so even if the suboptimal starting axis is picked it is not that big of a deal.


# Other ramblings/ Questions

## Liquid.

In liquid simulations the cost of querying dominates EVEN MORE than rebalancing since as opposed to particles that always repel, liquid particles repel if close to each other, but all attract if somewhat close. This means that the aabb's will intersect more often as the system tends to have overlapping aabbs.


## Rigid Bodies

If you want to use this tree for rigid bodies you have to overcome an obvious problem. You cannot move the bounding boxes once the tree it constructed. So while you are querying the tree to find bots that collide, you cannot move them then and there. An option is to insert loose bounding boxes and allow the bots to be moved within the loose bounding boxes. And then if the move need to be moved so much that they have to be moved out of the loose bounding boxes, re-construct the tree and query again until all the bots have finished moving, or you have done a set number of physics iterations, at which point you have some soft-body collision resolution fallback.

Ironically, even though to have a rigid body system you would need to have looser bounding boxes, performance of the system overall would probably improve. This is because rigid body systems enforce a level of planarity. In soft body, it is possible for literally every bot to be on touching each other (like a complete graph) causing an enourmous slow down. A rigid body physics system would not allow this state to ever happen.

## Continuous Collision detection

In order to use dinotree for continuous collision detection (suitable for very fast objects, for example), the aabbs that you insert into it must be big enough to contain the position of a bot before and after the time step. This way, upon aabb collisions, you can do fine grained contiuous collision detection and resolution.

## 3D

What about 3d? Making this multi dimensional would have added to the complexity, so the design desision was made to only target 2d. Its much easier for me as a developer to visualize 2d. So as a good first iteration of this algorithm, targeting just 2d simplifies things. Expanding it to 3d, shouldnt take too much effort. The hard part would be over. Code architecture would hopefully not need to be changed much.  


That's not to say one couldn't still take advantage of this system in a 3d simulation. Every bot could store a height field that you do an extra check against in the collision function. The downside is that imagine if there were many bots stacked on top of each other, but you only wanted to query a small cube. Then doing it this way, your query function would have to consider all those bots that were stacked. If there are only a few different height values, one could maintain a seperte 2d dinotree for each level. Looking at the real world though, and most usecases, your potential z values are much less than our potetial x and y values. So for many cases, it probably better to use the tree for 2 dimentions, and then naively handling the 3rd. Then you dont suffer from the "curse of dimensionality"?

## Pipelining

Pipelining. It might be possible to pipeline the process so that rebalancing and querying happen at the same time with the only downside being that bots react to their collisions one step later. To account for that the aabb's could be made slightly bigger and predict what they will hit the next step. This system might almost double the performance if the system wasnt using all its cores for each stage (rebalancing/querying), but we're talking a lot of added complexity, but probably more systems don't have that many cores.


## Exploiting temporal locality

There's really two different contexts in which temporal locality can be talked about. There's the time locality between states of the 2d world between calls to create and destroy the tree. Then there's the time locality of the internal locality as it makes it ways through the alogrithm.

The sort answer? It does not. For a while I had the design where the dividers would move as those they had mass. They would gently be pushed to which ever side had more bots. The problem with this approach is that the divider locations will mostly of the time be sub optimial. And the cost saved in rebalancing just isnt enough for the cost added to querying with a suboptimal partitioning. By always partitioning optimally, we get guarentees of the maximum number of bots in a node. Remember querying is the bottleneck, not rebalancing.


Another strategy to exploit temporal locality is by inserting looser bounding boxes into the tree and caching the results of a query for longer than one step. The upside to this is that you only have to build and query the tree every couple of iterations. There are a number of downsides, though:

* Your system performance now depends on the speed of the bots. The faster your bots move, the bigger their loose bounding boxes, the slower the querying becomes. This isnt a big deal considering the ammount that a bot moves between two frames is expected to be extremely small. But still, there are some corner cases where performance would deteriorate. For example, if every bot was going so fast it would just from one end of you screen to the other between world steps. So you would need to bound the velocity of your bots to a small value.

* You have to implement all the useful geometry tree functions all over again, or you can only use the useful geometry functions at the key world steps where the tree actually is constructed. For example, if you want to query a rectanlge area, the tree provides a nice function to do this, but you only have the tree every couple of iterations. The result is that you have to somehow implement a way to query all the bots in the rectangle area using your cached lists of colliding bots, or simply only query on the world steps in which you do have the built tree. Those queries will also be slower since you are working on a tree with loose boxes.

* Every bot needs to have a member variable that is its index. This isnt ideal to have since its redundant information. You can figure out a bots index from its position within the list fed into the tree. So it is just wasted space. For a very intestive algorithms like collision querying, having the memory footprint being operated being small is crucial. We can't rely on pointer offsets to determine the indicies of which bots are colliding when using the tree since we reordered the bots directly to make the tree to avoid a level of indirection. This increases the separation between the other fields.

* The maximum load on a world step is greater. Sure amortised, this caching system may computation, but the times you do construct and query the tree, you are doing so with loose bounding boxes. On top of that, while querying, you also have to build up a seperate data structure that caches the colliding pairs you find. 

* The api of the dinotree is flexible enough that you can implement loose bounding box + caching on top of it (without sacrificing parallelism) if desired.

So in short, this system doesnt take advantage of temporal locality, but the user can still take advantage of it by inserting loose bounding boxes and then querying less frequently to amortize the cost. I didnt explore this since I need to construct the tree every iteration anyway in my android demo, because I wanted the feedback of the user moving his finger around to be imeddiate. So to find all the bots touching the finger i need the tree to be up to date every single iteration. This is because I have no way of know where the user is going to put his finger down. I cant bound it by velocity or acceleration or anything. If I were to bound the touches "velocity", it would feel more slugish i think. It would also delay the user putting a new touch down for one iteration possibly.

# In depth algorithm overview:


## Construction

Construction works as follows, Given: a list of bots.

For every node we do the following:
1. First we find the median of the remaining bots (using pattern defeating quick select) and use its position as this nodes divider.
2. Then we bin the bots into three bins. Those strictly to the left of the divider, those strictly to the right, and those that intersect.
3. Then we sort the bots that intersect the divider along the opposite axis that was used to finding the median. These bots now live in this node.
4. Now this node is fully set up. Recurse left and right with the bots that were binned left and right. This can be done in parallel.


### Memory complexity during construction

Below is an example showing space usage for 5 aabb objects:

```text
x=size of one user defined object (without its aabb)
i=size of one index of an aabb object
r=size of one axis aligned bounding box.



1) xxxxx ->user provides bots

2) xxxxx iririririr -> the inner tree of (index,aabb) elements is created and sorted. 

3) xxxxx iririririr xrxrxrxrxr -> the inner tree is used to create the dinotree. 

4) xxxxx xrxrxrxrxr iririririr iiiii ->the indicies of the bots in generated (so that we know the indicies of where to move the bots back to)

5) xxxxx xrxrxrxrxr iiiii ->remove inner tree. Now the tree is setup and ready to be used. The user can call apply() to apply changes to the right bots (by using the index list stored).

6) xxxxx -> dinotree is destroyed leaving just the original slice.

```
So there is a step in the above example where quite a bit of memory is needed (step 4). Space usage at this step is 2*n*x+2*n*r+2*n*i. The size of x is user defined, so it could be smaller than r, but not likely. Likely x>r>i.  So we can bound the previous equation by 2*n*x+2*n*x+2*n*x=6xn. So it is linear space complexity, but it is a large constant. Space usage is high but this buys us an optimal memory placement of the final constructed tree to speed up the querying as much as possible.




### memory usage of construction

x=object (without its aabb)
i=index of an aabb object
r=aabb



xxxxx ->user provides bots

xxxxx iririririr -> the inner tree of (index,aabb) elements is created and sorted. 

xxxxx iririririr xrxrxrxrxr -> the inner tree is used to create the dinotree. 

xxxxx xrxrxrxrxr iririririr iiiii ->the indicies of the bots in generated (so that we know the indicies of where to move the bots back to)

xxxxx xrxrxrxrxr iiiii ->remove inner tree. Now the tree is setup and ready to be used. The user can call apply() to apply changes to the right bots (by using the index list stored).

xxxxx -> dinotree is destroyed leaving just the original slice.

###rebal


User puts the bots into the memory space
xxxxxxxx-------------------------
Sort and bin
aabbcccc-------------------------
Move to correct place
-----------aa==bbcccc------------




## Finding all intersecting pairs

Done via divide and conquer. For every node we do the following:
1) First we find all intersections with bots in that node using sweep and prune..
2) We recurse left and right finding all bots that intersect with bots in the node.
	Here we can quickly rule out entire nodes and their decendants if a node's aabb does not intersect
	with this nodes aabb.
3) At this point the bots in this node have been completely handled. We can safely move on to the children nodes 
   and treat them as two entirely seperate trees. Since these are two completely disjoint trees, they can be handling in
   parallel.


## Nbody

The nbody algorithm works in three steps. First a new version tree is built with extra data for each node. Then the tree is traversed taking advantage of this data. Then the tree is traversed again applying the changes made to the extra data from the construction in the first step.

The extra data that is stored in the tree is the sum of the masses of all the bots in that node and all the bots under it. The idea is that if two nodes are sufficiently far away from one another, they can be treated as a single body of mass.

So once the extra data is setup, for every node we do the following:
	Gravitate all the bots with each other that belong to this node.
	Recurse left and right gravitating all bots encountered with all the bots in this node.
		Here once we reach nodes that are sufficiently far away, we instead gravitate the node's extra data with this node's extra data, and at this point we can stop recursing.
	At this point it might appear we are done handling this node the problem has been reduced to two smaller ones, but we are not done yet. We additoinally have to gravitate all the bots on the left of this node with all the bots on the right of this node.
    For all nodes encountered while recursing the left side,
    	Recurse the right side, and handle all bots with all the bots on the left node.
    	If a node is suffeciently far away, treat it as a node mass instead and we can stop recursing.
    At this point we can safely exclude this node and handle the children and completely independent problems.



## Raycasting


TODO explain

## Knearest

TODO explain


## Rect

TODO explain.


