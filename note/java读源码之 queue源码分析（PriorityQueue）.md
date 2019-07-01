java读源码之 queue源码分析（LinkedList，PriorityQueue）

> 除了并发应用，Queue在JavaSE5中仅有的两个实现是LinkedList和PriorityQueue，它们的差异在于排序行为而不是性能。LinkedList在之前我们已经介绍过了，在这里我再稍微介绍下它作为队列的实现时的一些用法，而且我们要知道，LinkedList是一个双端队列，不过双端队列用的很少，我们主要还是介绍正常的队列

##### LinkedList:

LinkedList作为队列主要有下面几个方法：

