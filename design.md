## For the reader

Below I go over a bunch of design problems/decisions while developing this dinotree library.

I've written this assuming the reader knows what a [kdtree](https://en.wikipedia.org/wiki/K-d_tree),[sweep and prune](https://en.wikipedia.org/wiki/Sweep_and_prune) ,and [quad tree](https://en.wikipedia.org/wiki/Quadtree) are. 


## (Sweep and Prune) vs (Kd Tree) vs (KdTree + Sweep and Prune)

Sweep and prune is a simple AABB collision finding system, but it degenerates as there are more and more "false-positives" (objects that intersect on one axis, but not both). Kd Trees are great, but non-point objects that can't be inserted into children are left in the parent node and those objects must be collision checked with everybody else naivley. The tree height might also end up very large to satisfy the requirement that the leaf has only one element. A better solution is to use both. 

The basic idea is that you use a tree up until a specific tree height, and then switch to sweep and prune, and then additionally use sweep and prune for elements stuck in higher up tree nodes. The sweep and prune algorithm is a good candidate to use since it uses very little memory (just a stack that can be reused as you handle decendant nodes). But the real reason why it is good is the fact that the bots that belong to a non-leaf node in a kd tree are likely to be stewn across the divider in a line. Sweep and prune degenerates when the active list that it must maintain has many bots that end up not intersecting. This isnt likely to happen for the bots that belong to a node. The bots that belong to a node are guarenteed to touch the divider. If the divider partitions bots based off their x value, then the bots that belong to that node will all have x values that are roughly close together (they must intersect divider), but they y values can be vastly different (all the bots will be scattered up and down the dividing line). So when we do sweep and prune, it is important that we sweep and prune along axis that is different from the axis along which the divider is partitioning, otherwise it will degenetate to pratically the naive algorithm.


## KD tree vs Quad Tree

The main benefit of a quad tree is that tree construction is fast since we don't need to find the median at each level. But that comes at a cost of a potentially not great partitioning of the physical elements. Our goal is to make the querying as fast as possible as this is the part that can vary and dominate very easily in desnse/clumped up situations. The slow construction time of the kdtree is not ideal, but it is a very consistent load (doesnt vary from how clumped the elements are).

KD trees are also great in a multithreaded setting. With a kd tree, you are guarenteed that for any parent, there are an equal number of objects if you recurse the left side and the right side since you specifically chose the divider to be the median. 
This means that during the query phase, the work-load will be fairly equal on both sides. It might not be truely equal because even though for a given node, you can expect both the left and right sub-trees to have an equal number of elements, they most likely will be distributed differently within each sub-tree. For example the left sub-tree might have all of its elements stuck in just one node, while the right sub-tree has all of its elements in its leaf nodes. However, the size of each sub-tree is a somewhat good estimate of the size of the problem of querying it. So this is a good property for a divide and conquer multithreaded style algorithm. With a quad tree, the load is not guarenteed to be even between the four children nodes. 


## Tree space partitioning vs grid 

I wanted to make a collision system that could be used in the general case and did not need to be fine-tuned. Grid based collision systems suffer from the teapot-in-a-stadium problem. They also degenerate more rapidly as objects get more clumped up. If, however, you have a system where you have strict rules with how evenly distributed objects will be among the entire space you're checking collisions against, then I think a grid system can be better. But I think these systems are few and far in between. I think in most systems, for example, its perfectly possible for all the objects to exist entirely on one half of the space being collision checked leaving the other half empty. In such a case, half of the data structure of the grid system is not being used to partition anything. 


## Construction Cost vs Querying Cost


If you are simulating moving elements, it might seem slow to rebuild the tree every iteration. But from benching, most of the time querying is the cause of the slowdown. Rebuilding is always a constant load, but the load of the query can vary wildly depending on how many elements are overlapping.

