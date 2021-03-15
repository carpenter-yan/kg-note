##动态AOP自定义标签
之前讲过Spring中的自定义注解，如果声明了自定义的注解，那么就一定会在程序中的某个地方注册了对应的解析器。

AopNamespaceHandler.init
```
public void init() {
    this.registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
    this.registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
    this.registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
    this.registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
}
```

所有解析器，因为是对BeanDefinitionParser接口的统一实现，入口都是从parse 函数开始的
AspectJAutoProxyBeanDefinitionParser.parse
```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
    this.extendBeanDefinition(element, parserContext);
    return null;
}

//AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(ParserContext parserContext, Element sourceElement) {
    //1. 注册或者升级AnnotationAwareAspectJAutoProxyCreator
    BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext.getRegistry(), parserContext.extractSource(sourceElement));
    //2. 处理Proxy-target-class以及expose-proxy属性
    useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
    registerComponentIfNecessary(beanDefinition, parserContext);
}
```

1. 注册或者升级AnnotationAwareAspectJAutoProxyCreator
```
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    // 如果已经存在了自动代政创建器且存在的自动代理创建者且与现在的不一致，那么需要根据仇先级来判断到底需要使用谁
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
        BeanDefinition apcDefinition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                //改变bean最重要的就是改变bean所对应的className属性
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        //如果已经存在自动代理创建器并且与将要创建的一致，那么无须再次创建
        return null;
    } else {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
        beanDefinition.setSource(source);
        beanDefinition.getPropertyValues().add("order", -2147483648);
        beanDefinition.setRole(2);
        registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
        return beanDefinition;
    }
}
```

2. 处理Proxy-target-class以及expose-proxy属性**
```
private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, Element sourceElement) {
    if (sourceElement != null) {
        boolean proxyTargetClass = Boolean.valueOf(sourceElement.getAttribute("proxy-target-class"));
        if (proxyTargetClass) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }

        boolean exposeProxy = Boolean.valueOf(sourceElement.getAttribute("expose-proxy"));
        if (exposeProxy) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }
    }
}

public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
        BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
        definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
    }
}

public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")) {
        BeanDefinition definition = registry.getBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator");
        definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
    }
}
```


