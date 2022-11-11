# Simulated annealing applied to the traveling salesman problem

> [Simulated annealing applied to the traveling salesman problem | Code Capsule](https://codecapsule.com/2010/04/06/simulated-annealing-traveling-salesman/)

Simulated annealing is an optimization technique that finds an approximation of the global minimum of a function. When working on an optimization problem, a model and a cost function are designed specifically for this problem. By applying the simulated annealing technique to this cost function, an optimal solution can be found. In this article, I present the simulated annealing technique, I explain how it applies to the traveling salesman problem, and I perform experiments to understand how the different parameters control the details of the search for an optimal solution. I also provide an implementation in Python, along with graphic visualization of the solutions.

> ‎模拟退火是一种优化技术，用于查找函数的全局最小值的近似值。在处理优化问题时，模型和成本函数是专门为此问题设计的。通过将模拟退火技术应用于此成本函数，可以找到最佳解决方案。在本文中，我将介绍模拟退火技术，解释它如何应用于旅行推销员问题，并进行实验以了解不同参数如何控制搜索最佳解决方案的细节。我还在Python中提供了一个实现，以及解决方案的图形可视化。‎

## Simulated annealing

On Wikipedia, we can read:

> The name and inspiration come from annealing in metallurgy, a technique involving heating and controlled cooling of a material to increase the size of its crystals and reduce their defects.

The computer version of simulated annealing mimics the metallurgy one, and finds lower levels of energy for the cost function. The algorithm begins with a high temperature, and slowly cools down to a low temperature. Cooling down is done simply by having a loop on a `temperature` variable, and by multiplying this variable by a number between 0 and 1 at every iteration:

> ‎在维基百科上，我们可以读到：‎
>
>> ‎这个名字和灵感来自冶金中的退火，这是一种涉及加热和控制冷却材料的技术，以增加其晶体的尺寸并减少其缺陷。‎
>>
>
> ‎模拟退火的计算机版本模仿冶金，并为成本函数找到较低水平的能量。该算法从高温开始，然后慢慢冷却到低温。冷却只需在变量temperature上有一个循环，并在每次迭代时将此变量乘以0到1之间的数字即可完成：

```
temperature = temperature_start
while temperature > temperature_end:
    compute cost values and update optimal solution
    temperature = temperature * cooling_factor
```

We want to minimize the value of the cost function. This function has parameters, so minimizing its values is done by finding the good set of parameters. An initial cost value is computed for a set of random parameters, and by modifying this set, we are going to find an optimal solution. In order to find new parameter sets, we are going to change one of the parameters of the cost function at random at every iteration. Then we need to check if the change has been fruitful, and for that we compare the cost values before and after the change. If the new cost value is smaller than the previous cost value (the previous being the best solution known so far), then the change in parameters is a gain, and therefore we keep it. But if the new cost value is bigger than the previous one, it is not directly rejected. This is what makes the interest of the simulated annealing technique compared to hill climbing, as it prevents the search from being stuck in local minima. Even if the new cost value is bigger than the previous one, it can still be kept with a certain probability. This probability is computed based on the difference between the new and the previous cost values, and on the temperature.

> ‎我们希望最小化成本函数的值。此函数具有参数，因此通过查找正确的参数集来最小化其值。计算一组随机参数的初始成本值，通过修改此参数，我们将找到最佳解。为了找到新的参数集，我们将在每次迭代时随机更改成本函数的一个参数。然后，我们需要检查更改是否富有成效，为此，我们比较更改前后的成本值。如果新的成本值小于以前的成本值（以前的成本值是迄今为止已知的最佳解决方案），则参数的变化是一种收益，因此我们保留它。但是，如果新的成本值大于前一个成本值，则不会直接拒绝。与爬山相比，这就是模拟退火技术的兴趣所在，因为它可以防止搜索停留在局部最小值中。即使新的成本值大于前一个成本值，它仍然可以以一定的概率保留。此概率是根据新的和以前的成本值之间的差异以及温度来计算的。‎

```
costnew = cost_function(...)
difference = costnew - costprevious
if difference < 0 or  e(-difference/temperature) > random(0,1):
    costprevious = costnew
```

We use the negative exponential as probability distribution (Kirkpatrick, 1984), and we compare the exponential value to a random number taken in the uniform distribution in the interval [0,1). By replacing these computations into the simulated annealing loop presented above, we obtain:

> ‎我们使用负指数作为概率分布（Kirkpatrick，1984），并将指数值与区间[0，1]中均匀分布中取的随机数进行比较。通过将这些计算替换到上面介绍的模拟退火回路中，我们得到：‎

```
costprevious = infinite
temperature = temperature_start
while temperature > temperature_end:
    costnew = cost_function(...)
    difference = costnew - costprevious
    if difference < 0 or  e(-difference/temperature) > random(0,1):
        costprevious = costnew
    temperature = temperature * cooling_factor
```

## The traveling salesman problem

The traveling salesman problem is a classic of Computer Science. In this problem, a traveling salesman has to visit all the cities in a given list. The difficulty is that he has to do that by visiting each city only once, and by minimizing the traveled distance. This is equivalent to finding a Hamiltonian cycle, which is NP-complete.

The traveling salesman problem is a minimization problem, since it consists in minimizing the distance traveled by the salesman during his tour. As the distance is what we want to minimize, it has to be our cost function. The parameters of this function are the cities in the list. Modifying a parameter of the function equals to changing the order of visit of the cities. Applying the simulated annealing technique to the traveling salesman problem can be summed up as follow:

1. Create the initial list of cities by shuffling the input list (ie: make the order of visit random).
2. At every iteration, two cities are swapped in the list. The cost value is the distance traveled by the salesman for the whole tour.
3. If the new distance, computed after the change, is shorter than the current distance, it is kept.
4. If the new distance is longer than the current one, it is kept with a certain probability.
5. We update the temperature at every iteration by slowly cooling down.

Moreover, two major optimizations can be used to speed up the computation of the distances:

1. Instead of recomputing the distance between two cities every time it is required, the distances between all pairs of cities can be precomputed in a table, and used thereafter. Actually, a triangular matrix is enough, as the distance between cities A and B is the same as the distance between B and A.
2. As a change in parameters consists in swapping two cities, it is useless to recompute the total distance for the whole tour. Indeed, only the distances changed by the swap should be recomputed.

> ‎旅行推销员问题是计算机科学的经典之作。在这个问题中，旅行推销员必须访问给定列表中的所有城市。困难在于，他必须通过只访问每个城市一次，并通过最小化旅行距离来做到这一点。这相当于找到一个哈密顿循环，它是NP完全的。‎
>
> ‎旅行推销员问题是一个最小化问题，因为它包括最小化销售人员在旅行期间旅行的距离。由于距离是我们想要最小化的，因此它必须是我们的成本函数。此函数的参数是列表中的城市。修改函数的参数等于更改城市的访问顺序。将模拟退火技术应用于旅行推销员问题可以总结如下：‎
>
> 1. ‎通过随机排列输入列表（即：使访问顺序随机化）来创建城市的初始列表。‎
> 2. ‎在每次迭代中，列表中都会交换两个城市。成本值是销售人员在整个行程中行驶的距离。‎
> 3. ‎如果在更改后计算的新距离短于当前距离，则保留该距离。‎
> 4. ‎如果新距离比当前距离长，则以一定的概率保持该距离。‎
> 5. ‎我们通过缓慢冷却来更新每次迭代的温度。‎
>
> ‎此外，可以使用两个主要优化来加快距离的计算速度：‎
>
> 1. ‎无需在每次需要时都重新计算两个城市之间的距离，可以在表中预先计算所有城市对之间的距离，并在此后使用。实际上，三角形矩阵就足够了，因为城市A和B之间的距离与B和A之间的距离相同。‎
> 2. ‎由于参数的变化包括交换两个城市，因此重新计算整个旅程的总距离是没有用的。实际上，只有交换所改变的距离才应该重新计算。‎

## Experiment

I have implemented simulated annealing using Python and the design described in the previous section. This implementation is available for download at the end of this article. The data I am using are GPS coordinates of 50 European cities. I am using an Intel Atom 1.6Ghz processor on Linux Ubuntu to run my experiments. Also, for all the experiments, I am using a constant seed for the random number generator, which is the value  *42* .
The first experiment that I perform uses only ten cities. Figure 1 shows the initial tour, formed by a random visit order. Figure 2 presents the optimal tour obtained using simulated annealing. A **32% improvement** is observed from the initial tour to the optimal tour, as distance goes from 12699 km down to 8588 km. This solution was found in  *2 seconds* . Figure 3 shows how the optimal solution improves over the course of the simulated annealing.

> ‎我已经使用Python和上一节中描述的设计实现了模拟退火。本文末尾提供了此实现的下载。我使用的数据是50个欧洲城市的GPS坐标。我在 Linux Ubuntu 上使用 Intel Atom 1.6Ghz 处理器来运行我的实验。此外，对于所有实验，我使用随机数生成器的常量种子，即值‎*‎42‎*‎。‎
> ‎我做的第一个实验只用了十个城市。图 1 显示了由随机访问顺序形成的初始游览。图2显示了使用模拟退火获得的最佳游图。从最初的游览到最佳旅行，观察到‎**‎32%的改善‎**‎，因为距离从12699公里下降到8588公里。该解决方案在‎*‎2秒‎*‎内找到。图3显示了在模拟退火过程中最优解如何改进。‎

[![Initial tour - 10 cities](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_init_f0.9900_s1e+50_e0.1000.png?resize=700%2C600 "Initial tour - 10 cities")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_init_f0.9900_s1e+50_e0.1000.png)
Figure 1. Initial tour - 10 cities

[![Optimal tour - 10 cities, 1 iteration](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_i1_f0.9900_s1e+50_e0.1000.png?resize=700%2C600 "Optimal tour - 10 cities, 1 iteration")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_i1_f0.9900_s1e+50_e0.1000.png)
Figure 2. Optimal tour - 10 cities, 1 iteration

[![Distance metrics - 10 cities, 1 iteration](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c10_i1_f0.9900_s1e+50_e0.1000-720x540.png?resize=720%2C540 "Distance metrics - 10 cities, 1 iteration")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c10_i1_f0.9900_s1e+50_e0.1000.png)
Figure 3. Distance metrics - 10 cities, 1 iteration

## Restarts

One of the drawbacks of simulated annealing is that by testing random changes, the solution can diverge very quickly. In order to solve this problem, one can start back the annealing procedure from a previous solution known to be good. This technique is know as  **restart** . The decision to restart can come from various conditions. For example, annealing can restart after the temperature has reached a certain state, or after a certain number of steps. Here I am implementing a restart that takes place when the temperature reaches its final state. The cost function parameters are then set to the best tour known at this moment, instead of a random list as it is done to initialize the first step. Figures 4 and 5 below show the same simulated annealing algorithm used in the previous section, except that ten restarts are used in the hope of finding a better solution. On Figure 5, the vertical green bars indicate when a restart takes place. The computation takes *14 seconds* and a better solution is found indeed, but the gain is small. With a **32% improvement** on one iteration, we only obtain a **34% improvement** on ten iterations. Only the cities #9 and #10 are swapped from the solution found in one iteration. Moreover, we can observe on Figure 5 that the optimal distance does not change after the 3^rd^ restart, which means that 70% of the computation is useless. Click on the figures to enlarge.

> ‎模拟退火的缺点之一是，通过测试随机变化，解决方案可以非常迅速地发散。为了解决这个问题，可以从以前已知的良好解决方案开始退火程序。此技术称为‎‎重新启动‎‎。重新启动的决定可能来自各种情况。例如，退火可以在温度达到一定状态后或经过一定步骤后重新启动。在这里，我正在实现一个重新启动，当温度达到其最终状态时发生。然后，将成本函数参数设置为此时已知的最佳游览，而不是在初始化第一步时设置随机列表。下面的图4和图5显示了与上一节中使用的相同的模拟退火算法，只是使用了十次重新启动以找到更好的解决方案。在图 5 中，绿色竖线指示何时重新启动。计算需要‎‎14秒‎‎，确实找到了更好的解决方案，但增益很小。一次迭代‎‎提高了 32%，‎‎十次迭代后仅‎‎提高了 34%‎‎。只有城市 #9 和 #10 从一次迭代中找到的解决方案中交换。此外，我们可以在图5中观察到，‎‎在第3次‎‎重启后，最佳距离没有变化，这意味着70%的计算是无用的。点击数字放大‎.

[![Optimal tour - 10 cities, 10 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_i10_f0.9900_s1e+50_e0.1000.png?resize=700%2C600 "Optimal tour - 10 cities, 10 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c10_i10_f0.9900_s1e+50_e0.1000.png)
Figure 4. Optimal tour - 10 cities, 10 iterations

[![Distance metrics - 10 cities, 10 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c10_i10_f0.9900_s1e+50_e0.1000-720x432.png?resize=720%2C432 "Distance metrics - 10 cities, 10 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c10_i10_f0.9900_s1e+50_e0.1000.png)
Figure 5. Distance metrics - 10 cities, 10 iterations

## Playing with the parameters

In the previous experiment, the *shortest distance* metric in Figures 3 and 5 shows a nice convergence pattern. However, due to the random swap of cities in the annealing loop, the *tested distance* metric does not present any real convergence pattern. This metric is at the center of the algorithm, as it is from there that new optimal solutions can appear. The first thing that I am looking for when I am testing optimization and artificial intelligence algorithms is some kind of convergence in the observed metric. I agree that here, a simple city swap can lead to dramatic changes in the total traveled distance, but not seeing any convergence at all is very confusing. It seems there is some convergence for the tested distance in Figure 3 between the steps 11000 and 12000, but what is happening before is very chaotic, which gives me the feeling that simulated annealing does not really know what it is doing here. My guess is that the parameters of the algorithm need to be adapted to the problem to solve. Therefore, in the next section I am performing more experiments, this time with 50 cities, in order to study the effect of parameter changes on the algorithm’s behavior.

> ‎在前面的实验中，图 3 和图 5 中的‎*‎最短距离‎*‎指标显示了一个很好的收敛模式。然而，由于退火回路中城市的随机交换，‎*‎测试的距离‎*‎度量没有呈现任何真正的收敛模式。此指标位于算法的中心，因为从那里可以出现新的最优解决方案。当我测试优化和人工智能算法时，我首先要寻找的是观察到的指标中的某种收敛。我同意，在这里，简单的城市交换可能会导致总行驶距离发生巨大变化，但根本没有看到任何收敛是非常令人困惑的。图3中测试的距离在步骤11000和12000之间似乎有一些收敛，但之前发生的事情非常混乱，这让我觉得模拟退火并不真正知道它在这里做什么。我的猜测是，算法的参数需要适应要解决的问题。因此，在下一节中，我将进行更多的实验，这次是针对50个城市，以研究参数变化对算法行为的影响。‎

### Cooling factor and starting temperature

In order to test the effects of the cooling factor and of the starting temperature, I run the simulated annealing for 50 cities on 10 iterations by changing only these two parameters. All other parameters stay constant. Figure 6 shows the same experiment for different cooling factor and starting temperature as follows:

> ‎为了测试冷却系数和起始温度的影响，我通过仅更改这两个参数，在10次迭代中对50个城市运行模拟退火。所有其他参数保持不变。图6显示了针对不同冷却系数和起始温度的相同实验，如下所示：‎

(a). Cooling factor = 0.95, Starting temperature = 1e+10, Improvement: 43%, Time: 1 s
(b). Cooling factor = 0.95, Starting temperature = 1e+50, Improvement: 43%, Time: 3 s
(c). Cooling factor = 0.99, Starting temperature = 1e+10, Improvement: 57%, Time: 4 s
(d). Cooling factor = 0.99, Starting temperature = 1e+50, Improvement: 55%, Time: 16 s

[![Cooling factor and starting temperature comparisons](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_comparison01.png?resize=720%2C630 "Cooling factor and starting temperature comparisons")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_comparison01.png)
Figure 6. Cooling factor and starting temperature comparisons

I am very pleased with this series of experiments, because I am able to show a clear convergence of the tested distance over the course of an iteration. The number of steps changes for each of the experiments here. Therefore, the nice convergence pattern observed in Figure 6(a) is also present in Figures 6(b), 6(c) and 6(d), it is just less obvious as the curves are compressed over the y-axis. So what do we learn from this experiment? First, we can see that pre-convergence phase formed by the steps occurring before convergence in the tested distance, and which was our problem in Figures 3 and 5, is longer when the starting temperature is *1e+50* than when it is  *1e+10* . This is clearly visible in Figures 6(b) and 6(d) against Figures 6(a) and 6(c), respectively. These steps are due to many random changes in the tour configuration, in hope of seeing a new optimal solution emerging. But using the curve of shortest distance, we can see that these pre-convergence phases are very unproductive, since except in the first iterations, no new optimal solutions are found while they are executed. Moreover, the execution time for the experiment with a starting temperature of *1e+50* is more important than with a starting temperature of  *1e+10* . My conclusion is that we are just loosing time trying compute random stuff, which wastes processor cycles and delay convergence towards the optimal solution. Therefore in the case of our problem, a lower value, like  *1e+10* , is to be preferred.

Another interesting feature emerges from Figures 6(a) and 6(c). As we can see, convergence of the *shortest distance* operates faster when the cooling factor is *0.99* than when it is  *0.95* . There are around 500 steps in an iteration with a cooling factor of *0.95* against 2500 steps in an iteration with a cooling factor of  *0.99* . Indeed, with a value of  *0.99* , cooling takes more time, and therefore the algorithm has more opportunities for testing random solutions during the pre-convergence phase, but also to find more precise solutions during the convergence phase. Of course this has a cost, and computation time is 4 seconds for a cooling factor of *0.99* whereas it is only 1 second for a cooling factor of  *0.95* , but here the additional computations are worth it as they allow to reach a **57% improvement** compared to the initial tour.

> ‎我对这一系列实验非常满意，因为我能够在迭代过程中显示测试距离的清晰收敛。此处每个实验的步骤数都会发生变化。因此，图6（a）中观察到的良好收敛模式也存在于图6（b），6（c）和6（d）中，它只是不太明显，因为曲线在y轴上被压缩。那么，我们从这个实验中学到了什么呢？首先，我们可以看到，在测试距离内收敛之前发生的步骤形成的预收敛阶段，这是我们在图3和图5中遇到的问题，当起始温度为‎*‎1e + 50‎*‎时比在‎*‎1e + 10‎*‎时更长。这在图6（b）和图6（d）中清晰可见，分别与图6（a）和图6（c）相对。这些步骤是由于游览配置中的许多随机更改，希望看到一个新的最佳解决方案出现。但是使用最短距离曲线，我们可以看到这些预收敛阶段是非常无效的，因为除了在第一次迭代中，在执行它们时没有找到新的最优解。此外，起始温度为‎*‎1e+50‎*‎的实验的执行时间比起始温度为‎*‎1e+10‎*‎的实验时间更重要。我的结论是，我们只是浪费时间尝试计算随机的东西，这会浪费处理器周期并延迟收敛到最佳解决方案。因此，在我们的问题中，较低的值（如 ‎*‎1e+10‎*‎）是首选。‎
>
> ‎另一个有趣的特征出现在图6（a）和6（c）中。正如我们所看到的，当冷却因子为‎*‎0.99‎*‎时，‎*‎最短距离‎*‎的收敛比在‎*‎0.95‎*‎时运行得更快。冷却系数为 ‎*‎0.95‎*‎ 的迭代中大约有 500 个步骤，而冷却系数为 ‎*‎0.99‎*‎ 的迭代中有 2500 个步骤。实际上，对于‎*‎值为0.99‎*‎，冷却需要更多的时间，因此该算法在预收敛阶段有更多的机会测试随机解，但也可以在收敛阶段找到更精确的解。当然，这是有成本的，对于‎*‎0.99‎*‎的冷却系数，计算时间为4秒，而对于‎*‎0.95‎*‎的冷却系数，计算时间仅为1秒，但在这里，额外的计算是值得的，因为它们允许与初始旅行相比‎**‎达到57%的改进‎**‎。‎

### Ending temperature

Now I would like to look into the importance of the ending temperature. Therefore as above, I am running the simulated annealing for 50 cities on 10 iterations, but this time only the ending temperature changes and all other parameters remain constant. During these experiments, cooling factor is at  *0.95* , and the starting temperature at  *10e+50* . Figure 7 shows the consequences on the simulated annealing:

> ‎现在我想看看结束温度的重要性。因此，如上所述，我在10次迭代中对50个城市运行模拟退火，但这次只有结束温度变化，所有其他参数保持不变。在这些实验中，冷却因子为‎*‎0.95‎*‎，起始温度为‎*‎10e+50‎*‎。图7显示了对模拟退火的影响：‎

(a). Ending temperature = 0.001, Improvement: 60%, Time: 4 s
(b). Ending temperature = 0.01, Improvement: 58%, Time: 4 s
(c). Ending temperature = 0.1, Improvement: 57%, Time: 4 s
(d). Ending temperature = 1, Improvement: 53%, Time: 4 s

[![Ending temperature comparisons](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_comparison02.png?resize=720%2C630 "Ending temperature comparisons")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_comparison02.png)
Figure 7. Ending temperature comparisons

The pre-convergence and convergence phases are mostly the same for the *tested distance* across all experiments, but there is a detail that is worth noticing. It seems that as the ending temperature decreases, convergence gets more precise. Precision does not improves dramatically, but enough to refine the optimal solution found so far. The number of steps and the computation times are the same for all four experiments, however **improvement is of 53%** for an ending temperature of  *1* , **57%** for  *0.1* , **58%** for  *0.01* , and **60%** for  *0.001* . Even more interesting, the **60% improvement** in the case of *0.001* is reached during the first iteration only.

It seems that the starting and ending temperatures control the convergence pattern. In the previous section we saw that the starting temperature defines when the convergence starts, and here we see that the ending temperature controls the granularity of convergence, and allows to find solutions always more precise. But be careful, the ending temperature should not be too low. When testing an ending temperature of  *0.0001* , results are even better than for  *0.001* , but with *0.00001* results get worse. This is because if the ending temperature gets too low, then changes in configuration cannot operate correctly near the end of the annealing process. This causes the algorithm to remain stuck in some local minima.

> ‎对于所有实验的‎*‎测试距离‎*‎，预收敛和收敛阶段基本相同，但有一个细节值得注意。似乎随着结束温度的降低，收敛变得更加精确。精度不会显著提高，但足以完善迄今为止发现的最佳解决方案。所有四个实验的步数和计算时间都相同，但是对于结束温度为1，改善‎**‎率为53%，‎**‎对于0.1‎‎为‎**‎57%，‎**‎对于‎*‎0.01‎*‎为‎**‎58%，‎**‎对于‎*‎0.001‎*‎为‎**‎60%‎**‎。‎‎更有趣的是，在‎*‎0.001‎*‎的情况下，仅在第一次迭代中达到了‎**‎60%的改进‎**‎。‎
>
> ‎似乎起始和结束温度控制着收敛模式。在上一节中，我们看到起始温度定义了收敛开始的时间，在这里我们看到结束温度控制收敛的粒度，并允许始终更精确地找到解决方案。但要小心，结束温度不宜过低。当测试‎*‎0.0001‎*‎的结束温度时，结果甚至比‎*‎0.001‎*‎更好，但是‎*‎0.00001‎*‎的结果变得更糟。这是因为如果结束温度变得太低，则配置的变化无法在退火过程结束时正确运行。这会导致算法停留在某个局部最小值中。‎

## Final experiment: the quest for the optimal tour

Now that I have studied carefully the important of every parameter on the behavior of the algorithm, I want to apply this knowledge to find an optimal solution for our tour of Europe in 50 cities. I am going to keep a starting temperature of *1e+10* to avoid wasting computations, a cooling factor of 0.99 to increase computation time and hopefully finding better configurations. Also, I am going to set the ending temperature at *0.0001* in order to get a good granularity at the end of the convergence phase. Finally, I run the simulated annealing on up to 100 restart iterations!

Figures 8 through 12 represent the initial tour and the optimal tour after 1, 10, 50 and 100 iterations, respectively. The optimal tour slowly appears. Figure 13 shows the distance metrics for the 100 iterations. The restart locations have not been plotted. The optimal solution after 100 iterations represent a **63% improvement** compared to the initial configuration, in  *46 seconds* . Not too bad, and we can see on Figure 12 that the tour found after 100 iterations is indeed more simple than the initial one! Click on the figures to enlarge.

> ‎现在我已经仔细研究了算法行为上每个参数的重要性，我想应用这些知识来为我们在50个城市的欧洲之旅找到最佳解决方案。我将保持‎*‎1e + 10‎*‎的起始温度以避免浪费计算，冷却因子为0.99以增加计算时间，并希望找到更好的配置。另外，我将结束温度设置为‎*‎0.0001‎*‎，以便在收敛阶段结束时获得良好的粒度。最后，我在多达 100 次重启迭代上运行模拟退火！‎
>
> ‎图 8 到图 12 分别表示 1 次、10 次、50 次和 100 次迭代后的初始游览和最佳游览。最佳游览缓慢出现。图 13 显示了 100 次迭代的距离指标。尚未绘制重新启动位置。与初始配置相比，100 次迭代后的最佳解决方案在 ‎*‎46 秒‎*‎内‎**‎提高了 63%‎**‎。还不错，我们可以在图 12 中看到，在 100 次迭代后发现的导览确实比初始导览更简单！点击数字放大。‎

[![Initial tour - 50 cities](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_init_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Initial tour - 50 cities")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_init_f0.9900_s1e+10_e0.0001.png)
Figure 8. Initial tour - 50 cities

[![Optimal tour - 50 cities, 1 iteration](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i1_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Optimal tour - 50 cities, 1 iteration")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i1_f0.9900_s1e+10_e0.0001.png)
Figure 9. Optimal tour - 50 cities, 1 iteration

[![Optimal tour - 50 cities, 10 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i10_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Optimal tour - 50 cities, 10 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i10_f0.9900_s1e+10_e0.0001.png)
Figure 10. Optimal tour - 50 cities, 10 iterations

[![Optimal tour - 50 cities, 50 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i50_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Optimal tour - 50 cities, 50 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i50_f0.9900_s1e+10_e0.0001.png)
Figure 11. Optimal tour - 50 cities, 50 iterations

[![Optimal tour - 50 cities, 100 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i100_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Optimal tour - 50 cities, 100 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_cities_c50_i100_f0.9900_s1e+10_e0.0001.png)
Figure 12. Optimal tour - 50 cities, 100 iterations

[![Distance metrics - 50 cities, 100 iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c50_i100_f0.9900_s1e+10_e0.0001-720x432.png?resize=720%2C432 "Distance metrics - 50 cities, 100 iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/tsp_distances_c50_i100_f0.9900_s1e+10_e0.0001.png)
Figure 13. Distance metrics - 50 cities, 100 iterations

## Conclusion

In this article, I presented simulated annealing, an optimization technique aimed at finding an approximation to the global minimum of a function. I explained how this technique applies to the traveling salesman problem, and I used “ *restart* “, a method supposed to help to find more precise solutions. After some tests, I managed to find a good set of parameters for the algorithm which is adapted to the problem to solve. Finally, I applied the algorithm and I found an optimal path in a 50-city tour.

Now that I have played with simulated annealing quite a bit, I am convinced that this technique can find optimal solutions for complex problems. But I also have seen that the parameters of the algorithm are very sensitive. A good set of parameters is likely to differ from one problem to another. Therefore, before applying simulated annealing to a new problem, my plan would be to reproduce the same experiments that I have done here for the starting and ending temperature, and for the cooling factor. This will help me to find the right spot, where the parameter values allow to find an optimal solution quickly.

Regarding the restart method, my feeling is that it still has to be refined. Indeed, Figures 5, 6, 7 and 13 show that tested distance goes back to high values every time a restart occurs. This means that the algorithm goes back to trying random solutions at every restart, which is almost the same as restarting the algorithm from a random state. Therefore I have serious doubts whether I am doing it right. I think that playing with the starting and ending temperature values could fix that. For instance, by implementing a decay of starting temperature at every new restart, one could limit the randomness observed in the pre-convergence phase of an iteration. Thus, in addition to the convergence observed within the boundaries of a single iteration, a global convergence should also appear, as new iterations are performed.

> ‎在本文中，我介绍了模拟退火，这是一种优化技术，旨在找到函数全局最小值的近似值。我解释了这种技术如何应用于旅行推销员问题，我使用了“‎*‎重新启动‎*‎”，这种方法应该有助于找到更精确的解决方案。经过一些测试，我设法为算法找到了一组很好的参数，这些参数适合于要解决的问题。最后，我应用了算法，在50个城市的旅行中找到了一条最佳路径。‎
>
> ‎现在我已经玩了相当多的模拟退火，我相信这种技术可以为复杂问题找到最佳解决方案。但我也看到算法的参数非常敏感。一组好的参数可能因问题而异。因此，在将模拟退火应用于新问题之前，我的计划是重现我在这里为起始和结束温度以及冷却系数所做的相同实验。这将有助于我找到正确的位置，其中参数值允许快速找到最佳解决方案。‎
>
> ‎关于重启方法，我的感觉是它仍然需要改进。实际上，图 5、6、7 和 13 显示，每次重新启动时，测试距离都会恢复到高值。这意味着该算法在每次重新启动时都会返回尝试随机解，这几乎与从随机状态重新启动算法相同。因此，我严重怀疑我是否做对了。我认为使用起始和结束温度值可以解决这个问题。例如，通过在每次新的重启时实现起始温度的衰减，可以限制在迭代的预收敛阶段观察到的随机性。因此，除了在单个迭代的边界内观察到的收敛之外，在执行新的迭代时，还应该出现全局收敛。‎

## Python implementation

You can download the source code for the Python implementation that I have used to perform the experiments in this article, [annealing.py](http://github.com/goossaert/algorithms/raw/master/simulated_annealing/annealing.py), and the input data, [cities.csv](http://github.com/goossaert/algorithms/raw/master/simulated_annealing/cities.csv). Also, plotting the figures will require that you install the **matplotlib** package. You can find more information about this package on the [official matplotlib website](http://matplotlib.sourceforge.net/).

## References

* [D. Bertsimas and J. Tsitsiklis. Simulated annealing.  *Statistical Science* , 8(1):10-15, 1993.](http://web.mit.edu/people/jnt/Papers/J045-93-ber-anneal.pdf)
* [S. Kirkpatrick. Optimization by simulated annealing: Quantitative studies.  *Journal of Statistical Physics* , 34(5):975-986, 1984.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.123.7607&rep=rep1&type=pdf)
* [Simulated Annealing on Wikipedia.](http://en.wikipedia.org/wiki/Simulated_annealing)
