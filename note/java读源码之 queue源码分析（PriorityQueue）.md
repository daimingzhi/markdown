java读源码 之 queue源码分析（PriorityQueue，附图，希望大家一起交流）

##### 继承关系分析：

```java
public class PriorityQueue<E> extends AbstractQueue<E>    implements java.io.Serializable 
```

这里我们对比下之前学习过的ArrayDeque:

![pque01](H:\markdown\images\pque01.png)

可以发现，最大的区别就是ArrayDeque实现了Deque接口，也就是说ArrayDeque不仅仅是队列而且还是双端对了，但是PriorityQueue只能做队列使用，PriorityQueue的最大好处在于它可以排序

##### 构造函数分析：

```java
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
```

```java
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```

```java
public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}
```

```java
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

```java
public PriorityQueue(Collection<? extends E> c) {
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        initElementsFromCollection(ss);
    }
    else if (c instanceof PriorityQueue<?>) {
        PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        initFromPriorityQueue(pq);
    }
    else {
        this.comparator = null;
        initFromCollection(c);
    }
}
```

```java
public PriorityQueue(PriorityQueue<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initFromPriorityQueue(c);
}
```

```java
public PriorityQueue(SortedSet<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initElementsFromCollection(c);
}
```

阅读上面的代码，我们可以将构造函数分为两大类，一类是直接通过参数指定容量跟比较器的（若未指定比较器则按自然排序规则进行排序），前4个构造函数就是这种形式，另一种是通过参数传入一个集合，创建一个优先级队列的。我们主要分析第二种情况。

第二种情况主要是以下三个方法：

1. initElementsFromCollection，从名字中我们就知道，它是从集合中初始化元素，并且它的调用前提是在参数是一个SortedSet的情况，我们来看下它的实现

```java
// 非常简单，只是进行了一次数组的拷贝
private void initElementsFromCollection(Collection<? extends E> c) {
    Object[] a = c.toArray();
    // If c.toArray incorrectly doesn't return Object[], copy it.
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, a.length, Object[].class);
    int len = a.length;
    // 可以看出不允许出现为null的元素
    if (len == 1 || this.comparator != null)
        for (int i = 0; i < len; i++)
            if (a[i] == null)
                throw new NullPointerException();
    this.queue = a;
    this.size = a.length;
}
```

2. initFromPriorityQueue

```java
// 直接将作为参数的队列
private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
    if (c.getClass() == PriorityQueue.class) {
        this.queue = c.toArray();
        this.size = c.size();
    } else {
        initFromCollection(c);
    }
}
```

3. initFromCollection

```java
private void initFromCollection(Collection<? extends E> c) {
    initElementsFromCollection(c);
    heapify();
}
```

