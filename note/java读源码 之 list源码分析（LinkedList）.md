# java读源码 之 list源码分析（LinkedList）

[TOC]

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

1. **add,我们主要看这两个重载的方法**

   ```java
   // 往末尾添加元素
   public boolean add(E e) {
           linkLast(e);
           return true;
   }
   // 往头部添加
   public void addFirst(E var1) {
           this.linkFirst(var1);
   }
   // 往指定位置添加元素
   public void add(int index, E element) {
       // 判断角标是否越界
       checkPositionIndex(index);
   	// 说明是往尾部添加元素
       if (index == size)
           linkLast(element);
    else
       // 往集合中添加元素    
        linkBefore(element, node(index));
   }
   ```
   
   跟踪方法到，linkLast(e)。
   
   ```java
   void linkLast(E e) {
       // 申明一个Node,保存last的引用
       final Node<E> l = last;
       // 创建一个新节点，pre指向集合的last,next指针指向null,也即是集合的尾指针
       final Node<E> newNode = new Node<>(l, e, null);
       // 集合的尾指针指向新节点
       last = newNode;
    if (l == null)
           // 说明集合刚刚初始化，一个元素都没有，添加的是第一个元素
        first = newNode;
       else
           // 集合中原来的最后一个节点的next指针指向了新节点
           l.next = newNode;
       // 集合长度加1
       size++;
       // 集合修改次数加1
       modCount++;
   }
   ```
   
   用图形的方式我们分析下这个过程：
   
   假设我们现在有一个三个元素的集合，如下：![linklist](H:\markdown\images\linklist.png)
   
   现在我们要调用add方法往集合的尾部添加一个元素：
   
   1. 创建一个节点 
   
   ![node](H:\markdown\images\node.png)
   
   2. 需要将这个节点的pre指针指向当前集合的最后一个元素，同时将当前集合的最后一个元素的next指针指向这个新增的节点
   
      ![add](H:\markdown\images\add.png)
   
   3. 不要忘了，在添加后，我们集合的last指针也变成了我们新增的这个元素的next指针
   
   上面我们分析了往尾部添加一个元素的情况，往头部添加一个元素就不多追诉了，那么往集合中添加一个元素呢？
   
   我们先看下源码：
   
   首先调用了node方法
   
   ```java
   Node<E> node(int index) {
       // 这里主要是判断是离头部比较近还是离尾部比较近
       // 如果头部比较近就从头节点开始搜索，否则就尾节点开始搜索
       if (index < (size >> 1)) {
           Node<E> x = first;
           for (int i = 0; i < index; i++)
               // 一直搜索到指定位置
               x = x.next;
           // 然后返回这个指定位置上的元素
           return x;
       } else {
           Node<E> x = last;
           for (int i = size - 1; i > index; i--)
               x = x.prev;
           return x;
       }
   }
   ```
   
   之后调用linkBefore方法
   
   ```java
   // e：要添加的节点
   // succ: 当前位置上的节点
   // 这个方法主要就是将e添加到succ节点的前一个节点位置上
   void linkBefore(E e, Node<E> succ) {
       // assert succ != null;
       // 用pred指针指向succ的前一个节点
       final Node<E> pred = succ.prev;
       // 创建一个新节点，pre指针指向succ的前一个节点，next指针指向succ，节点元素为e
       final Node<E> newNode = new Node<>(pred, e, succ);
       // succ的pre指针指向新增的节点
       succ.prev = newNode;
       // 如果succ.prev==null，说明succ是头节点，所以添加后，我们新增的节点就是头节点
       if (pred == null)
           first = newNode;
       else
        // 否则的话，就将succ的前一个节点的next指针指向新增的节点
           pred.next = newNode;
       // 集合长度+1
       size++;
       // 修改次数加+1
       modCount++;
   }
   ```

​	我们也用图形的方式来描述下这个过程：

​	假设我们现在有一个三个元素的集合，如下：![linklist](H:\markdown\images\linklist.png)

我们要在index=1的位置上新增一个元素，也就是调用add(1,"新元素")

