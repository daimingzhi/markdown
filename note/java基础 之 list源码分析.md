java基础 之 list源码分析（ArrayList）

##### ArrayList:

###### 继承关系分析：

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

我们可以知道：

1. 继承了AbstractList

2. 实现了List接口

3. 实现了RandomAccess，这里举例说明下这个接口的作用，我们看一段代码：

   Collections类中的binarySearch方法：

   ```java
   public static <T>
   int binarySearch(List<? extends Comparable<? super T>> list, T key) {
   // 实现了RandomAccess接口或者集合长度小于5000
       if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
           return Collections.indexedBinarySearch(list, key);
       else
           return Collections.iteratorBinarySearch(list, key);
   }
   ```

   可以看到，如果实现了RandomAccess接口或者集合长度小于5000，会调用indexedBinarySearch，否则调用iteratorBinarySearch。

   我们来看下这个两个方法的区别：

   ```java
   int indexedBinarySearch(List<? extends Comparable<? super T>> list, T key) {
       int low = 0;
       int high = list.size()-1;
   
       while (low <= high) {
           int mid = (low + high) >>> 1;
           // 在这里采用的是直接通过下标获取指定元素的方式
           Comparable<? super T> midVal = list.get(mid);
           int cmp = midVal.compareTo(key);
   
           if (cmp < 0)
               low = mid + 1;
           else if (cmp > 0)
               high = mid - 1;
           else
               return mid; // key found
       }
       return -(low + 1);  // key not found
   }
   ```

   ```java
   private static <T>
   int iteratorBinarySearch(List<? extends Comparable<? super T>> list, T key)
   {
       int low = 0;
       int high = list.size()-1;
       // 这里采用的是迭代器的方式
       ListIterator<? extends Comparable<? super T>> i = list.listIterator();
       while (low <= high) {
           int mid = (low + high) >>> 1;
           Comparable<? super T> midVal = get(i, mid);
           int cmp = midVal.compareTo(key);
   
           if (cmp < 0)
               low = mid + 1;
           else if (cmp > 0)
               high = mid - 1;
           else
               return mid; // key found
       }
       return -(low + 1);  // key not found
   }
   ```

   我们查看源码可以发现，**ArrayList实现了RandomAccess接口，所以ArrayList会通过索引直接访问，而LinkedList没有实现RandomAccess接口，会采用迭代器的方式访问，这主要是跟他们的数据结构有关，ArrayList底层是数组，LinkedList底层是链表**，具体的原因就不详细说了，留给读者自行思考，有问题可以留言一起讨论。

4. 实现了Cloneable，代表可以被克隆

5. 实现了Serializable，代表了可以被序列化

###### 属性分析：

```java
/**
 * 初始容量
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * 当采用public ArrayList(Collection<? extends E> c)这种构造函数时
 * 若传入的集合对象大小为0的话，此时用这个空数组作为这个ArrayList的元素
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 *当调用空参构造时，用这个空数组作为ArrayList的元素，虽然都是空数组，但是空参构造时
 *会采用默认容量DEFAULT_CAPACITY = 10，是为了区分开是哪种方式构造的这个ArrayList
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 *实际存储了ArrayList的元素的集合
 *这里可以思考一个问题：为什么要使用transient关键字修饰？
 *我会在后文中详细解释
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * 存储了的实际元素的数量，这里主要要跟容量区分开
 *我们可以这样理解，size是实际存的数量，而CAPACITY（容量）是打算存的数量
 */
private int size;

/**
 *可分配的最大数组长度，实际上最大可分配到Integer.MAX_VALUE，后面我们结合源码分析
 *减8是因为有些虚拟机需要存储一些头信息，稍后我们会分析下为什么要减8
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 *从父类AbstractList中继承而来的属性，记录了集合被修改的次数，主要为了实现快速失败机制
 *后面在方法分析中在详细解释
 */
protected transient int modCount = 0;
```

在上面的属性分析，我们遗留了一个问题，即elementData为什么要使用transient关键字修饰？

现在来详细解释下：

​		我们知道` transient`用来表示一个域不是该对象序列化的一部分，当一个对象被序列化的时候，transient修饰的变量的值是不包括在序列化的表示中的。但是ArrayList又是可序列化的类，elementData是ArrayList具体存放元素的成员，用transient来修饰elementData，岂不是反序列化后的ArrayList丢失了原先的元素？这里就要说到我们后面要说到的两个方法

- private void writeObject(java.io.ObjectOutputStream s)

- private void readObject(java.io.ObjectInputStream s)

  ​	对这两个方法的分析，我们放到后文中去，这里先说下，之所以这样设计主要是因为**elementData是一个缓存数组，它通常会预留一些容量，等容量不足时再扩充容量，那么有些空间可能就没有实际存储元素，采用上诉的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组，从而节省空间和时间。**

###### 构造函数分析：

