##容器控制功能

###使用样例
AbstractRefreshableConfigApplicationContext.ClassPathXmlApplicationContext
`ApplicationContext bf= new ClassPathXmlApplicationContext("beanFactoryTest.xml")";`

```
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);
}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
        throws BeansException {

    super(parent);
    //设置配置文件路径
    setConfigLocations(configLocations);
    if (refresh) {
        //解析
        refresh();
    }
}

public void setConfigLocations(String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            //解析给定路径
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```

AbstractApplicationContext.refresh
```
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        /** 1. 初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。
         * 在某种情况下项目的使用需要读取某些系统变量，而这个变量的设置很可能会影响着系统的正确性，
         * 那么ClassPathXm!ApplicationContext为我们提供的这个准备函数就显得非常必要，
         * 它可以在Spring启动的时候提前对必需的变量进行存在性验证。*/
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 2.初始化BeanFactory并进行XML文件读取
        /** 2. 初始化BeanFactory，并进行XML文件读取。
         * 之前有提到ClassPathXmlApplicationContext包含着BeanFactory所提供的一切特征，那么
         * 在这一步骤中将会复用BeanFactory中的配置文件读取解析及其他功能，
         * 这一步之后，ClassPathXmlApplicationContext实际上就已经包含了BeanFactory所提供的功能，
         * 也就是可以进行bean的提取等基础操作了。*/
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 3.对BeanFacotry进行各种功能扩充
        /** 3. 对BeanFactory 进行各种功能填充。
         * @Qualifier与@Autowired应该是大家非常熟悉的注解，那么这两个注解正是在这一步骤中增加的支持。*/
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            /** 4. 子类覆盖方法做额外的处理。
             * Spring之所以强大，为世人所推崇，除了它功能上为大家提供了便例外，还有一方面是它的完美架构，
             * 开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能。这种开放式的设计在Spring 中随处可见，
             * 例如在本例中就提供了一个空的函数实现postProcessBeanFactory来方便程序员在业务上做进一步扩展。*/
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 5. 激活各种BeanFactory处理器。
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 6. 注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候。
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 7. 为上下文初始化Message源，即对不同语言的消息体进行国际化处理。
            initMessageSource();

            // Initialize event multicaster for this context.
            // 8. 初始化应用消息广播器，并放入"applicationEventMulticaster"bean 中。
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 9. 留给子类来初始化其他的bean。
            onRefresh();

            // Check for listener beans and register them.
            // 10. 在所有注册的bean中查找listenerbean，注册到消息广播器中。
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 11.初始化剩下的单实例（非惰性的）。
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 12. 完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人。
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```