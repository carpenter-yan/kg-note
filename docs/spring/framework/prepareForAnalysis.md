## 核心类
### 1.DefaultListableBeanFactory
- AliasRegistry：定义对alias 的简单增删改等操作。
- SimpleAliasRegistry：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现。
- SingletonBeanRegistry：定义对单例的注册及获取。
- DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现。
- FactoryBeanRegistrySupport ：在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能。
- BeanDefinitionRegistry：定义对BeanDefinition的各种增删改操作。
- BeanFactory：定义获取bean及bean的各种属性。
- HierarchicalBeanFactory：继承BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对parentFactory的支持。
- ConfigurableBeanFactory：提供配置Factory的各种方法。
- AutowireCapableBeanFactory ：提供创建bean、自动注入、初始化以及应用bean的后处理器。
- ListableBeanFactory：根据各种条件获取bean 的配置清单。
- ConfigurableListableBeanFactory: Beanfactory配置清单，指定忽略类型及接口等。
- AbstractBeanFactory：综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能。
- AbstractAutowireCapableBeanFactory：综合AbstractBeanFacto1y并对接口AutowireCapableBeanFactory进行实现。
- DefaultListableBeanFactory：综合上面所有功能，主要是对bean注册后的处理。

### 2.XmlBeanDefinitionReader
- BeanDefinitionReader：主要定义资源文件读取并转换为BeanDefinition的各个功能。
- EnvironmentCapable：定义获取Environment 方法。
- AbstractBeanDefinitionReader：对EnvironmentCapable、BeanDefinitionReader类定义的功能进行实现。
- BeanDefinitionDocumentReader：定义读取Docuemnt并注册BeanDefinition功能。

## 辅助类
### 1.ClassPathResource
在Java中，将不同来源的资源抽象成URL，通过注册不同的handler(URLStreamHandler)来处理不同来源的资源的读取逻辑，一般handler的类型使用不同前缀（协议，Protocol ）来识别，如"file:""http:""jar:"等，然而URL没有默认定义相对Classpath或ServletContext等资源的handler，虽然可以注册自己的URLStreamHandler 来解析特定的URL前缀（协议），比如"classpath:"，然而这需要了解URL的实现机制，而且URL也也没有提供基本的方法，如检查当前资源是否存在、检查当前资源是否可读等方法。因而Spring对其内部使用到的资源实现了自己的抽象结构： Resource接口封装底层资源。

## 使用样例
`BeanFactory bf= new XmlBeanFactory(new ClassPathResource("beanFactory.xml"))`

## 1.BeanFactory初始化
### 1.1 parentBeanFactory
[spring和springmvc父子容器概念介绍](https://www.cnblogs.com/grasp/p/11042580.html)  
### 1.2 ignoreDependencyInterface
```
public AbstractAutowireCapableBeanFactory() {
    super();
    ignoreDependencyInterface(BeanNameAware.class);
    ignoreDependencyInterface(BeanFactoryAware.class);
    ignoreDependencyInterface(BeanClassLoaderAware.class);
}
```
  这里有必要提及ignoreDependencylnterface方法。ignoreDependencyInterface的主要功能是忽略给定接口的向动装配功能，那么，这样做的目的是什么呢？会产生什么样的效果呢？  
  举例来说，当A中有属性B，那么当Spring在获取A的Bean的时候如果其属性B还没有初始化，那么Spring会自动初始化B，这也是Sprig提供的一个重要特性。但是，某些情况下，B不会被初始化，其中的一种情况就是B实现了BeanNameAware接口。Spring 中是这样介绍的：自动装配时忽略给定的依赖接口，典型应用是通过过其他方式解析Application上下文注册依赖，类似于BeanFactory 通过BeanFactoryAware进行注入或者ApplicationContext通过ApplicationContextAware 进行注入。

## 2.XmlBeanDefinitionReader加载Bean定义
### 2.1:XmlBeanDefinitionReader.loadBeanDefinitions
- 封装资源文件。当进入XmnlBeanDefinitionReader后首先对参数Resource使用EncodedResource类进行封装。
- 获取输入流。从Resource中获取对应的InputStrearn并构造lnputSource 。
- 通过构造的InputSource实例和Resource实例继续调用函数doLoadBeanDefinitions。
### 2.2:XmlBeanDefinitionReader.doLoadBeanDefinitions
```
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
            getValidationModeForResource(resource), isNamespaceAware());
}
```
1. 获取对XML 文件的验证模式。
    getValidationModeForResource->detectValidationMode->XmlValidationModeDetector.detectValidationMode->hasDoctype  
    通过判断xml配置文件是否包含"DOCTYPE"字符来判断验证模式为DTD还是XSD。
2. 加载XML文件，并得到对应的Document。DefaultDocumentLoader.loadDocument通过DOM模式解析XML文档。
```
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
        ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    if (logger.isDebugEnabled()) {
        logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
    }
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    return builder.parse(inputSource);
}
```
2.1 getEntityResolver()用法
何为EntityResolver？官网这样解释如果SAX应用程序需要实现自定义处理外部实体，则必须实现此接口并使用setEntityResolver方法向SAX驱动器注册一个实例。也就是说，对于解析一个XML,SAX首先读取该XML文档上的声明，根据声明去寻找相应的DTD定义，以便对文档进行一个验证。默认的寻找规则，即通过网络（实现上就是声明的DTD的URI地址）来下载相应的DTD声明，并进行认证。下载的过程是一个漫长的过程，而且当网络中断或不可用时，这里会报错，就是因为相应的DTD声明没有被找到的原因。EntityResolver的作用是项目本身就可以提供一个如何寻找DTD声明的方法，即由程序来实现寻找DTD声明的过程，比如我们将DTD文件放到项目中某处，在实现时直接将此文档读取并返回给SAX即可。这样就避免了通过网络来寻找相应的声明。

3. 根据返回的Document注册Bean信息。
```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
单一职责原则，从xmlReader委托给documentReader。其真正类型是DefaultBeanDefinitionDocumentReader。
documentReader.registerBeanDefinitions方法中获取了document的root节点并调用doRegisterBeanDefinitions继续完成注册。
之前一直是XML加载解析的准备阶段，doRegisterBeanDefinitions算是真正开始解析了。

###创建读取上下文
XmlBeanDefinitionReader.createReaderContext
```
public XmlReaderContext createReaderContext(Resource resource) {
    return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
            this.sourceExtractor, this, getNamespaceHandlerResolver());
}

public NamespaceHandlerResolver getNamespaceHandlerResolver() {
    if (this.namespaceHandlerResolver == null) {
        this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
    }
    return this.namespaceHandlerResolver;
}

protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
    return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
}
```