# java读源码 之 list源码分析（LinkedList）

##### LinkedList:

###### 继承关系分析：

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

这里的Cloneable，Serializable，List这三个接口就不多赘述了，之前在介绍ArrayList的时候已经说过了。主要分析下AbstractSequentialList跟Deque

1. AbstractSequentialList

   AbstractSequentialList 实现了get(int index)、set(int index, E element)、add(int index, E element) 和 remove(int index)这些函数。LinkedList是双向链表；既然它继承于AbstractSequentialList，就相当于已经实现了“get(int index)这些接口”

2. Deque

   实现了Deque接口，代表LinkedList能被当作双端队列使用

###### 字段分析：

```java
// 长度
transient int size = 0;

// 头节点
transient Node<E> first;

// 尾节点
transient Node<E> last;

// 继承了AbstractSequentialList，AbstractSequentialList又继承了AbstractList
protected transient int modCount = 0;

```

###### 构造函数分析：

```java
public LinkedList() {
}
```

```java
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

###### 方法分析：

1. add,我们主要看这两个重载的方法

   ```java
   // 往末尾添加元素
   public boolean add(E e) {
           linkLast(e);
           return true;
   }
   // 往指定位置添加元素
   public void add(int index, E element) {
       // 
       checkPositionIndex(index);
   
       if (index == size)
           linkLast(element);
       else
           linkBefore(element, node(index));
   }
   ```

   跟踪方法到，linkLast(e)。

   ```java
   void linkLast(E e) {
       final Node<E> l = last;
       final Node<E> newNode = new Node<>(l, e, null);
       last = newNode;
       if (l == null)
           first = newNode;
       else
           l.next = newNode;
       size++;
       modCount++;
   }
   ```

   用图形的方式我们分析下这个过程：

   假设我们现在有一个三个元素的集合，如下：
