##注册InfrastructureAdvisorAutoProxyCreator
我们以默认配置为例子进行分析，进入AopAutoProxyConfigurer类的configureAutoProxyCreator
```
public static void configureAutoProxyCreator(Element element, ParserContext parserContext) {
    AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);
    String txAdvisorBeanName = "org.springframework.transaction.config.internalTransactionAdvisor";
    if (!parserContext.getRegistry().containsBeanDefinition(txAdvisorBeanName)) {
        Object eleSource = parserContext.extractSource(element);
        //1. 创建TransactionAttributeSource的bean
        RootBeanDefinition sourceDef = new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource");
        sourceDef.setSource(eleSource);
        sourceDef.setRole(2);
        String sourceName = parserContext.getReaderContext().registerWithGeneratedName(sourceDef);
        //2. 创建TransactionInterceptor的bean
        RootBeanDefinition interceptorDef = new RootBeanDefinition(TransactionInterceptor.class);
        interceptorDef.setSource(eleSource);
        interceptorDef.setRole(2);
        AnnotationDrivenBeanDefinitionParser.registerTransactionManager(element, interceptorDef);
        interceptorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
        String interceptorName = parserContext.getReaderContext().registerWithGeneratedName(interceptorDef);
        //3. 创建TransactionAttributeSourceAdvisor的bean
        RootBeanDefinition advisorDef = new RootBeanDefinition(BeanFactoryTransactionAttributeSourceAdvisor.class);
        advisorDef.setSource(eleSource);
        advisorDef.setRole(2);
        advisorDef.getPropertyValues().add("transactionAttributeSource", new RuntimeBeanReference(sourceName));
        advisorDef.getPropertyValues().add("adviceBeanName", interceptorName);
        if (element.hasAttribute("order")) {
            advisorDef.getPropertyValues().add("order", element.getAttribute("order"));
        }

        parserContext.getRegistry().registerBeanDefinition(txAdvisorBeanName, advisorDef);
        //4. 创建CompositeComponentDefinition的bean
        CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), eleSource);
        compositeDef.addNestedComponent(new BeanComponentDefinition(sourceDef, sourceName));
        compositeDef.addNestedComponent(new BeanComponentDefinition(interceptorDef, interceptorName));
        compositeDef.addNestedComponent(new BeanComponentDefinition(advisorDef, txAdvisorBeanName));
        parserContext.registerComponent(compositeDef);
    }
}
```


```
// AopNamespaceUtils.registerAutoProxyCreatorIfNecessary
public static void registerAutoProxyCreatorIfNecessary(
        ParserContext parserContext, Element sourceElement) {

    BeanDefinition beanDefinition = AopConfigUtils.registerAutoProxyCreatorIfNecessary(
            parserContext.getRegistry(), parserContext.extractSource(sourceElement));
    useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
    registerComponentIfNecessary(beanDefinition, parserContext);
}

// AopConfigUtils.registerAutoProxyCreatorIfNecessary
public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}
```
对于解析来的代码流程AOP中已经有所分析，上面的两个函数主要目的是注册了InfrastructureAdvisorAutoProxyCreator类型的bean，
那么注册这个类的目的是什么呢?
从类的层次结构中可以看到，InfrastructureAdvisorAutoProxyCreator间接实现了SmartInstantiationAwareBeanPostProcessor，
而SmartInstantiationAwareBeanPostProcessor又继承自InstantiationAwareBeanPostProcessor，也就是说在Spring中，
所有bean实例化时Spring都会保证调用其postProcessAfterInitialization方法，其实现是在父类AbstractAutoProxyCreator类中实现。
当实例化bean时便会调用此方法，方法如下：

```
AbstractAutoProxyCreator.java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
这里实现的主要目的是对指定bean 进行封装，当然首先要确定是否需要封装，检测及封装的工作都委托给了wrapIfNecessary函数进行。

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    // 给定的bean类是否代表一个基础设施类，不应代理，或者配置了指定bean不需要自动代理
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // Create proxy if we have advice.
    // 1. 找出指定bean 对应的增强器。
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        //2. 根据找出的增强器创建代理。
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}


```

