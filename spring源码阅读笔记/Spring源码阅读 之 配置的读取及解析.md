Spring源码阅读 之 配置的读取及解析

> 在上文中我们已经知道了Spring如何从我们给定的位置加载到配置文件，并将文件包装成一个Resource对象。这篇文章我们将要探讨的就是，如何从这个Resouce对象中加载我们的bean呢？
>
> 大家可以跟着我一步步来，一定要把Spring啃完，加油~

上篇文章中，我们已经跟踪到了`org.springframework.beans.factory.support.AbstractBeanDefinitionReader`的`loadBeanDefinitions`方法，继续追踪代码，进入`org.springframework.beans.factory.xml.XmlBeanDefinitionReader`这个类中，追踪到如下代码：

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}
```

我们可以看到，到这里将`Resource对`象又进行了一次封装，封装成一个`EncodedResource`对象，那么`EncodedResource`又是做什么的？通过名字，我们大致可以推测这个类主要是用于对资源文件的编码进行处理的。我们可以发现它有如下方法：

```java
public Reader getReader() throws IOException {
    if (this.charset != null) {
        return new InputStreamReader(this.resource.getInputStream(), this.charset);
    } else {
        // 当我们设置了对应的编码属性时，Spring会使用相应的编码作为输入流的编码
        return this.encoding != null ? new 
            InputStreamReader(this.resource.getInputStream(), this.encoding) : new InputStreamReader(this.resource.getInputStream());
    }
}
```

OK，回到我们的主线，最终进入`XmlBeanDefinitionReader`的`doLoadBeanDefinitions`方法

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
            // 最终在这一步通过解析XML文件获取到了一个dom对象
            // 具体的解析逻辑就不多说了，解析XML都大同小异，有兴趣的读者可以自行百度
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			// 省略部分代码
            ....... 

```

