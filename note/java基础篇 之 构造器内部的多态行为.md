java基础篇 之 构造器内部的多态行为

​	我们来看下下面这段代码：

```java
public class Main {
    public static void main(String[] args) {
        new Son(5);
    }
}

class Person{
    void draw(){
        System.out.println("person draw");
    }
    Person (){
        System.out.println("before person draw");
        draw();
        System.out.println("after person draw");
    }
}

class Son extends Person{
    private int radius = 1;
    Son(int radius){
        this.radius = radius;
        System.out.println("son'radius="+radius);
    }
    @Override
    void draw(){
        System.out.println("son draw,radius="+radius);
    }
}

```

我们可以来猜下，运行结果是什么呢？

```
before person draw
son draw,radius=0
after person draw
son'radius=5
```

看到这个结果有迷惑的同学吗？为什么中间会输出`son draw,radius=0`呢？

下面我来解释下，首先，我们知道，在创建一个子类对象时，子类构造函数执行的时候，会默认调用父类的空参构造（如果有的话，如果没有空参构造，必须显示调用）。我们将其补全就成了这样

```java
    Son(int radius){
    	super();
        this.radius = radius;
        System.out.println("son'radius="+radius);
    }
```

我们又知道Person的构造函数执行的时候，调用了draw();方法，其实前面也省略了默认的this,我们也将其补全

```java
class Person{
    void draw(){
        System.out.println("person draw");
    }
    Person (){
        System.out.println("before person draw");
        this.draw();
        System.out.println("after person draw");
    }
}
```

看到这里不知道大家有没有明白一点？

在执行Son的构造方法时，会先调用Person类的构造函数，此时这个构造函数中，又调用了draw方法，并且是通过this调用的。这里this指向的是一个还未完全完成初始化的Son对象（因为构造Son的构造函数还未执行，此时还在执行父类的初始化），此时的radius被赋了一个默认初始值0。到这里大家应该明白了吧

