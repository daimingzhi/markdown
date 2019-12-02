动态代理学习（一）自己动手模拟JDK动态代理

> 最近一直在学习Spring的源码，Spring底层大量使用了动态代理。所以花一些时间对动态代理的知识做一下总结。
>
> 1. 我们自己动手模拟一个动态代理
>
> 2. 对JDK动态代理的源码进行分析

[TOC]

##### 场景：

```java
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

public class Main {
	public static void main(String[] args) {
		MyServiceImpl target = new MyServiceImpl();
    }
}
```

我们现在要对`target`对象进行代理。大家可以想想，**我们如何去生成这个代理对象呢？**

##### 思路：

###### 分析：

- 我们先不考虑需要针对target生成一个代理对象，就单纯的生成一个对象来说，我们该怎么办呢？肯定是不能new的，因为我们根本没这个类。

- 所以为了动态的生成这个对象，我们需要动态的生成一个类，也就是说动态的加载一个类到jvm中

所以我们可以这么做：

1. 根据我们需要生成一个.java文件
2. 动态编译成一个.class文件
3. 拿到这个class文件后，我们通过反射获取一个对象

![动态代理1](H:\markdown\鲁班学院学习大纲及笔记\动态代理1.png)

现在问题来了，我们需要生成的java文件该是什么样子呢？我们可以思考，如果要对这个类做静态代理我们需要怎么做？

```java
package com.dmz.proxy;

import com.dmz.proxy.target.MyService;

/**
 * 静态代理
 */
public class StaticProxy implements MyService {

	private MyService target;

	public StaticProxy(MyService target) {
		this.target = target;
	}

	@Override
	public void test01() {
		System.out.println("proxy print log for test01");
		target.test01();
	}

	@Override
	public void test02(String s) {
		System.out.println("proxy print log for test02");
		target.test02(s);
	}
}

```

上面就是静态代理的代码，如果我们可以动态的生成这样的一个.java文件，然后调用jdk的方法进行编译，是不是就解决问题了呢？

##### 实践：

所以我们现在需要

1. 拼接字符串，将上面的代码以字符串的形式拼接出来并写入到磁盘文件上，并命名为xxxx.java文件
2. 编译.java文件，生成.class文件
3. 加载这个class文件到JVM内存中（实际上就是方法区中），得到一个class对象

4. 调用反射方法，class.newInstanc(.....)

代码如下：

```java
public class ProxyUtil {
	/**
	 * @param target 目标对象
	 * @return 代理对象
	 */
	public static Object newInstance(Object target) {
		Object proxy = null;
        // 开始拼接字符串
		Class targetInf = target.getClass().getInterfaces()[0];
		Method[] methods = targetInf.getDeclaredMethods();
		String line = System.lineSeparator();
		String tab = "\t";
		String infName = targetInf.getSimpleName();
		String content = "";
		String packageContent = "package com.dmz.proxy;" + line;
		String importContent = "import " + targetInf.getName() + ";" + line;
		String clazzFirstLineContent = "public class $Proxy implements " + infName + "{" + line;
		String filedContent = tab + "private " + infName + " target;" + line;
		String constructorContent = tab + "public $Proxy (" + infName + " target){" + line
				+ tab + tab + "this.target =target;"
				+ line + tab + "}" + line;
		String methodContent = "";
		for (Method method : methods) {
			String returnTypeName = method.getReturnType().getSimpleName();
			String methodName = method.getName();
			// Sting.class String.class
			Class args[] = method.getParameterTypes();
			String argsContent = "";
			String paramsContent = "";
			int flag = 0;
			for (Class arg : args) {
				String temp = arg.getSimpleName();
				//String
				//String p0,Sting p1,
				argsContent += temp + " p" + flag + ",";
				paramsContent += "p" + flag + ",";
				flag++;
			}
			if (argsContent.length() > 0) {
				argsContent = argsContent.substring(0, argsContent.lastIndexOf(",") - 1);
				paramsContent = paramsContent.substring(0, paramsContent.lastIndexOf(",") - 1);
			}

			methodContent += tab + "public " + returnTypeName + " " + methodName + "(" + argsContent + ") {" + line
					+ tab + tab + "System.out.println(\"proxy print log for " + methodName + "\");" + line
					+ tab + tab + "target." + methodName + "(" + paramsContent + ");" + line
					+ tab + "}" + line;

		}

		content = packageContent + importContent + clazzFirstLineContent + filedContent + constructorContent + methodContent + "}";
		//字符串拼接结束
        
        
        // 开始生成.java文件
		File file = new File("g:\\com\\dmz\\proxy\\$Proxy.java");
		try {
			if (!file.exists()) {
				file.createNewFile();
			}
			FileWriter fw = new FileWriter(file);
			fw.write(content);
			fw.flush();
			fw.close();
		// .java文件生成结束
                     
            
        // 开始进行编译   
			JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();

			StandardJavaFileManager fileMgr = compiler.getStandardFileManager(null, null, null);
			Iterable units = fileMgr.getJavaFileObjects(file);

			JavaCompiler.CompilationTask t = compiler.getTask(null, fileMgr, null, null, null, units);
			t.call();
			fileMgr.close();
		// 编译结束，生成.class文件
            
			// 从G盘中加载class文件
			URL[] urls = new URL[]{new URL("file:G:\\\\")};
			URLClassLoader urlClassLoader = new URLClassLoader(urls);
			// 加载
			Class clazz = urlClassLoader.loadClass("com.dmz.proxy.$Proxy");
            // 加载结束
            
            // 构造代理对象
			Constructor constructor = clazz.getConstructor(targetInf);
			proxy = constructor.newInstance(target);

		} catch (Exception e) {
			e.printStackTrace();
		}
		return proxy;
	}
}
```

我们调用这个方法：

```java
public class Main {
    public static void main(String[] args) {
        MyServiceImpl target = new MyServiceImpl();
        MyService o = ((MyService) ProxyUtil.newInstance(target));
        o.test01();
        o.test02("test02");
    }
}
```

会在我们的G盘中生成文件：

![1574343270860](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\1574343270860.png)

打开.java文件可以看到如下内容：

![1574343299677](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\1574343299677.png)

同时控制台会正常打印：

![1574343340717](C:\Users\daimzh\AppData\Roaming\Typora\typora-user-images\1574343340717.png)

这样，我们就完成了一个简单的代理。

