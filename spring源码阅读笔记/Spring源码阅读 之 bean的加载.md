Spring源码阅读 之 配置的读取，解析及bean的注册

> 在上文中我们已经知道了Spring如何从我们给定的位置加载到配置文件，并将文件包装成一个Resource对象。这篇文章我们将要探讨的就是，如何从这个Resouce对象中加载到我们的容器？加载到容器后又是什么样子呢？
>
> 大家可以跟着我一步步来，一定要把Spring啃完，加油~

[TOC]

##### 前期准备：

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

##### 开始解析：

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
		// abstract属性为true时，默认容器不创建这个bean,一般与parent属性配合使用
        // 以提供通用模板
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}
		// 是否需要懒加载，若为单例默认不是，若为多例，默认是
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
		
        // 解析autowire属性
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));
		
        // 解析depends-on属性
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}
		
        // 解析autowire-candidate属性
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
		
        // 解析primary属性
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}
		
        // 解析init-method属性
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}
		
        // 解析destory-method属性
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}
		
        // 解析factory-method属性
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
        
        // 解析factory-bean属性
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```

**上面还有几个标签的属性没有介绍，下面我们一起过一遍**：

- autowire属性

| **模式**    | **说明**                                                     |
| ----------- | ------------------------------------------------------------ |
| no          | (默认)不采用autowire机制.。这种情况，当我们需要使用依赖注入，只能用<ref/>标签。 |
| byName      | 通过属性的名称自动装配（注入）。Spring会在容器中查找名称与bean属性名称一致的bean，并自动注入到bean属性中。当然bean的属性需要有setter方法。例如：bean A有个属性master，master的setter方法就是setMaster，A设置了autowire="byName"，那么Spring就会在容器中查找名为master的bean通过setMaster方法注入到A中。 |
| byType      | 通过类型自动装配（注入）。Spring会在容器中查找类（Class）与bean属性类一致的bean，并自动注入到bean属性中，如果容器中包含多个这个类型的bean，Spring将抛出异常。如果没有找到这个类型的bean，那么注入动作将不会执行。 |
| constructor | 类似于byType，但是是通过构造函数的参数类型来匹配。假设bean A有构造函数A(B b,   C c)，那么Spring会在容器中查找类型为B和C的bean通过构造函数A(B b, C c)注入到A中。与byType一样，如果存在多个bean类型为B或者C，则会抛出异常。但时与byType不同的是，如果在容器中找不到匹配的类的bean，将抛出异常，因为Spring无法调用构造函数实例化这个bean。 |
| default     | 采用父级标签（即beans的default-autowire属性）的配置。        |

- autowire-candidate

前面我们说到配置有autowire属性的bean，Spring在实例化这个bean的时候会在容器中查找匹配的bean对autowire bean进行属性注入，这些被查找的bean我们称为候选bean。作为候选bean，我凭什么就要被你用，老子不给你用。所以候选bean给自己增加了autowire-candidate="false"属性（默认是true），那么容器就不会把这个bean当做候选bean了，即这个bean不会被当做自动装配对象。同样，<beans/>标签可以定义default-autowire-candidate="false"属性让它包含的所有bean都不做为候选bean。我的地盘我做主。

- primary

primary这个翻译过来是 首要的，首选的意思。

primary的值有true和false两个可以选择。默认为false。

当一个bean的primary设置为true，然后容器中有多个与该bean相同类型的其他bean，

此时，当使用@Autowired想要注入一个这个类型的bean时，就不会因为容器中存在多个该类型的bean而出现异常。而是优先使用primary为true的bean。

不过，如果容器中不仅有多个该类型的bean，而且这些bean中有多个的primary的值设置为true，那么使用byType注入还是会出错。

- init-method

用于指定bean初始化时指定执行的方法

- destory-method

用于指定bean销毁时指定执行的方法

- factory-method

指定从工厂中获取bean的方法

- factory-bean

指定工厂

示例如下：

```xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="
http://www.springframework.org/schema/beans 
http://www.springframework.org/schema/beans/spring-beans.xsd">
<!-- 默认使用无参数的构造放创建对象 -->
<bean id="bean1" class="com.igeek.Bean1"/>
<!-- 通过类中的静态方法获取类的对象 -->
<!-- factory-method:返回对象的静态方法的名称,相当于Bean2.getInstance()-->
<bean id="bean2" factory-method="getInstance" class="com.igeek.Bean2"/>
 WW
