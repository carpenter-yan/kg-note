##初始化消息资源

AbstractApplicationContext.initMessageSource

JDK 的java.util 包中提供了几个支持本地化的格式化操作工具类NumberFormat、DateFormat、MessageFormat，
而在Spring 中的国际化资源操作也无非是对于这些类的封装操作。
Spring 定义了访问国际化信息的MessageSource接口，并提供了几个易用的实现类。
MessageSource分别被HierarchicalMessageSource和ApplicationContext接口扩展
HierarchicalMessageSource接口最重要的两个实现类是ResourceBundleMessageSource和ReloadableResourceBundleMessageSource。
它们基于Java 的ResourceBundle基础类实现，允许仅通过资源名加载国际化资源。ReloadableResourceBundleMessageSource 提供了定时刷新功能，
允许在不重启系统的情况下，更新资源的信息。StaticMessageSource 主要用于程序测试，它允许通过编程的方式提供国际化信息。
而DelegatingMessageSource 是为方便操作父MessageSource而提供的代理类。
```
protected void initMessageSource() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        //如果在配置中已经配置了messageSource，那么将messageSource提取并记录在this.messageSource中
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // Make MessageSource aware of parent MessageSource.
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // Only set parent context as parent MessageSource if no parent MessageSource
                // registered already.
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Using MessageSource [" + this.messageSource + "]");
        }
    }
    else {
        // Use empty MessageSource to be able to accept getMessage calls.
        // 如果用户并没有定义配置文件，那么使用临时的DelegatingMessageSource以便于作为调用getMessage方法的返回
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
                    "': using default [" + this.messageSource + "]");
        }
    }
}
```