For example, in a bench where inside of the collision call-back function I do a reasonable collision response with 80_000 bots, if there are 0.8 times (or 65_000 ) collisions or more, querying takes longer than rebuilding. For your system, it might be impossible for there to even be 0.8 * n collisions, in which case building the tree will always be the slower part. For many systems, 0.8 * n collisions can happen. For example if you were to simulate a 2d ball-pit, every ball could be touching 6 other balls (https://en.wikipedia.org/wiki/Circle_packing), and that is without soft-body physics. So in that system, there are 0.8 * n collisions. So in that case, querying is the bottle neck. With liquid or soft-body physics, the number can be every higher. up to n * n.

Rebuilding the first level of the tree does take some time, but it is still just a fraction of the entire building algorithm in some crucial cases, provided that it was able to partition almost all the bots into two planes. 

Additionally, we have been assuming that once we build the tree, we are just finding all the colliding pairs of the elements. In reality, there might be many different queries we want to do on the same tree. So this is another reason we want the tree to be built to make querying as fast as possible, because we don't know how many queries the user might want to do on it. In addition to finding all colliding pairs, its quite reasonable the user might want to do some k_nearest querying, some rectangle area querying, or some raycasting.

## Exploiting Temporal Locality (with loose bounding boxes)

The main reason against exploiting temporal locality is that adding any kind of "memory" to the tree where you save the positions of the dividers to use as good heuristic positions for next iterations will come at a cost of a sub optimal tree layout which will hurt the query algorithm. Our goal is to make the query algorithm as fast as possible since that is what can dominate.

One strategy to exploit temporal locality is by inserting looser bounding boxes into the tree and caching the results of a query for longer than one step. The upside to this is that you only have to build and query the tree every couple of iterations. There are a number of downsides, though:

* Your system performance now depends on the speed of the bots. The faster your bots move, the bigger their loose bounding boxes, the slower the querying becomes. This isnt a big deal considering the ammount that a bot moves between two frames is expected to be extremely small. But still, there are some corner cases where performance would deteriorate. For example, if every bot was going so fast it would just from one end of you screen to the other between world steps. So you would also need to bound the velocity of your bots to a small value.

* You have to implement all the useful geometry tree functions all over again, or you can only use the useful geometry functions at the key world steps where the tree actually is constructed. For example, if you want to query a rectanlge area, the tree provides a nice function to do this, but you only have the tree every couple of iterations. The result is that you have to somehow implement a way to query all the bots in the rectangle area using your cached lists of colliding bots, or simply only query on the world steps in which you do have the built tree. Those queries will also be slower since you are working on a tree with loose boxes.

* Every bot needs to have a member variable that is its index. This isnt ideal to have since its redundant information. You can figure out a bots index from its position within the list fed into the tree. So it is just wasted space. For a very intestive algorithms like collision querying, having the memory footprint being operated being small is crucial. We can't rely on pointer offsets to determine the indicies of which bots are colliding when using the tree since we reordered the bots directly to make the tree to avoid a level of indirection. This increases the separation between the other fields.

* The maximum load on a world step is greater. Sure amortised, this caching system may computation, but the times you do construct and query the tree, you are doing so with loose bounding boxes. On top of that, while querying, you also have to build up a seperate data structure that caches the colliding pairs you find. 

* The api of the dinotree is flexible enough that you can implement loose bounding box + caching on top of it (without sacrificing parallelism) if desired.

So in short, this system doesnt take advantage of temporal locality, but the user can still take advantage of it by inserting loose bounding boxes and then querying less frequently to amortize the cost. I didnt explore this since I need to construct the tree every iteration anyway in my android demo, because I wanted the feedback of the user moving his finger around to be imeddiate. So to find all the bots touching the finger i need the tree to be up to date every single iteration. This is because I have no way of know where the user is going to put his finger down. I cant bound it by velocity or acceleration or anything. If I were to bound the touches "velocity", it would feel more slugish i think. It would also delay the user putting a new touch down for one iteration possibly.

## Expoiting Temporal Locality (caching medians)

I would love to try the following: Instead of finding the median at every level, find an approximate median. Additionally, keep a weighted average of the medians from previous tree builds and let it degrade with time. Get an approximate median using median of medians. This would ensure worst case linear time when building one level of the tree. This would allow the rest of the algorithm to be parallelized sooner.

This would mean that query would be slower since we are not using heuristics and not using the true median, but this might be a small slowdown and it might speed of construction significatly.

For a while I had the design where the dividers would move as those they had mass. They would gently be pushed to which ever side had more bots. The problem with this approach is that the divider locations will mostly of the time be sub optimial. And the cost saved in rebalancing just isnt enough for the cost added to querying with a suboptimal partitioning. By always partitioning optimally, we get guarentees of the maximum number of bots in a node. Remember querying is the bottleneck, not rebalancing.


## Tree structure data seperate from elements in memory, vs intertwined

There is a certain appeal to storing the tree elements and tree data intertwined in the same piece of contiguous memory. Cache coherency would be improved since there is very little chance that the tree data, and the elements that belong to that node are not in the same cache block. But there is added complexity. Because the elements that are inserted into the tree are generic, the alignment of the objects may not match that of the tree data object. This would lead to wasted padded space that must be inserted into the tree to accomodate this. Additionally, while this data structure could be faster at querying, it would be slower if the user just wants to iterate through all the elements in the tree. The cache misses don't happen that often since the chance of a cache miss only happens on a per node basis, instead of per element. Moreover, I think the tree data structue itself is very small and has a good chance of being completely in a cache block.

When rebalancing, it is much easier to do it with two seperate arrays intsead of a heterogeneous array. The heterogenous array laid out in dfs in order made up of node types and bot types would give you better memory locality as you decended the tree, but comes at much complication with memory alignment. Also, since the size of the bots are variable based on the number type used for the bounding box, its possible that there doesnt exist a good alignment to allow the bots and nodes to be placed compactle interspersed next to each other. The result of that would be a lot of dead memory space inbetween the elements of the array, which would ironically make it less cache friendly.

## Tree structure as portable in memory, vs not.

A portable data structure has some benefits. The main benefit is that serialization can be a no-op. Its already serialized and can be placed directly into binary file. The problem is that memory alignment is a platform specific thing and this binary file format would be expected to not be platform specific. So I didn't bother with this. The tree data is in its own heap allocated array, and the tree elements are in its own heap allocated array, and both arrays could be in completely different parts of the memory. Another alternative would be to be extremely liberal with the padding between the two arrays so that any platform could handle it, but this leaves a bad taste in my mouth since you still ultimately need to make an assumption about all platforms. The best alternative is you just put the two arrays next to each other in the file, and upon serialization/deserialization you account of padding. The problem with this is that serialization is not a no-op anymore. But to that I say, why is that a bad thing. You want different things at different times, so why not have a function to transform the data between the two even if it is a little slow? When the data structure is in meory you want it to be as fast as possible, so you should 100% take advantage of platform specific alignment properties. When the data structure is stored in a file, you want to optimizate platform independance and file size instead.


## Leaves as unique type, vs single node type.

The leaf elements of the tree data structure don't need as much information as their parent nodes. They don't have dividers. So half of the nodes of the data structure haa a field that store no information. It can be tempting to give the leaf nodes a seperate type to improve memory usage, but the divider size is small, and the number of nodes in general is relaviely small compared to the number of bots. The leaves would also have to live in their own contrainer (or you'd have to deal with alignment problems), which would lead to more complexity while iterating down the tree.


## Dfs in-order vs Dfs pre-order vs Bfs order.

Dfs in order uses more memory during construction from during recursion, but it gives you a memory layout that more matches how it is logically. Dfs order has the nice property of the following: If a node is to the left of another node in space, then that node is to the left of that node in memmory. 

Dfs pre-order and post-order might be more cache friendly. The these layouts, there is just one memory segment that shrinks in size as you recurse. These layouts are more "recursive" looking to me.

The main primitive that they use is accessing the two children nodes from a parent node. Fundamentally, if I have a node, I would hope that the two children nodes are somewhat nearby to where the node I am currently at is. More importantly, the further down the tree I go, I would hope this is more and more so the case (as I have to do it for more and more nodes). For example, the closeness of the children nodes to the root isnt that important since there is only one of those. On the other hand, the closeness of the children nodes for the nodes at the 5th level of the tree are more important since that are 32 of them. 

So how can we layout the nodes in memory to achieve this? Well, putting them in memory in breadth first order doesnt cut it. This achieves the exact opposite. For example, the children of the root are literally right next to it. On the other hand the children of the most left node on the 5th level only show up after you look over all the other nodes at the 5th level. It turns out in-order depth first search gives us the properties that we want. With this ordering, all the parents of leaf nodes are literally right next to them in memory. The children of the root node are potentially extremely far apart, but that is okay since there is only one of them.

In reality, it doesnt really matter because all this does is reduce cache misses when handling collision between two nodes which doesnt happen as often as an object to object basis. 


## Mutable vs Mutable + Read-Only api

Most of the query algorithms only provide an api where they operate on a mutable reference
to the dinotree and return mutable reference results.
Ideally there would be sibling query functions that take a read-only dinotree
and produce read-only reference results. 
The main benefit would be that you could run different types of query algorithms on the tree
simulatiously safely. 
In rust right now, there isn't a easy way to re-use code
between two versions of an algorithm where one uses mutable references and one uses read-only references.
In the rust source code, you can see they do this using macros. From rust core's slice code:

```
// The shared definition of the `Iter` and `IterMut` iterators
macro_rules! iterator {
	...
}
```

But I didnt want to make these already complicated query algorithms even harder to read by
wrapping them all in macros. The simplest query algorithm, rect querying, does provide read-only versions.
I'm hoping that eventualy you will be able to 'parameterize over mutability' in rust so that i dont have to use macros.

In any-case, there arn't many use-cases that I can think of where you'd want read-only versions of certain 
query algorithms. For example, finding all colliding pairs. What would the user do with only read-only references
to all the colliding pairs? There may be some cases where the user would just want to draw or list all the colliding pairs,
but most use-cases I can think of, you want to actually mutate the pairs in some way.

Other query algorithms I can see some benefits. For example, raycasting. In raycasting the user might just want to
draw a line to the element that is returned from the raycast, in which case the user wouldn't need a mutable reference
to the result. So if I were to provide a read-only raycast api, and the user wanted to raycast many times, then that could
be easily be parallelized. For now, the user must use the mutable raycast api and handle each raycast sequentially. This
is simplier to implement (no macros), and the raycast algorithm is already very fast compared to other algrithms such
as constructing the tree and finding colliding pairs.


## Making HasAabb an unsafe trait vs Not

Making a trait unsafe is something nobody wants to do, but in this instance it lets as make some simple assumptions
that lets us do interesting things safely. 

The key idea is the following:
If two rectangle queries do not intersect, then it is guarenteed that the elements are mutually exclusive.
This means that we can safely return mutable references to all the elements in the first rectangle,
and all the elements in the second rectangle simultaneously. 

For this to work the user must uphold the contract of HasAabb such that the aabb returned is always the same
while it is inserted in the tree.
This is hard for the user not to do since they only have read-only reference to self, but still possible using
RefCell or Mutex. If the user violates this, then despite two query rectangles being mutually exclusive,
the same bot might be in both. So at the cost of making HasAabb unsafe, we can make the MultiRect Api not unsafe.


## Different Data Layouts


There are three main datalayouts for each of the elements in a dinotree that are interesting:
```
(Rect<isize>,&mut T)
(Rect<isize>,T),
&mut (Rect<isize>,T)
```

Because the tree construction code is generic over the elements that are inserted (as long as they implement HasAabb),
The user can easily try all three data layouts.

The default layout is almost always the fastest. 
There are a few corner cases where if T is very small, and the bots are very dense, direct is faster, but it is marginal.

The default layout is good because during dinotree construction and querying we make heavy use of aabb's, but don't actually need T all that much. We only need T when we actually detect a collision, which doesnt happen that often. Most of the time we are just
ruling out possible colliding pairs by checking their aabbs.

Yes the times we do need T could lead to a cache miss, but this happens infrequently enough that that is ok.
Using the direct layout we are less likely to get a cache miss on access of T, but now the aabbs are further apart from each other
because we have all these T's in the way. It also took us longer to put the aabb's and T's all together in one contiguous memory.

One thing that is interesting to note is that if T has its aabb already inside of it, then `(Rect<isize>,&mut T)` duplicates that memory. This is still faster than `&mut (Rect<isize>,T)`, but it still feels wasteful. To get around this, you can make it so that T doesnt have the aabb inside of it, but it just has the information needed to make it. Then you can make the aabbs as you make the `(Rect<isize>,&mut T)` in memory. So for example T could have just in it the position and radius. This way you're using the very fast tree data layout of `(Rect<isize>,&mut T)`, but at the same time you don't have two copies of every objects aabb in memory. 

If we were inserting references into the tree, then the original order of the bots is preserved during construction/destruction of the tree. However, for the direct layout, we are inserting the actual bots to remove this layer of indirection. So when are done using the tree, we want to return the bots to the user is the same order that they were put in. This way the user can rely on indicies for other algorithms to uniquely identify a bot. To do this, during tree construction, we also build up a Vec of offsets to be used to return the bots to their original position. We keep this as a seperate data structure as it will only be used on destruction of the tree. If we were to put the offset data into the tree itself, it would be wasted space and would hurt the memory locality of the tree query algorithms. We only need to use these offsets once, during destruction. It shouldnt be the case that all querying algorithms that might be performed on the tree suffer performance for this.


## Forbidding the user from violating the invariants of the tree

We have an interesting problem with our tree. We want the user to be able to mutate the elements directly in the tree,
but we also do not want to let them mutate the aabb's of each of the elements of the tree. Doing so would
mean we'd need to re-construct the tree.

So when iterating over the tree, we can't return &mut T. So to get around that, there is ProtectedBBox that wraps
around a &mut T that also implements HasAabb. 

So that makes the user unable to get a &mut T, but even if we just give the user a &mut [ProtectedBBox<T>] where is another problem. The user could swap two node's slices around. So to get around that we have ProtectedBBoxSlice that wraps
around a &mut [ProtectedBBox<T>] and provides an interator who Item is a ProtectedBBox<T>.

But there is still another problem, we can't return a &mut Node either. Because then the user could swap the entire node
between two nodes in the tree. So to get around that we have a ProtectedNode that wraps around a &mut Node.

So that's alot of work, but now the user is physically unable to violate the invariants of the tree and at the same time
we do not have a level of indirection. It is tempting
to just let it be the user's responsibility to not violate the invariants of the tree, and that would be a fine design
choice if it weren't for the fact that HasAabb is unsafe (See Making HasAabb an unsafe trait vs Not Section).


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

## AABB data layout

The aabb we use is made up of ranges that look like : start,end instead of start,length.  If you define them as a start and a length then querying intersections between rectangles requires that you do floating point arithmatic. The advantage of the start,end data layout is that all the dinotree query algorithms don't need to do a single floating point calculation. It is all
just comparisons. The downside, is that if you want the dimentions on the aabb, you have to calculate them, how this isnt something that any of the tree algorithms need. 


## Only pick ODD height trees.

In order to assure that the leaf nodes all map to a relatively square piece of space, we force the tree to always have odd height. Picking an even height would be that one axis would be divided one ore time than the other axis.

## Multithreading

Evenly dividing up work into chuncks for up to N cores is the name of the game here. You don't want to make assumptions about how many cores the user has. Why set up your system to only take up advantage of 2 cores if 16 are available. So here, using a thread pool library like rayon is useful. 

In big complicated system, it can be tempting to keep sub-systems sequential and then just allow multiple sub-systems to happen in parallel if they are computation heavy. But this is not fully taking advantage of multithreading. You want each subsystem to itself be parallizable. So this collision detection system is parallizable. 


## Knowing the axis at compile time

A problem with using recursion on an kd tree is that every time you recurse, you have to access a different axis, so you might have branches in your code. A branch predictor might have problems seeing the pattern that the axis alternate with each call. One way to avoid this would be to handle the nodes in bfs order. Then you only have to alternate the axis once every level. But this way you lose the nice divide and conquer aspect of splitting the problem into two and handling those two problems concurrently. So to avoid, this, the axis of a particular recursive call is known at compile time. Each recursive call, will call the next with the next axis type. This way all branching based off of the axis is known at compile time. A downside to this is that the starting axis of the tree
must be chosen at compile time. It is certainly possible to create a wrapper around two specialized versions of the tree, one for each axis, but this would leads to alot of generated code, for benefit. Not much time is spent handling the root node anyway, so even if the suboptimal starting axis is picked it is not that big of a deal.


## Liquid.

In liquid simulations the cost of querying dominates EVEN MORE than rebalancing since as opposed to particles that always repel, liquid particles repel if close to each other, but all attract if somewhat close. This means that the aabb's will intersect more often as the system tends to have overlapping aabbs.


## Rigid Bodies

If you want to use this tree for rigid bodies you have to overcome an obvious problem. You cannot move the bounding boxes once the tree it constructed. So while you are querying the tree to find bots that collide, you cannot move them then and there. An option is to insert loose bounding boxes and allow the bots to be moved within the loose bounding boxes. And then if the move need to be moved so much that they have to be moved out of the loose bounding boxes, re-construct the tree and query again until all the bots have finished moving, or you have done a set number of physics iterations, at which point you have some soft-body collision resolution fallback.

Ironically, even though to have a rigid body system you would need to have looser bounding boxes, performance of the system overall could improve. This is because rigid body systems enforce a level of spareness. In soft body, it is possible for literally every bot to be on touching each other causing many colliding pairs to be handled. A rigid body physics system would not allow this state to happen.

## Continuous Collision detection

In order to use dinotree for continuous collision detection (suitable for very fast objects, for example), the aabbs that you insert into it must be big enough to contain the position of a bot before and after the time step. This way, upon aabb collisions, you can do fine grained contiuous collision detection and resolution.

## 3D

What about 3d? Making this library multi dimensional would have added to the complexity, so the design desision was made to only target 2d. Its much easier for me as a developer to visualize 2d. So as a good first iteration of this library, targeting just 2d simplifies things. Expanding it to 3d, shouldnt take too much effort. The hard part would be over. Code architecture would hopefully not need to be changed much.  


That's not to say one couldn't still take advantage of this system in a 3d simulation. Every bot could store a height field that you do an extra check against in the collision function. The downside is that imagine if there were many bots stacked on top of each other, but you only wanted to query a small cube. Then doing it this way, your query function would have to consider all those bots that were stacked. If there are only a few different height values, one could maintain a seperte 2d dinotree for each level. Looking at the real world though, and most usecases, your potential z values are much less than our potetial x and y values. So for many cases, it probably actually better to use the tree for 2 dimentions, and then naively handling the 3rd. Then you dont suffer from the "curse of dimensionality"? An even better 3d solution might be to use sweep-and-prune in the z dimesion, and the 2d dinotree for x and y.

## Pipelining

It might be possible to pipeline the process so that rebalancing and querying happen at the same time with the only downside being that bots react to their collisions one step later. To account for that the aabb's could be made slightly bigger and predict what they will hit the next step. This system could almost double the performance if the system wasnt using all its cores for each stage (rebalancing/querying), but we're talking a lot of added complexity, but probably more systems don't have that many cores.

## Testing correctness

Simply using rust has a big impact on testing. Because of its heavy use of static typing, many bugs are caught at compile time. This translates to less testing as there are fewer possible paths that the produced program can take. Also the fact that the api is generic over the underlying number type used is useful. This means that we can test the system using integers and we can expect it to work for floats. It is easier to test with integers since we can more easily construct specific scenarios where one number is one value more or less than another. So in this way we can expose corner cases.

A good test is a test that tests with good certainty that a large portion of code is working properly.
Maintaining tests comes at the cost of anchoring down the design of the production code in addition to having to be maintained themselves. As a result, making good abstractions between your crates and modules that have simple and well defined apis is very important. Then you can have a few simple tests to fully excersise an api and verify large amounts of code.

This crate's sole purpose is to provide a method of providing collision detection that is faster than the naive method. So a good high level test would be to compare the query results from using this crate to the naive method (which is much easier to verify is correct). This one test can be performed on many different inputs of lists of bots to try to expose any corner cases. So this one test when fed with both random and specifically tailed inputs to expose corner cases, can show with a lot of certainty that the crate is satisfying the api. 

The tailored inputs is important. For example, a case where two bounding boxes collide but only at the corner is an extremely unlikely case that may never present themselves in random inputs. To test this case, we have to turn to more point-directed tests with specific constructed set up input bot lists. They can still be verified in the same manner, though.

## Benching

Writing benches that validate every single piece of the algorithm design is a hassle although it would be nice. Ideally you dont want to rely on my word to say that, for example, using sweep and prune to find colliding pairs actually sped things up. It could be that while the algorithm is correct and fast that this particular aspect of the algorithm actually slows things down. 

So I dont think writing low level benches are worth it. If you are unsure of a piece of code, you can bench the algorithm as a whole, change a piece of the algorithm, and bench again and compare results. Because at the end of the day, we already tested the correctness, and that is the most important thing. So I think this strategy, coupled with code coverage and just general reasoning of the code can supplement tons of benches to validate the innards of the algorithm.

Always measure code before investing time in optimizing. As you design your program. You form in your mind ideas of what you think the bottle necks in your code are. When you actually measure your program, your hunches can be wildly off.

When dealing with parallelism, benching small units can give you a warped sense of performance. Onces the units are combined, there may be more contention for work stealing. With small units, you have a false sense that the cpu's are not busy doing other things. For example, I parallalized creating the container range for each node. Benchmarks showed that it was faster. But when I benched the rebalancing as a whole, it was slower with the parallel container creation. So in this way, benching small units isnt quite as useful as testing small units is. That said, if you know that your code doesnt depend on some global resource like a threadpool, then benching small units is great.

Platform dependance. Rust is a great language that strives for platform independant code. But at the end of the day, even though rust programs will behave the same on multiple platforms, their performance might be wildly different. And as soon as you start optimizing for one platform you have to wonder whether or not you are actually de-optimizing for another platform. For example, rebalancing is much slower on my android phone than querying. On my dell xps laptop, querying is the bottle neck instead. I have wondered why there is this disconnect. I think part of it is that rebalancing requires a lot of sorting, and sorting is something where it is hard to predict branches. So my laptop probably has a superior branch predictor. Another possible reason is memory writing. Rebalancing involves a lot of swapping, whereas querying does involve in any major writing to memory outside of what the user decides to do for each colliding pair. In any case, my end goal in creating this algorithm was to make the querying as fast as possible so as to get the most consistent performance regardless of how many bots were colliding.


# Dynamic Allocation

Sure dynamic allocation is "slow", but that doesn't mean you should avoid it. You should use it as a tool to speed up a program. It has a cost, but the idea is that with that allocated memory you can get more performance gains.

The problem is that everybody has to buy into the system for it to work. Anybody who allocated a bunch of memory and doesn't return it because they want to avoid allocating it again is hogging that space for longer than it needs it.

Dynamic allocation is fast. Dynamically allocating large vecs in one allocation is fast. Its only when you're dynamically allocting thousands of small objects does it become bad. Even then, probably the allocations are fast, but because the memory will likely be fragmented, iterating over a list of those objects could be very slow. Concepually, I have to remind myself that if you dynamically allocate a giant block, its simply reserving that area in memory. There isnt any expensive zeroing out of all that memory unless you want to do it. That's not to say the complicated algorithms the allocator has to do arn't complicated, but still relatively cheap.

Often times, its not the dynamic allocation that is slow, but some other residial of doing it in the first place. For example, dynamically allocating a bunch of pointers to an array, and then sorting that list of pointers. The allocation is fast, its just that there is no cache coherency. Two pointers in your list of pointers could very easily point to memory locations that are very far apart.

Writing apis that don't do dynamic allocation is tough and can be cumbursome, since you probably have to have the user give you a slice of a certain size. On the other hand this let's the user know exactly how much memory your crate needs. But users should have some faith that a dynamic allocation system is good enough for most purposes to justify a more complicated api where users have to manually manage memory. 

The thing is that if you don't use dynamic allocation, and you instead reserve some giant piece of memory for use of your system, then that memory is not taken advanage of when your system is not using it. It is wasted space. If you know your system will always be using it then sure it is fine. But I can see this system being used sometimes only 30 times a second. That is a lot of inbetween time where that memory that cannot be used by anything else. So really, the idea of dynamic allocation only works is everybody buys into the system. Another option is to make your api flexible enough that you pass is a slice of "workspace" memory, so that the user can decide whether to dynamically allocate it everytime or whatever. But this complicates the api for a very small portion of users who would want to not using the standard allocator.

# When inserting float based AABBs, prefer NotNaN<> over OrderedFloat<>.

If you want to use this for floats, NotNaN has no overhead for comparisions, but has overhead for computation, the opposite is true for OrderedFloat.
For querying the tree colliding pairs, since no calculations are done, just a whole lot of comparisions, prefer NotNaN<>.
Other queries do require arithmatic, like raycast and knearest. So in those cases maybe ordered float is preferred.
















# In depth algorithm overview:


## Construction

Construction works as follows, Given: a list of bots.

For every node we do the following:
1. First we find the median of the remaining bots (using pattern defeating quick select) and use its position as this nodes divider.
2. Then we bin the bots into three bins. Those strictly to the left of the divider, those strictly to the right, and those that intersect.
3. Then we sort the bots that intersect the divider along the opposite axis that was used to finding the median. These bots now live in this node.
4. Now this node is fully set up. Recurse left and right with the bots that were binned left and right. This can be done in parallel.


## Finding all intersecting pairs

Done via divide and conquer. For every node we do the following:
1) First we find all intersections with bots in that node using sweep and prune..
2) We recurse left and right finding all bots that intersect with bots in the node.
	Here we can quickly rule out entire nodes and their decendants if a node's aabb does not intersect
	with this nodes aabb.
3) At this point the bots in this node have been completely handled. We can safely move on to the children nodes 
   and treat them as two entirely seperate trees. Since these are two completely disjoint trees, they can be handling in
   parallel.


## Nbody (experimental)

Here we use divide and conquer.

The nbody algorithm works in three steps. First a new version tree is built with extra data for each node. Then the tree is traversed taking advantage of this data. Then the tree is traversed again applying the changes made to the extra data from the construction in the first step.

The extra data that is stored in the tree is the sum of the masses of all the bots in that node and all the bots under it. The idea is that if two nodes are sufficiently far away from one another, they can be treated as a single body of mass.

So once the extra data is setup, for every node we do the following:
	Gravitate all the bots with each other that belong to this node.
	Recurse left and right gravitating all bots encountered with all the bots in this node.
		Here once we reach nodes that are sufficiently far away, we instead gravitate the node's extra data with this node's extra data, and at this point we can stop recursing.
	At this point it might appear we are done handling this node the problem has been reduced to two smaller ones, but we are not done yet. We additionally have to gravitate all the bots on the left of this node with all the bots on the right of this node.
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