1. 我们要获取到index=1的这个位置上的元素，通过源码我们知道，会通过头节点遍历到第二个节点后，返回给我们这个节点，也就是node方法

   ![inde=1](H:\markdown\images\inde=1.png)

2. 因为我们要在index=1的位置上插入元素，所以，我们要将index=0位置上的元素的next指针指向我们新增的元素，同时我们要新增的节点的pre指针指向头节点，另外，新增节点的next指针指向原index=1上位置的节点，index=1位置上的节点的pre指针指向我们新增的节点

![addbefore](H:\markdown\images\addbefore.png)

2. **get方**法

   ```java
   public E get(int index) {
       checkElementIndex(index);
       // get就是调用我们之前分析过的node方法
       // 获取指定位置上的节点，然后返回节点保存的元素
       return node(index).item;
   }
   ```

3. **对比ArrayList我们也分析下迭代器**

   跟踪源码我们可以发现，核心就是下面这个内部类

   ```java
   private class ListItr implements ListIterator<E> {
       // 记录当前迭代器最后一个返回的节点
       private Node<E> lastReturned;
       // 下一个迭代的节点
       private Node<E> next;
       // 下一个迭代的节点的索引位置
       private int nextIndex;
       // 快速失败机制
       private int expectedModCount = modCount;
   	// 创建一个从指定位置开始迭代的迭代器
       ListItr(int index) {
           // assert isPositionIndex(index);
           next = (index == size) ? null : node(index);
           nextIndex = index;
       }
   	
       public boolean hasNext() {
           return nextIndex < size;
       }
   	// 返回准备迭代的元素
       public E next() {
           checkForComodification();
           if (!hasNext())
               throw new NoSuchElementException();
   		// 记录返回的节点
           lastReturned = next;
           // 记录迭代器将要迭代的下一个节点
           next = next.next;
           // 索引+1
           nextIndex++;
           return lastReturned.item;
       }
   
       public boolean hasPrevious() {
           return nextIndex > 0;
       }
   
       public E previous() {
           checkForComodification();
           if (!hasPrevious())
               throw new NoSuchElementException();
   
           lastReturned = next = (next == null) ? last : next.prev;
           nextIndex--;
           return lastReturned.item;
       }
   
       public int nextIndex() {
           return nextIndex;
       }
   
       public int previousIndex() {
           return nextIndex - 1;
       }
   	// 分析下这个方法中的unlink方法
       public void remove() {
           checkForComodification();
           if (lastReturned == null)
               throw new IllegalStateException();
   
           Node<E> lastNext = lastReturned.next;
           // 移除上一次迭代的节点
           unlink(lastReturned);
           // 说明通过previous()方法迭代
           if (next == lastReturned)
               // 这个时候nextIndex不用发生变化
               next = lastNext;
           else
               // 因为移除了一个元素，所以nextIndex需要减1
               nextIndex--;
           lastReturned = null;
           expectedModCount++;
       }
   
       public void set(E e) {
           if (lastReturned == null)
               throw new IllegalStateException();
           checkForComodification();
           lastReturned.item = e;
       }
   
       public void add(E e) {
           // 快速失败机制检查
           checkForComodification();
           lastReturned = null;
           if (next == null)
               linkLast(e);
           else
               linkBefore(e, next);
           nextIndex++;
           expectedModCount++;
       }
   ```

   

   ```java
   // 用于将元素从集合中移除
   // 其实做的就是，将这个节点的pre指针指向的节点的next指针指向这个节点的next指针指向的节点
   // 同时将这个节点的next指针指向的节点的pre指针指向这个节点的pre指针指向的节点
   E unlink(Node<E> x) {
       // assert x != null;
       final E element = x.item;
       final Node<E> next = x.next;
       final Node<E> prev = x.prev;
   
       if (prev == null) {
           first = next;
       } else {
           prev.next = next;
           x.prev = null;
       }
   
       if (next == null) {
           last = prev;
       } else {
           next.prev = prev;
           x.next = null;
       }
   
       x.item = null;
       size--;
       modCount++;
       return element;
   }
   ```

   图片描述如下：

![duan](H:\markdown\images\duan.png)