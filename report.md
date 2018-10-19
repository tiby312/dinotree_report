



The goal of this dinotree crate is to provide efficient broadphase collision querying. So let us look at some cold hard statistic on how it performs. 

# Test Setup

Before we can measure and compare performance of this algorithm, we have to come up with a good way to test it. We often want to see how performance degrades as the size of the problem increasing, but we also do not want to influence any other variables. In this way a spiral distribution of the bots is ideal. It allows us to grow the size of the problem without effecting the density of the bots. It also fills out the entire 2d space. 

The spiral distribution takes 3 inputs: 
n: the number of bots
horizontal_grow: the seperation between the bots as they are laied out.
spiral_grow: the rate at which the bots grown outward from the center.


We increase n to increase the size of the problem.
We can increase the spiral_grow to decrease the number of bots intersecting.

![chart](./graphs/spiral_visual.png)

The below chart shows as influencing the spiral_grow effects the number of bot itersections.

![chart](./graphs/spiral_data.png)















blablabls


![chart](./graphs/colfind_float_vs_integer.png)


blablabla

![chart](./graphs/colfind_height_heuristic.png)
