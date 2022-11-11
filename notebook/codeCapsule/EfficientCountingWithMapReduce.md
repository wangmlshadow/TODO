# Efficient counting with MapReduce

> [Efficient counting with MapReduce | Code Capsule](https://codecapsule.com/2010/04/15/efficient-counting-mapreduce/)

Counting with MapReduce seems straightforward. All what is needed is to map the pairs to the same intermediate key, and leave the reduce take care of counting all the items. But wait, what if we have millions of items? Then one reducer, that is to say one process on one computer, will be forced to handle millions of pairs at once. Nonetheless this is going to be very slow, and all the interest of having a cluster will be missed, but there is something more important: what if the data is too big to fit in memory? Here I am showing how to count elements using MapReduce in a way that really split up the task between multiple workers.

> ‎使用MapReduce似乎很简单。所有需要做的就是将对映射到相同的中间键，并让 reduce 负责计算所有项目。但是等等，如果我们有数百万需要被计算的单元呢？然后，一个化简器，即一台计算机上的一个进程，将被迫同时处理数百万个对。尽管如此，这将是非常缓慢的，并且将错过拥有集群的所有兴趣，但是还有更重要的事情：如果数据太大而无法放入内存怎么办？在这里，我展示了如何使用MapReduce来计算元素，以一种真正在多个工人之间分配任务的方式。‎

## The one-iteration solution

Let us have a look at the solution discussed above. This solution counts items in a data in only one MapReduce iteration. Note that the values are replace with the value  *1* . Indeed, as counting does not require to keep track of the values, they are all changed to a common simple value to simplify computations. This solution seems pretty sweet, except that as we can see on Figure 1, reducing all the pairs to the same intermediate key gives one reducer,  *and one reducer only* , a huge workload for counting the items. This can be efficient if the dataset is small. But there are cases in which the dataset is so big that it does not even fit into the memory of a single computer, or maybe it is so big that the computation on only one reducer is going to be very slow, and we need to know the count as soon as possible. As we will see in the next section, there is a way to improve workload balance along with computation time, at the cost of an additional iteration.

> ‎让我们看一下上面讨论的解决方案。此解决方案仅在一次 MapReduce 迭代中对数据中的项进行计数。请注意，这些值将替换为值 ‎*‎1‎*‎。实际上，由于计数不需要跟踪值，因此它们都更改为一个通用的简单值以简化计算。这个解决方案看起来非常有效，除了我们在图 1 中看到的那样，将所有对减少到相同的中间键会给一个化简器‎*‎和一个化简器‎*‎带来巨大的工作量来计算项目。如果数据集很小，这可能很有效。但在某些情况下，数据集是如此之大，以至于它甚至不适合一台计算机的内存，或者它可能太大，以至于只有一个化简器上的计算会非常慢，我们需要尽快知道计数。正如我们将在下一节中看到的那样，有一种方法可以改善工作负载平衡以及计算时间，但代价是额外的迭代。‎

[![MapReduce - Counting with one iteration](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_counting01.png?resize=700%2C620 "MapReduce - Counting with one iteration")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_counting01.png)
Figure 1: MapReduce - Counting with one iteration

## The two-iteration solution

The solution to the counting problem on a distributed system is to use an additional iteration, in order to better split up the workload. Instead of reducing all the pairs to a unique intermediate key, here the mappers reduce the pairs to intermediate keys taken sequentially out of a common set. It does not matter really what this set is, as long as the keys are the same for all the mappers, and are taken sequentially. The set of intermediate keys can be simply the series of integer numbers from 1 to n:  *(1, 2, 3, …, n)* . By using such a set, we guarantee that the workload will be correctly shared between several reducers, actually between as many reducers as they are intermediate keys. So the number of possible intermediate keys, denoted  *N* , have to be chosen carefully. Indeed, you don’t want *N* to be too small, because then the memory issues of the one-iteration solution might not be fixed, and you don’t want *N* to be too big, because you will end up running one reducer per pair, which is quite inefficient.

Of course, the input dataset is never partitioned exactly equally between the mappers, thus the first intermediate keys in the set will be associated with more values than the last keys of the set. However, the pairs will be distributed enough to improve workload balance. Figure 2 shows this computation, and we can see that Reducer 4, which handle all pairs of which the intermediate key is  *4* , get less pairs to handle than the other reducers. Then the reducers count the pairs directed to them. At this stage, we do not have the count that we want, only intermediate counts. We need to sum up these counts, which requires another iteration.

In Figure 2, we can see that the second iteration is very simple: it is the same as the iteration we used in the one-iteration solution. All pairs are reduced to the same intermediate keys. The previous iteration guarantees us that there are only *N* pairs, so by choosing *N* appropriately, the mapper and the reducer of this iteration will receive workloads adapted to the architecture and the available memory of they machines they are running on. Finally The output this second iteration is the count of all items.

> 分布式系统上计数问题的解决方案是使用额外的迭代，以便更好地分配工作负载。映射器不是将所有对简化为唯一的中间键，而是将对减少到从公共集中按顺序取出的中间键。这个集合是什么并不重要，只要所有映射器的键都相同，并且是按顺序获取的。中间键集可以简单地从 1 到 n：‎*‎（1， 2， 3， ...， n）‎*‎ 的整数序列。通过使用这样的集合，我们保证工作负载将在多个化简器之间正确共享，实际上在中间键的化简器数量之间正确共享。因此，必须仔细选择可能的中间键的数量，记作‎*‎N‎*‎。事实上，你不希望‎*‎N‎*‎太小，因为那样一次迭代解决方案的内存问题可能不会得到解决，你不希望‎*‎N‎*‎太大，因为你最终会为每对运行一个化简器，这是非常低效的。‎
>
> ‎当然，输入数据集永远不会在映射器之间精确平均分区，因此集合中的第一个中间键将与比集合的最后一个键更多的值相关联。但是，这些对的分布将足够大，以改善工作负载平衡。图 2 显示了此计算，我们可以看到，处理中间键为 4 的所有对的 Reducer ‎*‎4‎*‎ 比其他简化器获得的要处理的对更少。然后，化简器计算指向它们的对数。在这个阶段，我们没有我们想要的计数，只有中间计数。我们需要总结这些计数，这需要另一次迭代。‎
>
> ‎在图 2 中，我们可以看到第二次迭代非常简单：它与我们在一次迭代解决方案中使用的迭代相同。所有对都简化为相同的中间键。上一个迭代向我们保证只有 ‎*‎N‎*‎ 对，因此通过适当地选择 ‎*‎N‎*‎，此迭代的映射器和化简器将接收适合该体系结构的工作负载以及运行它们的计算机的可用内存。最后，第二次迭代的输出是所有项的计数。

[![MapReduce - Counting efficiently with two iterations](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_counting02.png?resize=710%2C1270 "MapReduce - Counting efficiently with two iterations")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_counting02.png)
Figure 2: MapReduce - Counting efficiently with two iterations

## An implementation of the two-iteration solution using Python and the Prince API

Prince is an API that allows Python programs to run on Hadoop Streaming. The Prince API is available here: [http://wiki.github.com/goossaert/prince/](http://wiki.github.com/goossaert/prince/). Using Python, we can implement the solution discussed in the previous section. Here is the source code for efficiently counting words in input data, that you can also download here: [totalcount.py](http://github.com/goossaert/prince/raw/master/examples/totalcount.py).

For simplicity, I have chosen to use a modulo to loop on intermediate keys. Remember that the modulo is a costly operation, and that alternatives exist to improve computation time in case optimization is required. The `count_mapper()` and `count_reducer()` methods are used for the first iteration, and `sum_mapper()` and `count_mapper()` methods are used for the second iteration. The `count_items()` method takes care of chaining the two iterations and of running the tasks.

> ‎Prince是一个API，允许Python程序在Hadoop Streaming上运行。Prince API 可在此处获取：‎[‎http://wiki.github.com/goossaert/prince/‎](http://wiki.github.com/goossaert/prince/)‎。使用Python，我们可以实现上一节中讨论的解决方案。以下是有效计算输入数据中单词的源代码，您也可以在此处下载：‎[‎totalcount.py‎](http://github.com/goossaert/prince/raw/master/examples/totalcount.py)‎。‎
>
> ‎为简单起见，我选择使用模来循环使用中间键。请记住，模是一个代价高昂的操作，并且在需要优化的情况下，存在替代方案来缩短计算时间。`count_mapper`和 `count_reducer`方法用于第一次迭代，`sum_mapper`和 `count_mapper`方法用于第二次迭代。`count_items`该方法负责链接两个迭代并运行任务。

```python
import sys
import prince
 
 
def count_mapper(key, value):
    """
Distribute the values over keys, equality enough to average computational
complexity for the reducers. By using a modulo, we are sure to balance the
number of values for each key in the reducers. Therefore, this method
avoids the case where a single reducer faces the whole data set.
"""
    nb_buckets = 100 # this value is correct for small- and medium-sized
                     # data sets, but should be adjusted to each case
    key = int(key)
    for index, item in enumerate(value.split()):
        yield (key + index) % nb_buckets, 1
 
 
def count_reducer(key, values):
    """Sum up the items of same key"""
    try: yield key, sum([int(v) for v in values])
    except ValueError: pass # discard non-numerical values
 
 
def sum_mapper(key, value):
    """Map all intermediate sums to same key"""
    (index, count) = value.split()
    yield 1, count
 
 
def count_items(input, output):
    """Sum all the items in the input data set"""
    # Intermediate file name
    inter = output + ‘_inter’
 
    # Run the task with specified mapper and reducer methods
    prince.run(count_mapper, count_reducer, input, inter, inputformat=‘text’, outputformat=‘text’)
    prince.run(sum_mapper, count_reducer, inter + ‘/part*’, output, inputformat=‘text’, outputformat=‘text’)
 
    # Read the output file and print it
    file = prince.dfs.read(output + ‘/part*’, first=1)
    return int(file.split()[1])
 
 
def display_usage():
    print ‘usage: %s input output’ % sys.argv[0]
    print ‘ input: input file on the DFS’
    print ‘ output: output file on the DFS’
 
 
if __name__ == "__main__":
    # Always call prince.init() at the beginning of the program
    prince.init()
 
    if len(sys.argv) != 3:
        display_usage()
        sys.exit(0)
 
    input = sys.argv[1]
    output = sys.argv[2]
 
    # Count all items in the input data set and print the result
    print ‘Total items:’, count_items(input, output)
```
