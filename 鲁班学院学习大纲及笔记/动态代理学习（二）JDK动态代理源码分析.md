动态代理学习（二）JDK动态代理源码分析

> 上篇文章我们学习了如何自己实现一个动态代理，这篇文章我们从源码角度来分析下JDK的动态代理

先看一个Demo:

```java
public class MyInvocationHandler implements InvocationHandler {

	private MyService target;

	public MyInvocationHandler(MyService target) {
		this.target = target;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object invoke = method.invoke(target, args);
		System.out.println("proxy invoke");
		if (method.getReturnType().equals(Void.TYPE)) {
			return null;
		} else {
			System.out.println(invoke);
			return invoke+"proxy";
		}
	}
}

public interface MyService {
	void test01();
	void test02(String s);
}

public class MyServiceImpl implements MyService {

	@Override
	public void test01() {
		System.out.println("test01");
	}

	@Override
	public void test02(String s) {
		System.out.println(s);
	}
}
```

main方法：

```java
public class Main {
	public static void main(String[] args) {
		MyServiceImpl target = new MyServiceImpl();
		Class<? extends MyServiceImpl> clazz = target.getClass();
		MyService proxyInstance = (MyService) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces()
				, new MyInvocationHandler(target));
		proxyInstance.test01();
		proxyInstance.test02("test02");
	}
}
```

我们运行Debug观察下生成的`proxyInstance`对象：

![image-20191122152308722](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\image-20191122152308722.png)

可以得出以下几个结论：

1. 生成的代理类的类名是`$Proxy0`
2. 代理类持有我们的`MyInvocationHandler`对象

这里我们越过不重要的代码，直接端点到`java.lang.reflect.Proxy.ProxyClassFactory#apply`这个方法，

我们分段分析这个方法的代码（简单的代码我们就直接跳过了）：

```java
 Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
```

这段代码主要是为了确保类加载器对这个class文件解析后得到的是同一个对象。如果我们要确保两个对象相等的话，那么它们的类加载器必定是一样的。

```java
for (Class<?> intf : interfaces) {
    int flags = intf.getModifiers();
    if (!Modifier.isPublic(flags)) {
        accessFlags = Modifier.FINAL;
        String name = intf.getName();
        int n = name.lastIndexOf('.');
        String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
        if (proxyPkg == null) {
            proxyPkg = pkg;
        } else if (!pkg.equals(proxyPkg)) {
            throw new IllegalArgumentException(
                "non-public interfaces from different packages");
        }
    }
}

 if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }
```

这段代码主要是在判断接口是否是public的，如果不是public的那么需要将代理类生成在接口同名的包下。否则生成的代理类在`com.sun.proxy`包下。

这里我们可以做一个验证：

1. 我们测试接口如果不是public的，代理类会生成在接口的同一个包下，在这种情况下，我们可以在接口的同名包下新建一个类，类名为`$Proxy0`，如下：

```java
// 接口换为包访问权限
interface MyService {
	void test01();
	void test02(String s);
}
```

新建一个类，类名为`$Proxy0`

![](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\image-20191122160929009.png)

2. main函数进行测试

```java
	public static void main(String[] args) {
		MyServiceImpl target = new MyServiceImpl();
		Class<? extends MyServiceImpl> clazz = target.getClass();
        // 加载这个类到JVM中
		$Proxy0 proxy0 = new $Proxy0();
		MyService proxyInstance = (MyService) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces()
				, new MyInvocationHandler(target));
		proxyInstance.test01();
		proxyInstance.test02("test02");
		}
```

运行后发现报错：显示有重复的类定义

```java
Exception in thread "main" java.lang.LinkageError: loader (instance of  sun/misc/Launcher$AppClassLoader): attempted  duplicate class definition for name: "com/dmz/proxy/target/$Proxy0"
	at java.lang.reflect.Proxy.defineClass0(Native Method)
	at java.lang.reflect.Proxy.access$300(Proxy.java:228)
	at java.lang.reflect.Proxy$ProxyClassFactory.apply(Proxy.java:642)
	at java.lang.reflect.Proxy$ProxyClassFactory.apply(Proxy.java:557)
	at java.lang.reflect.WeakCache$Factory.get(WeakCache.java:230)
	at java.lang.reflect.WeakCache.get(WeakCache.java:127)
	at java.lang.reflect.Proxy.getProxyClass0(Proxy.java:419)
	at java.lang.reflect.Proxy.newProxyInstance(Proxy.java:719)
	at com.dmz.proxy.target.Main.main(Main.java:14)
```

