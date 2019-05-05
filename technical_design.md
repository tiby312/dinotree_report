export RUSTFLAGS="-C no-vectorize-loops -C no-vectorize-slp -llvm-args \"-inline\"" "



# Parallalization

Parallalization is done using rayon. The rust slice provided split_at_mut() and rayon's join() are two extremely powerful constructs. 
At a certain point while going down the tree, we switch to a sequential version as recommended by rayon's usage guidelines when used in recursive divide and conquer problems. 


#Colfind
I am not convinced that storing points+user supplied left,top,right,bottom border retrieval functions is faster than just
storing the aabb. This strategy would use less memory. (No need to sore the aabb, just the center point), But the aabb ends up needing to be calculated from the point every time its needed. I'm doubtful that this is faster because many floating point operations would be necessary, and there would be a duplication of efforts in re-generating the aabb during querying. During the colfinding,
the aabb for a bot might need to be checks a bunch of times. Thats a lot of extra floating point operations. So I think having the computed aabb in memory is better. The other downside is that you lose generality. All the aabb's must be the same size.


# Rigid Bodies

If you want to use this tree for rigid bodies you have to overcome an obvious problem. You cannot move the bounding boxes once the tree it constructed. So while you are querying the tree to find bots that collide, you cannot move them then and there. An option is to insert loose bounding boxes and allow the bots to be moved within the loose bounding boxes. And then if the move need to be moved so much that they have to be moved out of the loose bounding boxes, re-construct the tree and query again until all the bots have finished moving, or you have done a set number of physics iterations, at which point you have some soft-body collision resolution fallback.

Ironically, even though to have a rigid body system you would need to have looser bounding boxes, performance of the system overall would probably improve. This is because rigid body systems enforce a level of planarity. In soft body, it is possible for literally every bot to be on touching each other (like a complete graph) causing an enourmous slow down. A rigid body physics system would not allow this state to ever happen.



# Exploiting temporal locality

There's really two different contexts in which temporal locality can be talked about. There's the time locality between states of the 2d world between calls to create and destroy the tree. Then there's the time locality of the internal locality as it makes it ways through the alogrithm.

The sort answer? It does not. For a while I had the design where the dividers would move as those they had mass. They would gently be pushed to which ever side had more bots. The problem with this approach is that the divider locations will mostly of the time be sub optimial. And the cost saved in rebalancing just isnt enough for the cost added to querying with a suboptimal partitioning. By always partitioning optimally, we get guarentees of the maximum number of bots in a node. Remember querying is the bottleneck, not rebalancing.




Another strategy to exploit temporal locality is by inserting looser bounding boxes into the tree and caching the results of a query for longer than one step. The upside to this is that you only have to build and query the tree every couple of iterations. There are a number of downsides, though:

* Your system performance now depends on the speed of the bots. The faster your bots move, the bigger their loose bounding boxes, the slower the querying becomes. This isnt a big deal considering the ammount that a bot moves between two frames is expected to be extremely small. But still, there are some corner cases where performance would deteriorate. For example, if every bot was going so fast it would just from one end of you screen to the other between world steps. So you would need to bound the velocity of your bots to a small value.

* You have to implement all the useful geometry tree functions all over again, or you can only use the useful geometry functions at the key world steps where the tree actually is constructed. For example, if you want to query a rectanlge area, the tree provides a nice function to do this, but you only have the tree every couple of iterations. The result is that you have to somehow implement a way to query all the bots in the rectangle area using your cached lists of colliding bots, or simply only query on the world steps in which you do have the built tree. Those queries will also be slower since you are working on a tree with loose boxes.

* Every bot needs to have a member variable that is its index. This isnt ideal to have since its redundant information. You can figure out a bots index from its position within the list fed into the tree. So it is just wasted space. For a very intestive algorithms like collision querying, having the memory footprint being operated being small is crucial. We can't rely on pointer offsets to determine the indicies of which bots are colliding when using the tree since we reordered the bots directly to make the tree to avoid a level of indirection. This increases the separation between the other fields.

* The maximum load on a world step is greater. Sure amortised, this caching system may computation, but the times you do construct and query the tree, you are doing so with loose bounding boxes. On top of that, while querying, you also have to build up a seperate data structure that caches the colliding pairs you find. 

* The api of the dinotree is flexible enough that you can implement loose bounding box + caching on top of it (without sacrificing parallelism) if desired.



So in short, this system doesnt take advantage of temporal locality, but the user can still take advantage of it by inserting loose bounding boxes and then querying less frequently to amortize the cost. I didnt explore this since I need to construct the tree every iteration anyway in my android demo, because I wanted the feedback of the user moving his finger around to be imeddiate. So to find all the bots touching the finger i need the tree to be up to date every single iteration. This is because I have no way of know where the user is going to put his finger down. I cant bound it by velocity or acceleration or anything. If I were to bound the touches "velocity", it would feel more slugish i think. It would also delay the user putting a new touch down for one iteration possibly.


