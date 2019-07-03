java读源码之 Queue源码分析（ArrayDeque）

> 除了并发应用（并发包下的代码我之后会专门写），Queue在JavaSE5中仅有的两个实现是LinkedList和PriorityQueue，它们的差异在于排序行为而不是性能。1.6时新增了一个实现ArrayDeque，LinkedList在之前我们已经介绍过了，LinkedList作为队列使用时，也是调用它的add等方法，来维护队列先进先出的特性罢了，这里就不多赘述了，这篇文章主要介绍下ArrayDeque，也就是优先级队列

[TOC]

##### 继承关系分析：

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable
```

我们可以看到，它主要就是继承了一个AbstractQueue，我们再看下AbstractQueue的继承关系：

```java
public interface Deque<E> extends Queue<E> 
```

AbstractCollection这个类就不多说了，封装了集合类的一些通用方法，我们主要看下Queue接口

```java
// 定义了队列的一些基本操作
public interface Queue<E> extends Collection<E> {
	//向队列中插入一个元素，并返回true
    //如果队列已满，抛出IllegalStateException异常
    boolean add(E e);

    //向队列中插入一个元素，并返回true
    //如果队列已满，返回false
    boolean offer(E e);
	
    //取出队列头部的元素，并从队列中移除
    //队列为空，抛出NoSuchElementException异常
    E remove();

    //取出队列头部的元素，并从队列中移除
    //队列为空，返回null
    E poll();

    //取出队列头部的元素，但并不移除
    //如果队列为空，抛出NoSuchElementException异常
    E element();

	//取出队列头部的元素，但并不移除
    //队列为空，返回null
    E peek();
}
```

##### 构造函数分析：

```java
// 空参构造，就是创建了一个长度为16的数组
public ArrayDeque() {
    elements = new Object[16];
}

public ArrayDeque(int numElements) {
    // 根据参数创建一个合适容量的数组
    allocateElements(numElements);
}

public ArrayDeque(Collection<? extends E> c) {
    // 根据参数创建一个合适容量的数组
    allocateElements(c.size());
    // 再将集合中的元素全部添加到队列
    addAll(c);
}
```

```java
//ArrayDeque 对数组的大小(即队列的容量)有特殊的要求，必须是 2^n。通过 allocateElements方法计算初始容量
private void allocateElements(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    elements = new Object[initialCapacity];
}
```

上面这个算法很巧妙，也是这个类里面最有营养的一段了，我们来分析下：

首先，我们要知道它是做什么的，说白了就是根据传入的numElements元素数量，找到一个大于numElements的最小的2的n次方的数

其次，我们来分析下，为什么经过5次移位已经或运算后能实现这个功能，这里我们也用图来描述下：



##### 字段分析：

```java
//用数组存储元素
transient Object[] elements; // non-private to simplify nested class access
//头部元素的索引
transient int head;
//尾部下一个将要被加入的元素的索引
transient int tail;
//最小容量，必须为2的幂次方
private static final int MIN_INITIAL_CAPACITY = 8;
```