Spring学习笔记（一）

##### 什么是IOC，什么是DI？

以官网的说法，IOC也被称为DI。

##### Spring的自动装配

1. no
2. byName
3. byType
4. constructor

##### @Autowire 跟 @Resource的区别

1. 一个先type再name，另外一个先name再type
2. 解析的类不同
3. 来源不同，一个Spring，一个java

##### Spring的注入方式有几种？

1. setter
2. constructor

##### 单例对象中，注入原型对象，原型对象失去意义

1. 采用方法注入
   - @LookUp注解
   - 通过applicationContext对象每次getBean

##### Spring中bean的生命回调