# Continuous Collision detection

In order to use dinotree for continuous collision detection (suitable for very fast objects, for example), the aabbs that you insert into it must be big enough to contain the position of a bot before and after the time step. This way, upon aabb collisions, you can do fine grained contiuous collision detection and resolution.


# Extensions and Improvements

What about 3d? Making this multi dimensional would have added to the complexity, so the design desision was made to only target 2d. Its much easier for me as a developer to visualize 2d. So as a good first iteration of this algorithm, targeting just 2d simplifies things. Expanding it to 3d, shouldnt take too much effort. The hard part would be over. Code architecture would hopefully not need to be changed much.  


That's not to say one couldn't still take advantage of this system in a 3d simulation. Every bot could store a height field that you do an extra check against in the collision function. The downside is that imagine if there were many bots stacked on top of each other, but you only wanted to query a small cube. Then doing it this way, your query function would have to consider all those bots that were stacked. If there are only a few different height values, one could maintain a seperte 2d dinotree for each level. Looking at the real world though, and most usecases, your potential z values are much less than our potetial x and y values. So for many cases, it probably better to use the tree for 2 dimentions, and then naively handling the 3rd. Then you dont suffer from the "curse of dimensionality"?

Pipelining. It might be possible to pipeline the process so that rebalancing and querying happen at the same time with the only downside being that bots react to their collisions one step later.

Another possible "improvement" would be to store positions and radius instead of bounding boxes.
That saves one extra float, but it is less versatile. Also fixing the radius of the bots would be a
huge performance improvement. Every bot would only need to store a position then.



# Testing correctness

Simply using rust has a big impact on testing. Because of its heavy use of static typing, many bugs are caught at compile time. This translates to less testing as there are fewer possible paths that the produced program can take. Also the fact that the api is generic over the underlying number type used is useful. This means that we can test the system using integers and we can expect it to work for floats. It is easier to test with integers since we can more easily construct specific scenarios where one number is one value more or less than another. So in this way we can expose corner cases.

A good test is a test that tests with good certainty that a large portion of code is working properly.
Maintaining tests comes at the cost of anchoring down the design of the production code in addition to having to be maintained themselves. As a result, making good abstractions between your crates and modules that have very simple and well defined apis is very important. Then you can have a few simple tests to fully excersise an api and verify large amounts of code.

This crate's sole purpose is to provide a method of providing collision detection. So a good high level test would be to compare the query results from using this crate to the naive method (which is much easier to verify is correct). This one test can be performed on many different inputs of lists of bots to try to expose any corner cases. So this one test when fed with both random and specifically tailed inputs to expose corner cases, can show with a lot of certainty that the crate is satisfying the api. 

The tailored inputs is important. For example, a case where two bounding boxes collide but only at the corner is an extremely unlikely case that may never present themselves in random inputs. To test this case, we have to turn to more point-directed tests with specific constructed set up input bot lists. They can still be verified in the same manner, though.

# Benching

Writing benches that validate every single piece of the algorithm design is a hassle ,although it would be nice. Ideally you dont want to rely on my word to say that, for example, using sweep and prune to find colliding pairs actually sped things up. It could be that while the algorithm is correct and fast that this particular aspect of the algorithm actually slows things down. 

So I dont think writing low level benches are worth it. If you are unsure of a piece of code, you can bench the algorithm as a whole, change a piece of the algorithm, and bench again and compare results. Because at the end of the day, we already tested the correctness, and that is the most important thing. So I think this strategy, coupled with code coverage and just general reasoning of the code can supplement tons of benches to validate the innards of the algorithm.



One thing to notice. Collision detection is expensive, and dominates the running time. So right off the bat, you know it is a waste of effort to allocate time into optimizing the linear parts of your program, when you could be spending that time optimizing the bottleneck.

Always measure code before investing time in optimizing. As you design your program. You form in your mind ideas of what you think the bottle necks in your code are. When you actually measure your program, your hunches can be wildly off.

When dealing with parallelism, benching small units can give you a warped sense of performance. Onces the units are combined, there may be more contention for work stealing. With small units, you have a false sense that the cpu's are not busy doing other things. For example, I parallalized creating the container range for each node. Benchmarks showed that it was faster. But when I benched the rebalancing as a whole, it was slower with the parallel container creation. So in this way, benching small units isnt quite as useful as testing small units is. That said, if you know that your code doesnt depend on some global resource like a threadpool, then benching small units is great.

