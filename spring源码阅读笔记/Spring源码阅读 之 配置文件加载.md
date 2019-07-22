Spring源码阅读 之 配置的加载（希望有喜欢源码的朋友一起交流）

> 想写Spring的源码方面的东西想了好久了，之前花了一段时间学习了SpringCloud，现在总算对SpringCloud有了一个大概的了解，从今天开始好好读一篇Spring的源码，结合书本跟网上的一些资料，希望能坚持下去，完成这一系列的文章，给自己加油！！！！

版本是基于5.1.X的，大家在阅读的时候请注意，不过因为是Spring的一些基础流程，估计版本间差异也不会太大

关于配置的加载主要可以分为两种：

1. 注解式配置的加载
2. XML配置加载

注解配置相对于XML更加简单，其过程主要在于注解的解析，而XML配置是用xml文件格式保存配置，其中就涉及到了文件的读取，读取后要进行标签的解析，稍显复杂。我们这篇文章主要就是来讲清楚Spring读取配置文件的过程，对于后面的XML标签的解析或者说是注解的解析其实都是差不多的，在后面的文章中，我会慢慢介绍

我们先看一段代码：

```java
/**
 * @author dmz
 * @date Create in 20:49 2019/7/20
 */
public class Spring {
    public static void main(String[] args) {
        
        // 构建一个基于XML的Spring应用的上下文
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("application.xml");
        
    }
}
```

很简单，目的就是跟踪一个基于XML的Spring应用是如何被加载的，我们一步步点进去看源码可以发现：

```java
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
		// 这个构造函数暂时不说
		super(parent);
        // 这个方法主要是为了设置这个应用需要使用的Spring的配置文件
        // 主要是对配置文件的路径进行一些处理
		setConfigLocations(configLocations);
		if (refresh) {
            // 这是Spring的核心方法
			refresh();
		}
	}
```

Spring的源码很大，我们需要一点点啃，这篇文章目的只是分析配置的读取，所以只分析跟配置相关的代码，在这里也提醒大家，阅读源码时不要被细枝末节牵绊的太深，抓住自己需要分析的主线，然后不断探究，能做到这样就是极好的！！！

我们接着说代码：

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 为容器的刷新做准备
			prepareRefresh();
			// 主要分析这个代码
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// 省略后面的代码
			.......
	}

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新BeanFactory
   refreshBeanFactory();
   return getBeanFactory();
}
    
protected final void refreshBeanFactory() throws BeansException {
    // 如果当前容器的BeanFactory已经经过初始化，进行清空
      if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
      }
     try {
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
         // 我们主要需要关注的方法
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
    }
  }
    catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
}
    
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // 我们可以看到，在这一步，创建了一个XmlBeanDefinitionReader，构造参数是一个DefaultListableBeanFactory
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
		// 对这个reader的成员变量进行赋值
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
		// 提供给子类使用，让子类能自定义BeanDefinitionReader
        initBeanDefinitionReader(beanDefinitionReader);
        // 最后调用loadBeanDefinitions
        loadBeanDefinitions(beanDefinitionReader);
    }
```

我们将注意力放到`loadBeanDefinitions`这个方法上，从方法名称上我们可以知道，这个方法一定读取了配置文件，跟踪其代码最后会到`org.springframework.context.support.AbstractXmlApplicationContext`的`loadBeanDefinitions`方法，我们看下这个方法的具体实现：

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    // 我们之前给定的是一个application.xml，所以会进这个方法
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```