```java
	// 注册bean定义
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        // 获取一个BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        // 获取注册前BeanDefinition的数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

继续跟踪代码进入：`org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReade`的`doRegisterBeanDefinitions`方法中

```java
protected void doRegisterBeanDefinitions(Element root) {
    	// BeanDefinitionParserDelegate专门负责对标签的解析
		// 在遇到嵌套标签时，该方法要被递归调用，为了delegate不被污染，先将其用parent保存
    	// 在方法执行完成后再this.delegate = parent，进行引用赋值
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
            // 获取标签中的profile属性
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			// 判断标签中定义的profile属性的值是否在环境变量中定义过
				if                 		   (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		// 用于子类复写，本身是空实现
		preProcessXml(root);
    	// 解析标签
		parseBeanDefinitions(root, this.delegate);
   		// 用于子类复写，本身是空实现
		postProcessXml(root);

		this.delegate = parent;
	}
```

继续跟踪代码，进入`org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader`的`parseBeanDefinitions`方法

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        // 解析默认的标签
						parseDefaultElement(ele, delegate);
					}
					else {
                        // 解析自定义的标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

上面提到了两个概念，一个是默认标签，一个是自定义标签，那么什么是默认标签，什么是自定义标签呢？

```xml
// 默认标签
<bean id="bean"></bean>
// 自定义标签
<tx:annotation-driven/>
```

我们先来看默认标签的解析：

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
   // 对import标签进行处理
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
   // 对alias标签进行处理
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    // 对bean标签进行处理
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    // 对beans标签进行处理
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // 递归
        doRegisterBeanDefinitions(ele);
    }
}
```

我们从最关键的`processBeanDefinition(ele, delegate)`方法开始分析：

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {		
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

以上代码主要做了这么几件事：

1. 提取元素中的id和name属性
2. 进一步解析其他所有属性并统一封装至GenericBeanDefinition类型的实例中
3. 如果监测到bean没有指定beanName，那么使用默认规则为此bean生成beanName
4. 将获取到的信息封装到BeanDefinitionHolder中

为了更好的理解上面的过程，我们需要弄明白上面涉及到的几个类都是做什么的

1. `BeanDefinitionParserDelegate`

这个类，我在上面已经说过了，主要负责解析我们的XML文件，方法大纲如下：

![spring02](H:\markdown\images\spring02.png)

2. `BeanDefinitionHolder`

```java
public class BeanDefinitionHolder implements BeanMetadataElement {

	private final BeanDefinition beanDefinition;

	private final String beanName;

	@Nullable
	private final String[] aliases;
```

因为这个类的方法还是非常简单的，相信大家自己能看明白，它的几个成员变量如上所示，这个类的主要作用是作为一个包含了别名及bean名称的beanDefinition的句柄

3. `BeanDefinition`

`BeanDefinition`是一个接口，在Spring中常见的实现有三种：

- org.springframework.beans.factory.support.RootBeanDefinition
- org.springframework.beans.factory.support.ChildBeanDefinition
- org.springframework.beans.factory.support.GenericBeanDefinition

整理其类图如下：![spring03](H:\markdown\images\spring03.png)

​		`BeanDefinition`是配置文件`<bean>`元素标签在容器的内部表示形式。`<bean>`元素标签拥有class,scope,lazy-init等配置属性，`BeanDefinition`则体统了相应的beanClass,scope,lazyInit属性，`BeanDefinition`和`<bean>`中的属性是一一对应的。其中`RootBeanDefinition`是最常用的实现类，它对应一般性的`<bean>`元素标签，在2.5以后，Spring中新增了`GenericBeanDefinition`，这个类的优势在于，能够动态的指定父依赖，而不是将这个类硬编码成为一个`RootBeanDefinition`

​		在配置文件中可以定义父`<bean>`和子`<bean>`，父`<bean>`用`RootBeanDefinition`表示，而子`<bean>`用`ChildBeanDefinition`表示，而没有父`<bean>`的`<bean>`就使用`RootBeanDefinition`表示。`AbstractBeanDefinition`对两者共同的类信息进行抽象

​		Spring通过`BeanDefinition`将配置文件中的`<bean>`配置信息转换为容器的内部表示，并将这些`BeanDefinition`注册到`BeanDefinitionRegistry`中。Spring容器的`BeanDefinitionRegistry`就像是Spring配置信息的内存数据库，主要是以map的形式保存，后续操作直接从BeanDefinitionRegistry`BeanDefinitionRegistry`中读取配置信息。

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {
       //parserState是一个栈，里面只能存放ParseState.Entry类型的元素，所以，要想将String类型的		   //beanName存入栈中，必须将beanName封装成Entry元素,这个类主要用于打印错误信息
		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
            // 前面已经说了BeanDefinition是配置文件在容器中的体现，所以在解析配置文件时必定要创建一个
            // BeanDefinition来承载对应的数据，就是在这个地方创建的
            // 用给定的className跟parent创建一个GenericBeanDefinition
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
		 
          	// 对"<id="" name="" scope="" ..>"配置形式进行解析，解析出该Bean的一些生命周期、 
          	// 是否延迟加载、自动装配方式等属性值，并且赋值给上面生成的BeanDefinition实例 
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
             //解析Bean的"<description ../>"属性  
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
			// 解析元数据
			parseMetaElements(ele, bd);
            // 解析lookup-method属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            // 解析replace-method属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
			// 解析构造函数参数
			parseConstructorArgElements(ele, bd);
            // 解析property子元素
			parsePropertyElements(ele, bd);
            // 解析qualifier子元素
			parseQualifierElements(ele, bd);
		    // ....省略部分代码.....
	}
```

接下来我们逐步分析每一个方法，并了解一下标签中每个属性的作用：

```java
	public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
		// singleton这个属性一个被scope替代
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
        // 指定了这个bean是单例还是多例的，单例singleton/多例prototype
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// 在嵌套bean标签的情况下，如果没指定scope的属性则使用父标签的scope属性值
			bd.setScope(containingBean.getScope());
		}
		// 
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));

		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if (isDefaultValue(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}

		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}

		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}

		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```