Platform dependance. Rust is a great language that strives for platform independant code. But at the end of the day, even though rust programs will behave the same on multiple platforms, their performance might be wildly different. And as soon as you start optimizing for one platform you have to wonder whether or not you are actually de-optimizing for another platform. For example, rebalancing is much slower on my android phone than querying. On my dell xps laptop, querying is the bottle neck instead. I have wondered why there is this disconnect. I think part of it is that rebalancing requires a lot of sorting, and sorting is something where it is hard to predict branches. So my laptop probably has a superior branch predictor. Another possible reason is memory writing. Rebalancing involves a lot of swapping, whereas querying does involve in any major writing to memory outside of what the user decides to do for each colliding pair. In any case, my end goal in creating this algorithm was to make the querying as fast as possible so as to get the most consistent performance regardless of how many bots were colliding.

In fact, I ended up with 3 competing rebalancing algorithms. The first one would simply create pointers to all the bots, and sorted the pointers. This one was the slowest. I think it is because only one field is relevant to this algorithm, the bounding box rect. So all the other fields were just creating space that needed to be jumped over. So the distance between relevant information to be used by the algotihm wa high. On the other hand this method didnt have to allocate much memory. There is also the problem that its highly dependant on where the given slice is in memory. If its far away from the vec of pointers, then every deref is expensive, probably.

The second one would create a list of rects and ids pulled from the bots and sort that. The main characteristic of this method is that there is no layer of indirection. The downside is that swapping elements is more expensive since you are not swapping pointers, you are swapping bounding boxes coupled with an id. So this method made the median-finding part of the algorithm very fast, but made the sorting slower.

The third method was to create a list of rects, and then create a list of pointers to that list of rects and then sort that. The obvious downside is that you end up dynamically allocating two seperate vecs. But really it doesnt use any more memory that method 2. It has the benefit of swapping only pointers, and it also has better memory locality that method1.

For large numbers of bots 50,000+, the second method seems to be the best on both my phone and laptop.


# Dynamic Allocation

Sure dynamic allocation is "slow", but that doesn't mean you should avoid it. You should use it as a tool to speed up a program. It has a cost, but the idea is that with that allocated memory you can get more performance gains.

The problem is that everybody has to buy into the system for it to work. Anybody who allocated a bunch of memory and doesn't return it because they want to avoid allocating it again is hogging that space for longer than it needs it.

Writing apis that don't do dynamic allocation is tough and can be cumbursome, since you probably have to have the user give you a slice of a certain size. On the other hand this let's the user know exactly how much memory your crate needs. But users should have some faith that a dynamic allocation system is good enough for most purposes to justify a more complicated api where users have to manually manage memory. 


Dynamic allocation is fast. Dynamically allocating large vecs in one allocation is fast. Its only when you're dynamically allocting thousands of small objects does it become bad. Even then, probably the allocations are fast, but because the memory will likely be fragmented, iterating over a list of those objects could be very slow. Concepually, I have to remind myself that if you dynamically allocate a giant block, its simply reserving that area in memory. There isnt any expensive zeroing out of all that memory unless you want to do it. That's not to say the complicated algorithms the allocator has to do arn't complicated, but still relatively cheap.

Often times, its not the dynamic allocation that is slow, but some other residial of doing it in the first place. For example, dynamically allocating a bunch of pointers to an array, and then sorting that list of pointers. The allocation is fast, its just that there is no cache coherency. Two pointers in your list of pointers could very easily point to memory locations that are very far apart.


The thing is that if you don't use dynamic allocation, and you instead reserve some giant piece of memory for use of your system, then that memory is not taken advanage of when your system is not using it. It is wasted space. If you know your system will always be using it then sure it is fine. But I can see this system being used sometimes only 30 times a second. That is a lot of inbetween time where that memory that cannot be used by anything else. So really, the idea of dynamic allocation only works is everybody buys into the system. Another option is to make your api flexible enough that you pass is a slice of "workspace" memory, so that the user can decide whether to dynamically allocate it everytime or whatever. But this complicates the api for a very small portion of users who would want to not using the standard allocator.



# General thoughts on optimizing

Optimization is a great balancing act. Indirection, locality, memory vs computation, dividng and conquer. Every algorithm has unique properties. On top of the designing the algorithm, there is a whole nother level of questions when it comes to implementing said algorithm. How to best write the code for maintainability, readabilty, performance, simplicity in the api. Concurrency is the strangest of all. The theory and practice clash together.  Making desisions of when to stabailize and write tests, or when to capitalize on some new design oportunity, it goes on. Is it better to generate out less code with more branches, or more code with less branches.  Good design is especially important since the original author is imbuned with the enthusiasm of making something new, where as other people will TODO finish.