继续跟踪，进入`org.springframework.beans.factory.support.AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法

```java
	public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
        // 获取一个资源加载器
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}
		// 这里获取到的资源加载器是一个org.springframework.context.support.ClassPathXmlApplicationContext，而所有的applicationContext都实现了ResourcePatternResolver接口
		if (resourceLoader instanceof ResourcePatternResolver) {
			try {
                // 呼~，终于到这个加载资源的方法了，这个方法会根据不同location，选择对应的resourceLoader加载配置文件
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int count = loadBeanDefinitions(resources);
		......省略部分代码......
```

在我们的示例中，最终会调用`org.springframework.core.io.DefaultResourceLoader`的`getResource`方法

```java
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");

    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }

    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // Try to parse the location as a URL...
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // No URL -> resolve as resource path.
            return getResourceByPath(location);
        }
    }
}
```

我们可以看到，这个方法会根据一些匹配规则，比如说localtion是否以“/”开头，是否是一个FileURL等等分别创建

ClassPathResource，FileUrlResource，UrlResource等，我们可以看一下它的类图：

![spring01](H:\markdown\images\spring01.png)

​	在java中，将不同来源的资源抽象成URL，通过注册不同的handler（URLStramHandler）来处理不同来源的资源的读取逻辑，一般hanlder使用不同前缀（协议，Protocol）来识别，如“file”,"http:"等，然而URL没有默认定义相对Classpath或者ServletContext等资源的hanlder,虽然可以注册自己的URLStream来解析特定的URL前缀，比如“classpath:”，然后这需要了解URL的实现机制，而且URL也没有提供基本的方法，如检查当前资源是否存在，检查当前资源是否可读等，因而Spring对其内部使用到的资源实现了自己的抽象结构：Resource接口封装底层资源

```java
public interface InputStreamSource {

    /**
     * 顶层接口，返回资源对应的输入流
  	 */
    InputStream getInputStream() throws IOException;

}
```



```java
public interface Resource extends InputStreamSource {

	/**
	 * 是否存在
	 */
	boolean exists();

	/**
	 * 是否可读
	 */
	default boolean isReadable() {
		return exists();
	}

	/**
	 * 是否打开
	 */
	default boolean isOpen() {
		return false;
	}

	/**
     * 是否是一个file,如果是的话，getFile()一定能正常返回
	 */
	default boolean isFile() {
		return false;
	}

	/**
     * 返回资源对应的URL
	 */
	URL getURL() throws IOException;

	/**
     * 返回资源对应的URI
	 */
	URI getURI() throws IOException;

	/**
     * 返回资源对应的文件
	 */
	File getFile() throws IOException;

	/**
     * 返回一个根据当前资源转变的可读的nio中的ReadableByteChannel
	 */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
     * 返回资源的字节长度
	 */
	long contentLength() throws IOException;

	/**
	 * 返回上一次修改的时间戳
	 */
	long lastModified() throws IOException;

	/**
	 * 基于当前资源，创建一个相对资源
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * 返回资源的文件名(不带路径)，没有文件名则返回null
	 */
	@Nullable
	String getFilename();

	/**
	 * 返回资源描述
	 */
	String getDescription();

}
```

绝大部分情况下，我们都是直接在resoures直接创建配置文件，所以大部分情况，我们使用的都是`ClassPathResource`,我们不妨看一下它对这些方法的实现，加深我们对这些接口的理解

```java
@Override
// 可以发现，就是通过类加载器获取对应资源，最终的实现还是会依赖于我们jdk中的URL类
// 最后通过URL返回一个流
public InputStream getInputStream() throws IOException {
    InputStream is;
    if (this.clazz != null) {
        is = this.clazz.getResourceAsStream(this.path);
    }
    else if (this.classLoader != null) {
        is = this.classLoader.getResourceAsStream(this.path);
    }
    else {
        is = ClassLoader.getSystemResourceAsStream(this.path);
    }
    if (is == null) {
        throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
    }
    return is;
}
// 判断这个ClassPathResource对应的路径能否被解析成一个java中的URL，如果能则存在
public boolean exists() {
    return (resolveURL() != null);
}
.......
```

经过上面的分析我们已经知道了Spring怎么加载一个配置文件，我们可以总结如下：

1. 创建一个`XmlBeanDefinitionReader`
2. 获取对应的`ResourceLoader`资源加载器
3. 根据location的不同匹配模式，采用不同的ResourceLoader进行资源的加载
4. Spring封装了java中的URL类，自定义了一套资源加载策略
5. 最后返回一个Spring中的Resource对象