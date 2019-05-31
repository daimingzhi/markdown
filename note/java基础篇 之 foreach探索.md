java基础篇 之 foreach探索

我们看下这段代码：

```java
public class Main {
    public static void main(String[] args) {
        List list = new ArrayList();
        list.add(1);
        list.add(2);
        list.add(3);
        for (Object o : list) {
            System.out.println(o);
        }
    }
}
```

我们知道，foreach底层采用的是迭代器进行实现的，所以我们将文件编译，

执行命令：`javac Main.java`

可以得到如下的字节码文件：

```java
public class Main {
    public Main() {
    }

    public static void main(String[] var0) {
        ArrayList var1 = new ArrayList();
        var1.add(1);
        var1.add(2);
        var1.add(3);
        Iterator var2 = var1.iterator();

        while(var2.hasNext()) {
            Object var3 = var2.next();
            System.out.println(var3);
        }

    }
}
```

这样也就验证了，foreach确实底层采用的是Iterator进行实现的。

现在问题来了，我们知道默认的foreach只能向前迭代，如果我们想使用向后的便利的foreach该怎么办呢？既然我们知道了并且验证了foreach底层使用的是iterator，那么我们能否能自定义一个呢？

代码如下:

```java
/**
 * @author dmz
 * @date Create in 12:28 2019/5/24
 */
public class MyList extends ArrayList {
    @Override
    public Iterator iterator() {
        System.out.println("我要向后走");
        return new Iterator() {
            int current = size()-1;

            @Override
            public boolean hasNext() {
                return current > -1;
            }

            @Override
            public Object next() {
                return get(current--);
            }
        };
    }
}
```

执行结果如下：

```
我要向后走
3
2
1
```

