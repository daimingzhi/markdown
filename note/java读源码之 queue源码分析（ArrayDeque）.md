java读源码之 Queue源码分析（ArrayDeque，附详细的算法解释，希望跟大家一起交流）

> 除了并发应用（并发包下的代码我之后会专门写），Queue在JavaSE5中仅有的两个实现是LinkedList和PriorityQueue，它们的差异在于排序行为而不是性能。1.6时新增了一个实现ArrayDeque，LinkedList在之前我们已经介绍过了，LinkedList作为队列使用时，也是调用它的add等方法，来维护队列先进先出的特性罢了，这里就不多赘述了，这篇文章主要介绍下ArrayDeque

[TOC]

##### 继承关系分析：

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
```

我们可以看到，它主要就是实现了一个Deque，我们再看下Deque的继承关系：

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

第一步：

![ArrayDeque01](H:\markdown\images\ArrayDeque01.png)

第二步：

如果我们要找到比这个数大的最小的一个2的幂次方的数应该怎么做呢？

首先，我们知道，2^(n+1)>M>=2^n（最高位的1在第n位上），所以我们要得到的数，其实就是2^(n+1)

其次，我们要知道2^n+2^(n-1)+........+2^0=2^(n+1)-1

所以，我们要得到的数，其实就是将M低位全部补1然后+1即可。

那么如何对低位进行补1呢？这个时候，我们可以拿着最高位的1对余下的所有位数进行或运算，这样就可以将所有的低位都变成1，之后加1就得到到我们目标数。

图示如下：![arraydeque02](H:\markdown\images\arraydeque02.png)

再进行一次无符号右移，这次移动的位数是两位，因为此时我们能确保有两个1，现在要做的就是将这两个1右移两位，与原数中的值进行或运算，经过这样一次次右移以及或运算，最终能确保所有的第位都被用1补齐

![arraydeque03](H:\markdown\images\arraydeque03.png)

现在我们有4个1，所以下次我们要移动的位数是4，之后是8，再之后是16，因为int占32位，所有到16后可以确保所有的低位都已经用1补齐，之后加1，就得到我们想要的数了



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

##### 方法分析：

这里主要就分析下ArrayDeque对队列接口中一些方法的实现：

```java
public boolean add(E e) {    addLast(e);    return true;}
```

```java
public boolean offer(E e) {    return offerLast(e);}
```

```java
public E remove() {    return removeFirst();}
```

```java
public E poll() {    return pollFirst();}
```

```java
public E element() {    return getFirst();}
```

```java
public E peek() {    return peekFirst();}
```

1. addLast(E e)

```java
public void addLast(E e) {
    if (e == null)
        // 如果添加的元素为null，报NP
        throw new NullPointerException();
    // 加入元素
    elements[tail] = e;
    // 代表数组中元素已满
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

要理解上面这段代码，我们首先要知道的是ArrayDeque底层是一个循环数组，什么是循环数组呢？

假设我们现在有一个队列，初始容量为8![arraydeque04](H:\markdown\images\arraydeque04.png)

考虑一种临界情况，就是现在我们的队列还差一个元素就被添加满了，此时的head=0,tail=7![arraydeque05](H:\markdown\images\arraydeque05.png)

此时我们我们移除了队首的两个元素，队列变成了下面这个样子：![arraydeque06](H:\markdown\images\arraydeque06.png)

这个时候，我们再调用add方法添加一个元素，此时tail=8,并且这个条件`(tail = (tail + 1) & (elements.length - 1)) == head`是不满足的，也就是不会进行扩容，此时的tail会移动到队首来，也就是下面这个样子

![arraydeque07](H:\markdown\images\arraydeque07.png)

这就是循环数组了，我们可以将它想象成一个首尾相连的环形

接下来我们分析代码：

```java
if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
```

这段代码的意思是，如果数组中的元素满了，就进行双倍扩容。那么为什么`(tail = (tail + 1) & (elements.length - 1)) == head`条件成立时就代表队列满了呢？我们可以思考下。

我们可以这样想，当tail指针跟head指针重合时，队列是不是就满了？head指针指向的是队首的元素，而tail指针指向的是下一个要添加的元素的位置，如果下一个要添加的元素已经要覆盖队首的元素了，队列肯定已经满了。

那么接下来的问题就是，为什么`(tail = (tail + 1) & (elements.length - 1)) == head`就代表tail跟head指针重合了呢？

首先，我们要知道，当指针将要重合时，**tail+1-head=elements.length**，elements.length代表当前数组长度

稍微解释下：

第一次扩容时，tail-head应该等于还未扩容时数组的长度，举例：数组长度为8，head=0，tail=7,此时我们往队列末尾添加一个元素，因为此时tail+1=8,tail+1-head=8，满足了我们的要求，所以此时要进行扩容，扩容后数组长度16

第二次库容时，tail-head应该等于经过一次扩容后数组的长度，tail-head=16，举例：假设我们一直在往队列末尾添加元素没有移除元素，那么此时数组长度为16，head=0，tail=15,此时我们往队列末尾添加一个元素，此时tail+1=16,tail+1-head=16,满足我们要求，所以进行扩容，扩容后数组长度为32

..........

我们之前分析过了`elements.length`一定是2的幂次方的数，用二进制可以表示如下：

![arraydeque08](H:\markdown\images\arraydeque08.png)

`elements.length - 1`可以表示为：

![arraydeque09](H:\markdown\images\arraydeque09.png)

在我们调用addlast方法时，如果需要扩容，tail一定是大于等于length的，那么这个时候tail用二进制的方式表示

![arraydeque09](H:\markdown\images\arraydeque09.png)

我们不难发现，此时将tail的值跟elements.length - 1的值进行与运算，就是将tail的最高位的1置为0，也就是相当于tail=tail-2^n，满足了我们的**tail-head=elements.length**公式，此时需要扩容。

实际上这个操作，相当于在对length进行取模运算。

分析了扩容的触发情况了之后，我们再来看看扩容是怎么实现的

```java
// 进行双倍扩容
private void doubleCapacity() {
        assert head == tail;
    // 申明变量p用于保存头指针
        int p = head;
    // n保存当前数组长度
        int n = elements.length;
    // 从头指针到数组末尾元素的个数
        int r = n - p; 
    // 扩容后容量为原来的2倍
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
    // 第一次拷贝
        System.arraycopy(elements, p, a, 0, r);
    // 第二次拷贝
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    }

```

图解如下：

1. 假设我们队列当前的状态如下：

   ![arraydeque11](H:\markdown\images\arraydeque11.png)

2. 创建一个2倍长度的数组![arraydeque12](H:\markdown\images\arraydeque12.png)

3. 进行复制![arraydeque13](H:\markdown\images\arraydeque13.png)

4. 将head置为0，tail置为扩容前数组长度

至此，对于ArrayDeque中的add方法的整个流程我们就分析完了，继续我的方法分析

2. offerlast

   调用的就是addLast

3. removeFirst

   ```java
   public E removeFirst() {    E x = pollFirst();    if (x == null)        throw new NoSuchElementException();    return x;}
   ```

   核心就是：

   ```java
    public E pollFirst() {
           int h = head;
           @SuppressWarnings("unchecked")
           E result = (E) elements[h];
           // Element is null if deque empty
           if (result == null)
               return null;
           elements[h] = null;     // Must null out slot
           head = (h + 1) & (elements.length - 1);
           return result;
       }
   ```

   上面主要是对头指针的移动，跟尾指针的移动差不多，就不再分析了

4. 剩下的pollFirst,peekFirst,getFirst都差不多，就不多说了

**另外额外分析一个方法，size()**

```java
public int size() {    return (tail - head) & (elements.length - 1);}
```

如果让我们来实现计算一个循环数组中元素的个数，可能我们就会直接分两种情况了，一个是head小于tail，另外一种是head大于tail。而jdk中直接采用了位运算的方法，分析下其原理：

两种情况：

1. head<tail

![arraydeque14](H:\markdown\images\arraydeque14.png)

tail指针代表的是下一个元素将要添加进来的位置，所以此时数组中只有2，3号位置上有元素，此时数组size=tail-head=2，前面我们已经说过了，(tail - head) & (elements.length - 1)，这个与运算其实就相当于在对elements.length这个值进行模运算，大家仔细想一下应该不难理解，而此时length肯定是大于tail-head的，所以(tail-head)& (elements.length - 1)=(tail-head)mod(elements.length - 1)=tail-head=2

2. head>tail

![arraydeque15](H:\markdown\images\arraydeque15.png)

此时我们需要计数的元素应该是，（0，1，4，5，6，7），也就是非（2，3）的元素

这里我用颜色标明下：![arraydeque16](H:\markdown\images\arraydeque16.png)

我们已经知道了，(tail - head) & (elements.length - 1)=(tail - head) mod (elements.length)

当我们对一个负数进行模运算时，我们可以找到它对应的正数的对于elements.length同模数，这里给大家推荐一篇文章：https://blog.csdn.net/zl10086111/article/details/80907428

当tail-head<0时，不难找到它的同模数就算是tail-head+elements.length，也就是说(tail - head) & (elements.length - 1)=(tail - head) mod (elements.length)=(tail-head+elements.length) mod  (elements.length),对照上面的图，我们也很容易理解，就是非（2，3）的元素的个数，也就是我们的size属性

##### 迭代器分析：

```java
 private class DeqIterator implements Iterator<E> {

        private int cursor = head;


        private int fence = tail;


        private int lastRet = -1;

        public boolean hasNext() {
            return cursor != fence;
        }

        public E next() {
            if (cursor == fence)
                throw new NoSuchElementException();
            @SuppressWarnings("unchecked")
            E result = (E) elements[cursor];
 
            if (tail != fence || result == null)
                throw new ConcurrentModificationException();
            lastRet = cursor;
            cursor = (cursor + 1) & (elements.length - 1);
            return result;
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (delete(lastRet)) { 
                cursor = (cursor - 1) & (elements.length - 1);
                fence = tail;
            }
            lastRet = -1;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] a = elements;
            int m = a.length - 1, f = fence, i = cursor;
            cursor = f;
            while (i != f) {
                @SuppressWarnings("unchecked") E e = (E)a[i];
                i = (i + 1) & m;
                if (e == null)
                    throw new ConcurrentModificationException();
                action.accept(e);
            }
        }
    }
```

其实对于ArrayDeque代码中，最有意思以及最巧妙的就是它的位运算了，迭代器这块就不做过多分析了，留给大家自己思考，有什么问题可以一起探讨~~~

​		









