



The goal of this dinotree crate is to provide efficient broadphase collision querying. So let us look at some cold hard statistic on how it performs. 

# Test Setup

Before we can measure and compare performance of this algorithm, we have to come up with a good way to test it. We often want to see how performance degrades as the size of the problem increasing, but we also do not want to influence any other variables. In this way a spiral distribution of the bots is ideal. It allows us to grow the size of the problem without effecting the density of the bots. It also fills out the entire 2d space. 

The spiral distribution takes 3 inputs: 
n: the number of bots
horizontal_grow: the seperation between the bots as they are laied out.
spiral_grow: the rate at which the bots grown outward from the center.


We increase n to increase the size of the problem.
We can increase the spiral_grow to decrease the number of bots intersecting.

![chart](./graphs/spiral_visualize.png)

The below chart shows as influencing the spiral_grow effects the number of bot itersections.

![chart](./graphs/spiral_data.png)


# Comparison against other Algorithms


![chart](./graphs/colfind_theory.png)


![chart](./graphs/colfind_num_pairs.png)

![chart](./graphs/colfind_num_pairs_detailed.png)



# Rebalancing vs Querying
![chart](./graphs/colfind_rebal_vs_query_num_bots.png)

![chart](./graphs/colfind_rebal_vs_query_spiral.png)


# Level Comparison
![chart](./graphs/level_analysis_theory_rebal.png)
![chart](./graphs/level_analysis_theory_query.png)

![chart](./graphs/level_analysis_bench_rebal.png)
![chart](./graphs/level_analysis_bench_query.png)




# Comparison of Tree Height


![chart](./graphs/colfind_height_heuristic.png)
![chart](./graphs/colfind_height_heuristic_3d.png)
![chart](./graphs/colfind_optimal_height_vs_heuristic_height.png)
![chart](./graphs/colfind_heuristic_bench_vs_optimal_bench.png)


# Comparison of Parallel Height

![chart](./graphs/parallel_height_heuristic.png)

# Comparison of primitive types

![chart](./graphs/colfind_float_vs_integer.png)



