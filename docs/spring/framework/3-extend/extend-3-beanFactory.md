##加载BeanFactory

AbstractApplicationContext.obtainFreshBeanFactory
```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //1. 初始化BeanFactory，并进行XML文件读取，并将得到的BeanFacotry记录在当前实体的属性中
    refreshBeanFactory();
    //2. 返回当前实体的beanFactory属性
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

AbstractRefreshableApplicationContext.refreshBeanFactory
```
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        //1. 创建DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        //2. 指定序列化ID 
        beanFactory.setSerializationId(getId());
        //3. 定制BeanFactory
        customizeBeanFactory(beanFactory);
        //4. 加载BeanDefinition
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            //5. 使用全局变量记录BeanFactory类实例
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

AbstractXmlApplicationContext.loadBeanDefinitions
```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        // 解析Bean配置
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```