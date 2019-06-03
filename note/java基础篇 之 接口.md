java基础篇 之 接口

##### 组合接口时的名字冲突：

看下面这段代码：

```java
interface I1 {
    int f();
}

interface I2 {
    void f();
}

interface I3 {
    int f(int a);
}

class C {
    public void f() {
        System.out.println(1);
    }
}

class C1 extends C implements I2 {

}
class C2 extends C implements I3 {
    @Override
    public int f(int a) {
        return 1;
    }
}
//这里报错了，是因为覆盖，实现跟重载被混在了一起
//C3继承了C，这个时候就有了一个void f()的方法
//C3实现了I1，就不得不实现它的int f()
//这个时候，C3就会出现两个同名函数，这显然是错误的
//class C3 extends C implements I1 {
//
//}
```

##### 接口中的域：

因为放入接口中的任何域都是自动static和final的，所以接口就成了一种很便捷的用来创建常量的工具。

```JAVA
public class Interface {
    String ONE = "1";
    String TWO = "2";
}
```

##### 嵌套接口：

```java
class Test {
    private interface B {
        void f();
    }

    class BImpl implements B {
        @Override
        public void f() {
            System.out.println(1);
        }
    }

    public B getB() {
        return new BImpl();
    }
}

public class Interface {
    public static void main(String[] args) {
        Test t = new Test();
        t.getB();
    }
}
```

我们看这种情况，提出一个问题，t.getB();的返回值我们要怎么接收呢？返回的是一个类中的私有的接口类型的对象。我们直接t.B肯定是不行的。这个时候，我们只能将返回值交给有权使用它的对象，所以我们要多定义一个属性或者方法

```java
class Test {
    private interface B {
        void f();
    }

    class BImpl implements B {
        @Override
        public void f() {
            System.out.println(1);
        }
    }

    public B bRef;

    public B getB() {
        return new BImpl();
    }
}

public class Interface {
    public static void main(String[] args) {
        Test t = new Test();
        t.bRef = t.getB();
    }
}
```