从上面我们就验证了，**如果不是public的那么需要将代理类生成在接口同名的包下**

接下来我们验证，正常情况下，代理类会被生成在`com.sun.proxy`包下

1. 同理，我们可以创建一个类，全类名为`com.sun.proxy.$Proxy0`

![image-20191122162034580](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\image-20191122162034580.png)

2. 同时我们将接口改为public的，同样的我们会发现会报同一个错。

至此，证明完毕。我们继续看代码

```java
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
                // 省略部分代码......
```

我们可以看到，通过`ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags)`生成一个字节流后，直接调用了defineClass0(....)方法，而且我们跟踪这个方法可以发现，这是一个本地方法。并且它直接返回了一个Class对象。

```java
private static native Class<?> defineClass0(ClassLoader loader, String name,
                                            byte[] b, int off, int len);
```

回顾我们上一篇文章的实现思路：

![动态代理1](H:\markdown\鲁班学院学习大纲及笔记\动态代理1.png)

对比后我们可以发现，我们自己实现时，是通过生成java文件，然后进行编译生成class文件，再将其加载到JVM中得到class对象。而对于jdk动态代理，直接通过一个字节流调用本地方法后直接生成class对象。

我们再回过头去看下jdk是如何给我们生成这个字节流的，这里我们主要关注`sun.misc.ProxyGenerator#generateClassFile`这个方法，这里我就不贴代码了。因为也是一些字符串的拼接动作，然后写入到一个字节流中，我们关注下最后生成的这个字节流是什么样子的，我们将其写入到一个文件中：

```java
		MyServiceImpl target = new MyServiceImpl();
		Class<? extends MyServiceImpl> clazz = target.getClass();
		MyService proxyInstance = (MyService) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces()
				, new MyInvocationHandler(target));

		byte[] bytes = ProxyGenerator.generateProxyClass("proxy", clazz.getInterfaces());

		File file = new File("G:\\com\\dmz\\proxy\\proxy.class");
		FileOutputStream outputStream = new FileOutputStream(file);
		outputStream.write(bytes);
		proxyInstance.test01();
		proxyInstance.test02("test02");
```

我们将得到的这个class文件放入idea反编译：

```java
public final class proxy extends Proxy implements MyService {
    private static Method m1;
    private static Method m4;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public proxy(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void test01() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void test02(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("com.dmz.proxy.target.MyService").getMethod("test01");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.dmz.proxy.target.MyService").getMethod("test02", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```

观察上面代码，我们可以发现以下几点：

1. 代理类继承了`Proxy`这个类，正因为如此，所以jdk动态代理只能实现基于接口的代理，而不能实现对整个类进行代理，因为java是单继承的。那么为什么代理类一定要继承`Proxy`这个类呢？我们可以发现代理类并没有使用`Proxy`中的什么属性或者方法（虽然使用了InvocationHandler对象，但是也可以在生成class之初就将InvocationHandler放入到代理类中）。所以实际上不进行继承也是没有任何关系的。查了很多资料后发现，找到一个比较合理的解释如下：

   > JDK的动态代理只允许动态代理接口是设计使然，因为动态代理一个类存在一些问题。在代理模式中代理类只做一些额外的拦截处理，实际处理是转发到原始类做的。这里存在两个对象，代理对象跟原始对象。如果允许动态代理一个类，那么代理对象也会继承类的字段，而这些字段是实际上是没有使用的，对内存空间是一个浪费。因为代理对象只做转发处理，对象的字段存取都是在原始对象上处理。更为致命的是如果代理的类中有final的方法，动态生成的类是没法覆盖这个方法的，没法代理，而且存取的字段是代理对象上的字段，这显然不是我们希望的结果。spring aop框架就是这种模式。

   总结起来主要两点

   - 我们在进行代理时，实际的方法执行逻辑仍然是交给目标类处理，这个时候代理类持有目标类中的字段只不过是对内存空间的一种浪费，其余没有任何作用。

   - 即使我们能接受对内存空间的浪费，然而如果我们在代理对象中操作代理对象中的字段，目标对象的字段不受任何影响，这显然也是不合理的。

   - 如果是基于继承实现代理，那么有final的方法的情况下，无法完成对final方法的代理。

     

2. 代理类实现了我们目标对象实现的接口，所以说JDK动态代理是基于接口实现的。

3. 代理对象不仅仅是对接口中的方法进行了代理，还对hashCode,equals,toString三个方法进行了代理，这也是为了覆盖目标类中的所有方法

至此，我们就完成对JDK动态代理的学习！喜欢的同学点个赞吧~~~~