# XML中Bean的加载

这将作为Spring源码学习的第一期，因为Spring最为人所知的两个特性就是IOC和DI（`Inversion of Control/Dependency Injection`）。

虽然使用XML注入的方式几乎已经很少使用了，但这仍然是最初Bean加载的方式。

## Bean的加载流程--代码1

```java
public void testSimpleLoad(){
    BeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
    MyTestBean bean = (MyTestBean) bf.getBean("myTestBean");
}
```

上面是XML方式注入bean最核心的两行代码，那么我们依次跟踪源码观察调用行为。

![xmlBeanFactory](../static/spring/xmlBeanFactory.png)

首先是得到`BeanFactory`的过程，`new ClassPathResource("beanFactoryTest.xml")`显然创建了一个`Resource`接口的实现类。

`Resource`接口是Spring抽象对底层资源的访问方式（File, URL, ClassPath等），后面会进行总结。

其中的`reader`是`XmlBeanDefinitionReader`。

![loadBeanDefinition](../static/spring/loadBeanDefinition.png)

### XmlBeanDefinitionReader

在真正的`loadlBeanDefinitions`方法中，观察这样一个片段

![doLoadBeanDefinition](../static/spring/doLoadBeanDefinition.png)

通过`Resource`获取到`InputStream`在将其封装成`InputSource`，

> 注释对InputSource的描述是 A single input source for an XML entity.

![doLoadBeanDefinition2](../static/spring/doLoadBeanDefinition2.png)

而在`doLoadBeanDefinitions`方法中，通过`DocumentLoad`的实现类`DefaultDocumentLoader`将`InputSource`解析成`Document`，这里的`Document`可以理解成被处理成了可以在Java中进行操作的XML文件结构。

### DefaultBeanDefinitionDocumentReader

那么最后终于到达`BeanDefinitionDocumentReader`接口实现类`DefaultBeanDefinitionDocumentReader`中进行Bean的加载了

![dom-parse](../static/spring/dom-parse.png)

可以首先看到的是对dom树进行了检查，是否设置了环境参数。

随后真正将dom树传入并进行bean的解析。这个方法调用过程没有什么特别的，然后下面是对**节点**的不同处理

![element-parse](../static/spring/element-parse.png)

这里同样可以看到对dom树节点的循环解析。

而接下来的工作就进入到了`BeanDefinitionParserDelegate`中了

### BeanDefinitionParserDelegate

![bdholder-register](../static/spring/bdholder-register.png)

首先一个重要的对象就是`BeanDefinitionHolder`，看名字就可以知道是对`BeanDefinition`的对象维护。其数据结构为：

```java
private final BeanDefinition beanDefinition; //bean的定义信息
 
private final String beanName; //bean的名字

@Nullable
private final String[] aliases; //bean的别名
```

获取`bdHolder`的过程，可以理解成按填入的class信息生成`BeanDefinition`的过程，同时封装成一个`Holder`，而在其中得到`AbstractBeanDefinition`的过程中：

![create-beanDefinition](../static/spring/create-beanDefinition.png)

可以看到对`bd`做了其依赖对象，以及构造参数的解析（这里一并进行了<bean>下定义内容的解析），并返回了`BeanDefinition`。

而上述的`createBeanDefinition`内部，其实是对传入的`className`进行了解析，**对这个Bean所实例化的类进行了加载**。

![load-class](../static/spring/load-class.png)

上图框中的方法返回的是`Class<?>`对象，传入的是`className`和`classLoader`。

然后最后将`name`-`BeanDefinition`作为键值对保持到了一个`ConcurrentHashMap`中。至此一个bean的定义的读取就完成了。

### 部分总结

 ![XML的Bean加载流程](../static/spring/XML的Bean加载流程.png)

1. Spring封装了自己的Resource结构用以实现对资源的同一抽象
2. 一个XML文件将被依次读取成Resource --> Document --> Bean的解析
3. 而对XML文件将进行格式验证，并最终获得Document对象
4. bean的注册过程实际加载的是类对象，而并没有立即实例化。考虑懒加载，但这个类又必然实例化，所以先将类加载进来。

## Bean的加载流程--代码2

接下来对标签的解析进行更细致的分析

--To be continue