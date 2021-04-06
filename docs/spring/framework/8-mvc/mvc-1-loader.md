###1. JavaWeb-servlet
[JavaWeb——Servlet（全网最详细教程包括Servlet源码分析）](https://blog.csdn.net/qq_19782019/article/details/80292110)  

###ContextLoaderListener
对于SpringMVC功能实现的分析，我们首先从Web.xml开始，在web.xml文件中我们首先配置的就是ContextLoaderListener

当使用编程方式的时候我们可以直接将Spring配置信息作为参数传入Spring容器中，如
ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");

但是在Web下，我们需要更多的是与Web环境相互结合，通常的办法是将路径以context-param的方式注册
并使用ContextLoaderListener进行监昕读取。ContextLoaderListener的作用就是启动Web容器时，
自动装配ApplicationContext的配置信息。因为它实现了ServletContextListener这个接口，
在web.xml 配置这个监昕器，启动容器时，就会默认执行它实现的方法，使用ServletContextListener接口，
开发者能够在为客户端请求提供服务之前向ServletContext 中添加任意的对象。
这个对象在ServletContext启动的时候被初始化， 然后在ServletContext整个运行期间都是可见的。
每一个Web应用都有一个ServletContext与之相关联。ServletContext对象在应用启动时被创建，
在应用关闭的时候被销毁。ServletContext在全局范围内有效，类似于应用中的一个全局变量。
在ServletContextListener中的核心逻辑便是初始化WebApplicationContext实例并存放至ServletContext中。

ContextLoaderListener.java
```
public void contextInitialized(ServletContextEvent event) {
    initWebApplicationContext(event.getServletContext());
}
```

ContextLoader.java
```
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 1. WebApplicationContext存在性的验证。
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
                "Cannot initialize context because there is already a root application context present - " +
                "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }

    servletContext.log("Initializing Spring root WebApplicationContext");
    Log logger = LogFactory.getLog(ContextLoader.class);
    if (logger.isInfoEnabled()) {
        logger.info("Root WebApplicationContext: initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        // Store context in local instance variable, to guarantee that
        // it is available on ServletContext shutdown.
        // 2. 创建WebApplicationContext 实例
        if (this.context == null) {
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent ->
                    // determine parent for root web application context, if any.
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        // 3. 将实例记录在servletContext 中。
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        // 4. 映射当前的类加载器与创建的实例到全局变量currentContextPerThread中。
        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            currentContext = this.context;
        }
        else if (ccl != null) {
            currentContextPerThread.put(ccl, this.context);
        }

        if (logger.isInfoEnabled()) {
            long elapsedTime = System.currentTimeMillis() - startTime;
            logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
        }

        return this.context;
    }
    catch (RuntimeException | Error ex) {
        logger.error("Context initialization failed", ex);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
        throw ex;
    }
}


protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    Class<?> contextClass = determineContextClass(sc);
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}

protected Class<?> determineContextClass(ServletContext servletContext) {
    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
    if (contextClassName != null) {
        try {
            return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load custom context class [" + contextClassName + "]", ex);
        }
    }
    else {
        contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
        try {
            return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load default context class [" + contextClassName + "]", ex);
        }
    }
}

在ContextLoader类中有这样的静态代码块：
static {
    // Load default strategy implementations from properties file.
    // This is currently strictly internal and not meant to be customized
    // by application developers.
    try {
        ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
    }
}

ContextLoader.properties
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```
综合以上代码分析，在初始化的过程中，程序首先会读取ContextLoader类的同目录下的属性文件ContextLoader.properties，
并根据其中的配置提取将要实现WebApplicationContext接口的实现类，并根据这个实现类通过反射的方式进行实例的创建。