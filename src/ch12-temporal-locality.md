
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
