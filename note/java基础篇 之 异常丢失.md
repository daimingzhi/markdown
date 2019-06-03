java基础篇 之 异常丢失

我们看如下代码：

```java
@Slf4j
public class Test {
    public static void main(String[] args) {
        try {
            try {
                test();
            } finally {
                test2();
            }
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }

    }

    public static void test() {
        System.out.println("test方法执行了");
        throw new RuntimeException("veryImportant exception");
    }

    public static void test2() {
        System.out.println("test2");
        throw new CustomeException("common exception");
    }
}
```

执行结果如下：

```

```

我们可以看到，在执行的适合，一个veryImportant exception丢失了，而抛出了一个common exception，这是相当严重的缺陷，因为异常可能会以一种比前面例子所示更微妙和难以察觉的方式完全丢失。相比之下，C++把“前一个异常还没处理就抛出下一个异常”的情形看成是糟糕的错误。

一种更加糟糕的编程手法如下所示：

```java
package com.study.spring.transaction.controller;

import com.study.spring.transaction.CustomeException;
import lombok.extern.slf4j.Slf4j;

/**
 * @Author: dmz
 * @Description:
 * @Date: Create in 2:40 2019/4/21
 */
@Slf4j
public class Test {
    public static void main(String[] args) {
        try {
            try {
                test();
            } finally {
                return;
            }
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }

    }

    public static void test() {
        System.out.println("test方法执行了");
        throw new RuntimeException("veryImportant exception");
    }

    public static void test2() {
        System.out.println("test2");
        throw new CustomeException("common exception");
    }
}
```

可以看到，在finally快中，我们直接将函数返回了，这个适合执行代码，会发现，即使抛出了异常，也不会有任何输出。

这种情况下我们怎么办呢？我觉得主要就是多加日志，虽然程序不会主动输出什么，但是我们可以在出现错误的地方自己打印日志

```java
@Slf4j
public class Test {
    public static void main(String[] args) {
        try {
            try {
                test2();
            } finally {
                return;
            }
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }

    }

    public static void test() {
        System.out.println("test方法执行了");
        throw new RuntimeException("veryImportant exception");
    }

    public static void test2() {
        System.out.println("test2");
        CustomeException common_exception = new CustomeException("common exception");
        log.error("test2调用失败", common_exception);
        throw common_exception;
    }
```

调用结果：

```
test2
00:46:09.508 [main] ERROR com.study.spring.transaction.controller.Test - test2调用失败
com.study.spring.transaction.CustomeException: common exception
	at com.study.spring.transaction.controller.Test.test2(Test.java:33)
	at com.study.spring.transaction.controller.Test.main(Test.java:16)

Process finished with exit code 0

```