```java
/**
 * 构造一个指定了初始容量的ArrayList，参数为负数的话，抛出IllegalArgumentException异常
 *
 * @param  initialCapacity  初始容量
 * 
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

/**
 * 空参构造，当第一次调用add方法时，将会采用默认值DEFAULT_CAPACITY=10作为初始容量
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 *构造一个包含了指定集合元素的ArrayList，如果集合元素为空，则此时ArrayList的容量也是0，
 *不会采用默认容量
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

###### 方法分析：

- add(E e)

```java
/**
 *添加指定的元素到集合末尾
 */
public boolean add(E e) {
    // 确保容量，继续跟踪这个方法
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
	// 在属性分析的适合我们已经说过了，如果是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，代表是通过
    // 空参构造创建的ArrayList,这个时候
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
	// 继续跟踪这个方法
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 要求的最小容量如果大于了现有的数组长度就进行扩容
    if (minCapacity - elementData.length > 0)
        // 继续跟踪这个方法
        grow(minCapacity);
}

private void grow(int minCapacity) {
    // 这里主要是做了一些溢出的考虑
    int oldCapacity = elementData.length;
    // 如果相加的和大于了int的最大值的话，这里就会得到一个负数，右移相当于模2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果是负数的话，相减肯定小于0
    if (newCapacity - minCapacity < 0)
    // 小于0的话，就将minCapacity（要求的最小容量，就是原有size+1）的值赋值给newCapacity(扩容后的	   // 容量)    
        newCapacity = minCapacity;
    // 如果扩容后的容量大于了 MAX_ARRAY_SIZE
    if (newCapacity - MAX_ARRAY_SIZE > 0)
    // 没有OOM发生的话，就干脆将newCapacity置为int的最大值
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

```

分析了add方法后，我们就可以对**ArrayList的扩容机制**有了一个很全面的了解：

- 第一次调用add后，如果是通过空参构造的话，默认会给一个10的初始容量

- 添加元素时，会判断要求的最小容量（size+1）是否超出了现有的数组长度，如果超出了要进行扩容

- 扩容时，会在原有容量的基础上进行1.5倍的扩容

- 如果扩容后的长度超出了int的最大值，就用size+1作为本次扩容后的容量

- 如果size+1大于了MAX_ARRAY_SIZE，就干脆用int的最大值作为容量

  **从上面也可以看出，如果add方法在添加的时候，不需要进行扩容的话，添加元素也是很快的，只需要将size+1上的指针指向指定元素就行了，如果涉及到扩容的话，性能就不高了，因为要移动一部分数组元素，并且添加元素的位置越靠前，移动的元素越多**

  - add(int index, E element)

  ```java
  // 在分析了add方法后，对于这个重载方法就不用花太多时间了
  public void add(int index, E element) {
      rangeCheckForAdd(index);
  ensureCapacityInternal(size + 1);  // Increments modCount!!
  // 这里相当于将数组从index位置开始到数据末尾的所有元素往后移动一位，然后将移动后数组上的index位置置为新添加的element元素
  System.arraycopy(elementData, index, elementData, index + 1,
                   size - index);
  elementData[index] = element;
  size++;
  ```
  }

  - clear()

  ```java
  // 这个方法非常简单，就是把所有的元素置为null,并且集合修改次数加1    
  public void clear() {
          modCount++;
  
          // clear to let GC do its work
          for (int i = 0; i < size; i++)
              elementData[i] = null;
  
          size = 0;
      }
  ```

  - ensureCapacity(int minCapacity)

  ```java
  // 在集合完成初始化后，调用进行手动扩容
  public void ensureCapacity(int minCapacity) {
      // 首先判断是通过什么方式初始化的，然后给一个初始容量
      int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
          ? 0
          : DEFAULT_CAPACITY;
      // 如果大于默认容量，进行扩容
      if (minCapacity > minExpand) {
          ensureExplicitCapacity(minCapacity);
      }
  }
  ```

  - E get(int index) 

    ```java
    public E get(int index) {
       // 检查是否角标越界
        rangeCheck(index);
    	// 直接从数组中获取index位置上的元素，效率很高
        return elementData(index);
    }
    ```

  - Iterator<E> iterator()

    ```java
    // 返回内部的迭代器
    public Iterator&lt;E&gt; iterator() {
        return new Itr();
    }
    ```

    我们接下来探究下迭代器的源码：

    ```java
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;
    
        public boolean hasNext() {
            return cursor != size;
        }
    
        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
    
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
    
            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    
        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }
    
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
    ```

  - ListIterator<E> listIterator()

    ```java
    // 返回另外一个迭代器
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }
    ```

    再看看这个迭代器的实现有什么区别？

    ```java
    // 继承了Itr
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }
    	
        // 可以向前遍历
        public boolean hasPrevious() {
            return cursor != 0;
        }
    
        public int nextIndex() {
            return cursor;
        }
    
        public int previousIndex() {
            return cursor - 1;
        }
    
        @SuppressWarnings("unchecked")
        public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }
    	
        // 新增了set方法
        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
    
            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    	// 新增了add方法
        public void add(E e) {
            checkForComodification();
    
            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
    ```



    **我们可以发现以上两个区别**：
    
    1. listIterator允许向前遍历
    2. listIterator允许在遍历的过程中添加元素