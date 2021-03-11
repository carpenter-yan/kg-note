##功能扩展

AbstractApplicationContext.prepareBeanFactory
```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    // 1.增加对SpEL语言的支持。
    /** SpEL是单独模块，只依赖于core模块，不依赖于其他模块，可以单独使用。
     * SpEL使用#{...}作为定界符，所有在大框号中的字符都将被认为是SpEL。*/
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 2.增加对属性编辑器的支持。
    /** 在bean的初始化后会调用ResourceEditorRegistrar的registerCustomEditors方法进行批量的通用属性编辑器注册。
     * 注册后，在属性填充的环节便可以直接让Spring使用这些编辑器进行属性的解析了。*/
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 3.增加对一些内置类，比如EnvironmentAware、MessageSourceAware的信息注入。
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 4.设直了依赖功能可忽略的接口。
    /** 当Spring将ApplicationContextAwareProcessor注册后，那么在invokeAwarelnterfaces方法中间接调用的Aware类已经不是普通的bean了，
    如ResourceLoaderAware、ApplicationEventPublisherAware等，那么当然需要在Spring做bean的依赖注入的时候忽略它们。
    而ignoreDependencyInterface的作用正是在此。*/
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 5.注册一些固定依赖的属性。
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    // 6.增加AspectJ的支持。
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    // 7.将相关环境变量及属性注册以单例模式注册。
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

###注册属性编辑器(2)
Spring中用于封装bean的是BeanWrapper类型，而它又间接继承了PropertyEditorRegistry类型，
也就是我们之前反复看到的方法参数PropertyEditorRegistry registry，其实大部分情况下都是BeanWrapper，
对于BeanWrapper在Spring中的默认实现是BeanWrapperlmpl，而BeanWrapperlmpl除了实现BeanWrapper接口外
还继承了PropertyEditorRegistrySupport，在PropertyEditorRegistrySupport中有这样一个方法
PropertyEditorRegistrySupport.createDefaultEditors

###添加ApplicationContextAwareProcessor处理器
ApplicationContextAwareProcessor.postProcessBeforeInitialization
```
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    AccessControlContext acc = null;

    if (System.getSecurityManager() != null &&
            (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
                    bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
                    bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareInterfaces(bean);
                return null;
            }
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }

    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
```
postProcessBeforelnitialization方法中调用了invokeAwarelnterfaces。从invokeAwareinterfaces方法中，
我们或许已经或多或少了解了Spring的用意，实现这些Aware接口的bean在被初始化之后，可以取得一些对应的资源。

###设置忽略依赖