<!-- 通过实例工厂创建Bean对象 -->
<!-- 配置实例工厂对象 -->
<bean id="bean3Factory" class="com.igeek.Bean3Factory"/>
<!-- 
	factory-bean:创建Bean的实例工厂对象
	factory-method:工厂对象中创建实例的方法
 -->
<bean id="bean3" factory-bean="bean3Factory" factory-method="getBean3"/>
```

**接下来我们看看元数据的解析**：

```java
	// 入参是当前标签对应的Element及对应的Beandefinition
	public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
		NodeList nl = ele.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
				Element metaElement = (Element) node;
				String key = metaElement.getAttribute(KEY_ATTRIBUTE);
				String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
				BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
				attribute.setSource(extractSource(metaElement));
                // 将解析的结果放入Beandefinition，底层通过一个LinkedHashMap存储
				attributeAccessor.addMetadataAttribute(attribute);
			}
		}
	}
```

首先，元数据是什么呢？我们看如下配置：

```xml
<bean id="bean" class="com.study.test.configuration.Bean">
    <meta key="test" value="my"/>
</bean>
```

这段代码不会体现在Bean的属性当中，而是一个额外的申明，当需要使用里面的信息时候可以通过Beandefinition的getAttribute(key)方法获取

**关于look-up属性的解析**：

首先我们要知道look-up属性是用来做什么的，看如下代码：

XML配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:tx="http://www.springframework.org/schema/cache"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
<bean id="cat" class="com.study.test.configuration.spring.entity.Cat"/>
<bean id="dog" class="com.study.test.configuration.spring.entity.Dog"/>
<bean id="test" class="com.study.test.configuration.spring.entity.Test">
    <lookup-method name="getBean" bean="cat"/>
</bean>
</beans>
```

实体类：

```java
public interface Animal {
    void eat();
}

public class Cat implements Animal {
    @Override
    public void eat(){
        System.out.println("猫吃鱼");
    }
}

public class Dog implements Animal {
    @Override
    public void eat(){
        System.out.println("狗吃肉");
    }
}

public abstract class Test {

    public void eat() {
        getBean().eat();
    }
    abstract Animal getBean();
}

public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
    Test bean = context.getBean(Test.class);
    bean.eat();
}
```

运行结果：

```
猫吃鱼
```

如果我们将`<lookup-method name="getBean" bean="cat"/>`改为`<lookup-method name="getBean" bean="dog"/>`

运行结果：

```
狗吃肉
```

通过以上示例，我们可以总结look-up属性的作用了：

> 我们可以将一个方法申明为返回某种类型的bean,但实际要返回的bean是我们进行配置的

就像上面的实例中，我们将getBean()申明为返回了一个annimal对象，但是实际返回的对象是我们在xml中配置的

现在我们再来看看解析的代码：

```java
public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
            Element ele = (Element) node;
            String methodName = ele.getAttribute(NAME_ATTRIBUTE);
            String beanRef = ele.getAttribute(BEAN_ELEMENT);
            LookupOverride override = new LookupOverride(methodName, beanRef);
            override.setSource(extractSource(ele));
            overrides.addOverride(override);
        }
    }
}
```

可以看到将解析的定义封装成了一个LookupOverride，并添加进了BeanDefinition的MethodOverrides属性中

**关于replaced-method的解析：**

首先，我们还是要现在知道这个属性是干什么的，先看如下示例：

修改XML文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--<bean id="cat" class="com.study.test.configuration.spring.entity.Cat"/>-->
    <!--<bean id="dog" class="com.study.test.configuration.spring.entity.Dog"/>-->
    <bean id="test" class="com.study.test.configuration.spring.entity.Cat">
        <!--<lookup-method name="getBean" bean="cat"/>-->
        <replaced-method name="eat" replacer="myReplacer"/>
    </bean>
    <bean id="myReplacer" class="com.study.test.configuration.spring.entity.MyReplacer"/>
