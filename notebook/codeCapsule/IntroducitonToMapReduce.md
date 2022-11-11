#  Introduction to MapReduce

> [Introduction to MapReduce | Code Capsule](https://codecapsule.com/2010/04/02/introduction-to-mapreduce/)

A few years back, thinking that you could have a cluster in your garage would have been crazy. Programming your own implementation of a reliable and powerful distributed system is feasible, but be ready to spend some months on it. Luckily, big companies and their need to handle increasing quantities of data led us to accessible solutions for cloud computing. The last groundbreaking solution in date, effective on clusters of cheap computers and developed by Google, is MapReduce. This article is yet another post on MapReduce, except that it is aimed at tech-savvy and non tech-savvy people, as it covers in details the different steps of a MapReduce iteration. It also explains how MapReduce is related to functional programming, why it enables parallel computing, and finally how the work is being distributed between workers during an iteration.

> ‎几年前，想在你的车库里弄有一个集群会很疯狂。对一个可靠而强大的分布式系统进行编程是可行的，但要准备好花几个月的时间在上面。幸运的是，大公司及其对处理越来越多的数据的需求使我们获得了可访问的云计算解决方案。迄今为止，由Google开发的一个突破性解决方案，在廉价计算机集群上有效，是MapReduce。这篇文章是MapReduce上的另一篇文章，除了它针对的是精通技术和非技术的人，因为它详细介绍了MapReduce迭代的不同步骤。它还解释了MapReduce如何与函数式编程相关，为什么它支持并行计算，以及最后工作如何在迭代过程中在工人之间分配。‎

## **What is MapReduce doing?**

MapReduce is a cloud computing framework developed by Google. It allows to compute operations on a cluster of computers, and therefore speed up computation time. It has been made possible by another technology developed at Google, the Google File System. As the GFS enables the storage of large quantities of data on a network of computers, it is possible to share files easily between different tasks on different computers. The idea behind MapReduce, as for any distributed system, is to split up the task to do into smaller subtasks, and have several machines compute these subtasks in parallel. MapReduce relies heavily on the GFS to partition the input data and share information between workers. For more details on MapReduce, like for instance how MapReduce uses GFS or how workers are communicating, please have look at the reference section at the end of this article.

> ‎MapReduce是由Google开发的云计算框架。它允许在计算机集群上计算操作，从而加快计算时间。它是由谷歌开发的另一种技术，谷歌文件系统实现的。由于 GFS 允许在计算机网络上存储大量数据，可以在不同计算机上的不同任务之间轻松共享文件。与任何分布式系统一样，MapReduce背后的想法是将要完成的任务拆分为更小的子任务，并让几台机器并行计算这些子任务。MapReduce严重依赖GFS对输入数据进行分区并在worker之间共享信息。有关MapReduce的更多详细信息，例如MapReduce如何使用GFS或工作人员如何通信，请查看本文末尾的参考部分。‎

## **Map and reduce operators in functional programming**

MapReduce is a programing paradigm, that allows to split up a task into smaller subtasks that can be executed in parallel and therefore run faster compared to a single computer execution. Programming using the MapReduce paradigm requires a bit of training, but once you get that pattern on your mind, things are quite straightforward. The idea of the map and reducer operators is not new, and was borrowed from functional programming.

A map operator, or  *mapper* , takes as input a set of items and an operation, and applies that operation on all the pairs, one by one. Suppose that the input is a set of numbers from one to six, and that the operation is the multiplication by the number two:

the input is: `<1, 2, 3, 4, 5, 6>,`
the ouput is: `<2, 4, 6, 8, 10, 12>.`

A reduce operator, or  *reducer* , takes as input a set of items and an operation, and applies that operation on all these items, reducing them to only one item. Suppose that the input is a set is of numbers from one to six, and the operation is the sum:

the input is: `<1, 2, 3, 4, 5, 6>,`
the ouput is: `<21>.`

> ‎MapReduce是一种编程范式，它允许将任务拆分为更小的子任务，这些子任务可以并行执行，因此与单个计算机执行相比，运行速度更快。使用MapReduce范式进行编程需要一些培训，但是一旦你想到了这个模式，事情就非常简单了。map和reducer运算符的想法并不新鲜，而是从函数式编程中借用的。‎
>
> ‎map运算符或‎*‎映射器‎*‎将一组项目和一个操作作为输入，并将该操作逐个应用于所有对。假设输入是从 1 到 6 的一组数字，并且运算是乘以数字 2：‎
>
> ‎输入为：‎`<1, 2, 3, 4, 5, 6>,‎‎输出为：<2, 4, 6, 8, 10, 12>.`
>
> ‎reduce 运算符或 ‎*‎reducer‎*‎ 将一组项目和一个操作作为输入，并将该操作应用于所有这些项目，从而将它们减少到只有一个项目。假设输入是一组从 1 到 6 的数字，运算是总和：‎
>
> ‎输入为：‎`<1, 2, 3, 4, 5, 6>,‎‎输出为：<21>.`

## **Associativity is key to parallelism**

There is something interesting to notice about the map and reduce operators: they are  *associative* . Associativity is a notion of mathematics which applies to binary operators (here binary does not relates to the binary base, but to the fact that some operators take only two operands). For instance, the + operator is a binary operators, as it takes two operands at a time. It is also associative, because the order in which the operations are performed does not matter:

(1 + 2) + 5 is the same as 1 + (2 + 5).

So suppose we have two computers. Using the associativity property, we can split up the above summation into two smaller subtasks:

`computer 1: task #1: 1 + 2 = 3,<br/>computer 2: task #2: 5,`

and then merge the intermediate results:

`computer 1: task #3: 3 + 5 = 8.`

Because the subtasks #1 and #2 are independent due to the associativity of the + operator, they can be computed on separate computers and their intermediate results can be merged later during another subtask, here subtask #3. If there is something you remember about parallel computation today, it has to be this:

**Associativity is key to parallelism.**

(Note: I shamelessly stole these words from my Algorithm Analysis professor at the University of Oklahoma).

Associativity is also the core idea behind MapReduce: map and reduce operations can be performed in any order on the input, as they are associative. This means that the workload can be divided among several computers. Thus, if we can express a computation only with of map and reduce operations, then we can split up the work in pieces and compute these pieces on multiple computers in parallel. This is how Google got a highly reliable distributed computing platform out of functional programming.

> ‎关于map和reduce运算符，有一些有趣的事情需要注意：它们是‎*‎关联的‎*‎。结合性是一种适用于二元运算符的数学概念（这里二进制与二进制基无关，而是与某些运算符仅取两个操作数的事实有关）。例如，+ 运算符是二元运算符，因为它一次需要两个操作数。它也是关联的，因为执行操作的顺序无关紧要：
>
> ‎（1 + 2） + 5 与 1 + （2 + 5） 相同。‎
>
> ‎假设我们有两台计算机。使用关联性属性，我们可以将上述求和拆分为两个较小的子任务：‎
>
> `computer 1: task #1: 1 + 2 = 3,computer 2: task #2: 5`
>
> ‎然后合并中间结果：‎
>
> `computer 1: task #3: 3 + 5 = 8.`
>
> ‎由于子任务 #1 和 #2 由于 + 运算符的关联性而是独立的，因此可以在单独的计算机上计算它们，并且它们的中间结果可以在以后的另一个子任务（此处为子任务 #3）中合并。如果你今天还记得关于并行计算的东西，那一定是这样的：‎
>
> **‎关联性是并行性的关键。‎**
>
> ‎（注意：我从俄克拉荷马大学的算法分析教授那里偷走了这些话）。‎
>
> ‎结合性也是MapReduce背后的核心思想：映射和约简操作可以在输入上以任何顺序执行，因为它们是关联的。这意味着工作负载可以在多台计算机之间分配。因此，如果我们只能用map和reduce运算来表示计算，那么我们可以将工作分成几部分，并在多台计算机上并行计算这些部分。这就是谷歌如何从函数式编程中获得高度可靠的分布式计算平台。‎

## **Map and reduce operators in MapReduce**

MapReduce is using the map and reduce operators a bit differently than in functional programming. Indeed, each item is not just a value, but a (key, value) pair. The key is important, because it is used to group the data. Moreover, MapReduce tasks are always formed by one map operation followed by one reduce operations. Therefore, to compute something with MapReduce, you need to specify your input data, your map operation, and your reduce operation.

A map operator takes an input pair, and produces as output a set of (key, value) pairs. These pairs are called the  *intermediate pairs* . The intermediate pairs are treated, and all pairs with the same keys are grouped together, whether or not they come from the same mapper. This creates a set where pairs are no longer simple (key, value) pair, but (key, <v1, v2, …>), as several pairs can have the same key.

A reduce operator takes as input a key and the set of values to which it is associated. The authors of the MapReduce paper say that “ *this allows [them] to handle lists of values that are too large to ft in memory* “.

Now let’s take an example to see how this apply in the real world. Suppose that our input is now a string, and that we want to count the number of *consonants* and *vowels. *Please consider that this example only intents to clarify the different steps of a MapReduce iteration, and not to offer the best efficiency. Figure 1 below shows the different steps for the string “CODECAPSULE”. First of all, each letter is considered as a value, and receives a unique number id for key: this forms the input data. These data are then partitioned and directed to three distinct mappers. These mappers treat each letter and determine whether it is a consonant or a vowel. Intermediate (key, value) pairs are outputted, still with the letter as value, but this time with either *consonant* or *vowel* for key. Then, a very important step is taking place: all pairs of similar keys are merged together. This means that all pairs with *consonant* keys are merged together, and all pairs with *vowel* keys are merged together. This guaranties that all pairs with similar keys will be treated by a single reducer. A simple way to implementing this is to use a hash function on the keys. The reducers then work on the new (key, values) pairs, and count the number of consonants and vowels. The output pairs of the reduce step, which are the number of letters in each of the two families, are merge to form the output data.

> ‎MapReduce使用map和reduce运算符的方式与函数式编程略有不同。实际上，每个项目不仅仅是一个值，而是一个（键，值）对。键很重要，因为它用于对数据进行分组。此外，MapReduce任务总是由一个map操作后跟一个reduce操作组成。因此，要使用MapReduce计算某些内容，您需要指定输入数据，映射操作和减少操作。‎
>
> ‎映射运算符采用输入对，并生成一组（键、值）对作为输出。这些对称为‎*‎中间对‎*‎。处理中间对，并将具有相同键的所有对组合在一起，无论它们是否来自同一映射器。这将创建一个集合，其中对不再是简单的（键，值）对，而是（键，<v1，v2，...>），因为几对可以具有相同的密钥。‎
>
> ‎reduce 运算符将键及其关联的值集作为输入。MapReduce论文的作者说，“‎*‎这允许[他们]处理太大而无法在内存中ft的值列表‎*‎”。‎
>
> ‎现在让我们举个例子，看看这在现实世界中是如何应用的。假设我们的输入现在是一个字符串，并且我们要计算‎*‎辅音‎*‎和‎*‎元音的数量。 ‎*‎请注意，此示例仅用于阐明 MapReduce 迭代的不同步骤，而不是提供最佳效率。下面的图 1 显示了字符串“CODECAPSULE”的不同步骤。首先，每个字母都被视为一个值，并接收键的唯一数字ID：这构成了输入数据。然后对这些数据进行分区并定向到三个不同的映射器。这些映射器处理每个字母并确定它是辅音还是元音。输出中间（键，值）对，仍然以字母作为值，但这次使用‎*‎辅音‎*‎或‎*‎元音‎*‎作为键。然后，一个非常重要的步骤正在发生：所有相似的密钥对都合并在一起。这意味着所有具有‎*‎辅音‎*‎键的对将合并在一起，并且所有具有‎*‎元音‎*‎键的对将合并在一起。这保证了所有具有相似密钥的配对将由单个减速器处理。实现此目的的一种简单方法是在键上使用哈希函数。然后，化简器处理新的（键、值）对，并计算辅音和元音的数量。将合并 reduce 步骤的输出对（即两个族中每个族中的字母数）以形成输出数据。‎

[![MapReduce - Counting consonants and vowels in a string](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_intro01.png?resize=700%2C890 "MapReduce - Counting consonants and vowels in a string")](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/04/mapreduce_intro01.png)
Figure 1: MapReduce - Counting consonants and vowels in a string

## Hadoop, an open-source implementation of MapReduce

Now that we have seen how cool MapReduce is, I have to tell you about a little drawback. MapReduce is a patented piece of software, and is not available for you to use. But don’t be sad, because the Apache Foundation, helped by Yahoo Inc., are working on an open source implementation of the MapReduce paradigm using Java, fast and reliable. This implementation, called  *Hadoop* , is available for free. So you can perform distributed computations, given that you have several computers that you can use to install Hadoop on.

> ‎现在我们已经看到了MapReduce有多酷，我必须告诉你一个小缺点。MapReduce是一款获得专利的软件，不供您使用。但不要难过，因为Apache基金会在雅虎公司的帮助下，正在使用Java快速可靠地实现MapReduce范式的开源。这个称为‎*‎Hadoop‎*‎的实现是免费提供的。因此，您可以执行分布式计算，因为您可以使用多台计算机来安装Hadoop。‎

## **References to go further**

* A great start on how to install the latest version of Hadoop is Michael Noll’s excellent tutorial, available [here](http://www.michael-noll.com/wiki/Running_Hadoop_On_Ubuntu_Linux_%28Multi-Node_Cluster%29).
* Another good start is a tutorial by Travis Hegner, in which he explains how he installed Hadoop on six computers under Ubuntu, available [here](http://www.travishegner.com/2009/06/hadoop-020-on-ubuntu-server-904-jaunty.html).
* Google released a publication regarding their conception of MapReduce, available [here](http://labs.google.com/papers/mapreduce.html).
