java基础篇 之 再探内部类跟final

之前写过一篇文章：[从垃圾回收机制解析为什么局部内部类只能访问final修饰的局部变量以及为什么加final能解决问题](https://blog.csdn.net/qq_41907991/article/details/80691485)，经过这两天的学习，发现有些不对，必须再来捋一捋

先看之前的例子:

```java
/**
 * @author dmz
 * @date Create in 22:28 2019/5/19
 */
public class Test {
    public void test() {
        String i = "10";
        A a = new A() {
            @Override
            public void test() {
                System.out.println(i);
            }
        };
    }
}
interface A {

    void test();

}
```

之前我还在Eclipse的时候这段代码明明是报错的，就像这样：

![20180614131859895](C:\Users\dell\Desktop\20180614131859895.png)

明明会提醒一定要申明为final，不知道为什么到了idea上就不会了

不过这都不是很重要，不影响我们今天的分析。

我们今天直接编译下上面的java文件：

```
javac Test.java
```

javac编译一下，得到下面两个文件，我都直接用idea打开了：

```java
// 内部类字节码文件
class Test$1 implements A {
    Test$1(Test var1, String var2) {
        this.this$0 = var1;
        this.val$i = var2;
    }

    public void test() {
        System.out.println(this.val$i);
    }
}
// 从上面我们可以看到，内部类对象默认持有外部类对象的引用
// 并且还持有外部类定义的变量的引用或者说是值
```

```java
// 外部类字节码文件
public class Test {
    public Test() {
    }

    public void test() {
        // 可以看到在编译的时候，默认加上了final关键字
        final String var1 = "10";
        A var10000 = new A() {
            public void test() {
                System.out.println(var1);
            }
        };
    }
}
```

那么现在问题来了，为什么编译的时候，默认会加上final呢？

这样有什么必要吗？

我们知道final主要作用就是：

1. 对于基本数据类型，保证值不可变
2. 对于引用数据类型，保证引用不可变

那么现在问题就变成了，为什么局部内部类中引用的外部数据必须要”不可变“呢？

其实这也是一种折衷的实现，**究其原因还是因为局部内部类的生命周期跟方法中定义的数据生命周期不一致导致的**，这点我在之前的文章里也说过了。

那么为什么加final能解决生命周期不一致的问题呢？

这就要看看我们之前编译后的class文件了，局部内部类其实是复制了一份方法内的数据的引用到自身的属性上。

如果这个引用是final的话，就代表了它不可变，那么局部内部类在复制的过程中，风险是否就缩小了呢？

比如，虽然我跟你生命周期不一样，但是我知道你一定是“10”这个字符串，你不会发生改变，那么即使你死亡了，

那么即使你这个引用死亡了，也并不影响我。大家说，是不是这么个道理呢？