</beans>
```

新增类：

```java
public class MyReplacer implements MethodReplacer {
    @Override
    public Object reimplement(Object obj, Method method, Object[] args) throws Throwable {
        System.out.println("替代方法执行了");
        return null;
    }
}
```

```java
public class Spring {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");

//        ApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        Cat bean = context.getBean(Cat.class);
        bean.eat();
    }
}
```

执行结果如下：

```
替代方法执行了
```

可以看到，使用这个标签结合自定义`org.springframework.beans.factory.support.MethodReplacer`的实现，我们可以动态的替换运行的方法

接下来看看解析的过程：

```java
	public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
				Element replacedMethodEle = (Element) node;
                // 要被替换的方法
				String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
                // 替换器
				String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
                // 获取的数据封装到ReplaceOverride
				ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
                // 获取子标签arg-type对应的Element元素
				List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
				for (Element argTypeEle : argTypeEles) {
					String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
				// 如果子标签中的match属性没有值，就取标签中的文本
					match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
					if (StringUtils.hasText(match)) {
                        // 记录参数类型
						replaceOverride.addTypeIdentifier(match);
					}
				}
				replaceOverride.setSource(extractSource(replacedMethodEle));
				overrides.addOverride(replaceOverride);
			}
		}
	}
```

**构造函数的解析：**

核心方法为`org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseConstructorArgElement`

代码如下：

```java
	public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		if (StringUtils.hasLength(indexAttr)) {
			try {
				int index = Integer.parseInt(indexAttr);
				if (index < 0) {
					error("'index' cannot be lower than 0", ele);
				}
				else {
					try {
						this.parseState.push(new ConstructorArgumentEntry(index));
						Object value = parsePropertyValue(ele, bd, null);
						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
						if (StringUtils.hasLength(typeAttr)) {
							valueHolder.setType(typeAttr);
						}
						if (StringUtils.hasLength(nameAttr)) {
							valueHolder.setName(nameAttr);
						}
						valueHolder.setSource(extractSource(ele));
						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
							error("Ambiguous constructor-arg entries for index " + index, ele);
						}
						else {
							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
						}
					}
					finally {
						this.parseState.pop();
					}
				}
			}
			catch (NumberFormatException ex) {
				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
			}
		}
		else {
			try {
				this.parseState.push(new ConstructorArgumentEntry());
				Object value = parsePropertyValue(ele, bd, null);
				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
				if (StringUtils.hasLength(typeAttr)) {
					valueHolder.setType(typeAttr);
				}
				if (StringUtils.hasLength(nameAttr)) {
					valueHolder.setName(nameAttr);
				}
				valueHolder.setSource(extractSource(ele));
				bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
			}
			finally {
				this.parseState.pop();
			}
		}
	}
```

我们结合标签来解释上面这段代码：

```xml
<bean id="myReplacer" class="com.study.test.configuration.spring.entity.MyReplacer">
    <constructor-arg value="zhangSan" name="name" type="java.lang.String"/>
</bean>
```

上面代码主要根据`constructor-arg`标签中是否有index属性而分成了两个分支：

1. 如果有index属性，那么将解析的index,type,name封装称为一个`ConstructorArgumentValues.ValueHolder`,然后添加进`indexedArgumentValues`，这实际上是一个Map集合，key是index，value是封装好的对象
2. 如果没有index数据，同样也封装成一个`ConstructorArgumentValues.ValueHolder`，但是这种情况下，会将其添加到`genericArgumentValues`,这是一个ArrayList

上面这种设计也充分利用了Map跟ArrayList的特性

到这里位置，标签的分析我们就完成了最核心的部分了，现在回过头，我们再去看代码：

进入`org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader`的`parseBeanDefinitions`方法：

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {			// 第一句我们已经分析完了
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册bean
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

现在我们的任务就落在了`bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder)`这句代码上，从名字上来说，我们可以知道，它大概的目的就是在需要的情况下装饰BeanDefinition，那么什么情况会需要呢？我们看下其实现：

```java
	public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = definitionHolder;

		// 需要的话，先装饰自定义的属性
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// 需要的话，再装饰嵌套的节点
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
```

当Spring中的bean使用的是默认的标签配置，但是其中的子元素却使用了自定义的配置时，这个时候就需要装饰了，举例如下：

```xml
<bean id="myReplacer" class="com.study.test.configuration.spring.entity.MyReplacer">
    <mybean:user username="aa"/>
</bean>
```

具体逻辑不再赘述。

##### 总结：

经过上篇文章[Spring源码阅读 之 配置](https://blog.csdn.net/qq_41907991/article/details/96912201)的加载跟这篇文章，我们已经学习完了配置的加载，读取，解析。这篇文章中，并没有将所有标签的解析都说完，还有import,alias自定义标签等的解析没有进行说明，留给读者自行学习，实际上都大同小异

有正在学习Spring源码的朋友的话，可以留言一起交流哈~

下篇文章我们将着重分析，bean的注册，敬请期待哦~

喜欢的话，点个赞，加个关注吧，万分感谢！