java读源码之 Queue源码分析（PriorityQueue）

> 除了并发应用（并发包下的代码我之后会专门写），Queue在JavaSE5中仅有的两个实现是LinkedList和PriorityQueue，它们的差异在于排序行为而不是性能。LinkedList在之前我们已经介绍过了，LinkedList作为队列使用时，也是调用它的add等方法，来维护队列先进先出的特性罢了，这里就不多赘述了，这篇文章主要介绍下PriorityQueue，也就是优先级队列

##### 继承关系分析：

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable
```

我们可以看到，它主要就是继承了一个AbstractQueue，我们再看下AbstractQueue的继承关系：

```java
public abstract class AbstractQueue<E>
    extends AbstractCollection<E>
    implements Queue<E>
```

AbstractCollection这个类就不多说了，封装了集合类的一些通用方法，我们主要看下Queue接口

```java
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

