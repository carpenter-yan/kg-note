## Tomcat启动
分析Tomcat嵌入原理首先要找到扩展入口，我们可以从启动信息开始TomcatEmbeddedServletContainer
->SpringApplication.run
    ->SpringApplication.createApplicationContext
    ->SpringApplication.refreshContext

SpringApplication.java
```
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
```
根据webApplicationType来判断创建哪种类型的Servlet。我们建立的是Web类型，所以实例化DEFAULT_SERVLET_WEB_CONTEXT_CLASS指定的类。
也就是AnnotationConfigServletWebServerApplicationContext类。这个类继承了ServletWebServerApplicationContext，
这才是真正的主角，而这个类最终继承了AbstractApplicationContext。

SpringApplication.java
```
private void refreshContext(ConfigurableApplicationContext context) {
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}
```
这里还是直接传递调用本类的refresh(context)方法，最后强转成父类AbstractApplicationContext，调用其refresh()方法

AbstractApplicationContext.java
```
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
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
我们看到onRefresh()方法是调用其子类的实现，根据我们上文的分析，我们这里的子类是ServletWebServerApplicationContext

ServletWebServerApplicationContext.java
```
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context",
                    ex);
        }
    }
    initPropertySources();
}
```
createWebServer()就是启动web服务，但是还没有真正启动Tomcat，是通过ServletWebServerFactory来获取的。
工厂类是一个接口，各个具体服务的实现是由各个子类来完成的，所以，就去看看TomcatServletWebServerFactory.getWebServer()的实现

TomcatServletWebServerFactory.java
```
public WebServer getWebServer(ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
            : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
}
```
主要做两件事，第一件事就是把Connnctor(我们称之为连接器)对象添加到Tomcat中；
第二件事就是configureEngine

Tomcat.java
```
public Engine getEngine() {
    Service service = getServer().findServices()[0];
    if (service.getContainer() != null) {
        return service.getContainer();
    }
    Engine engine = new StandardEngine();
    engine.setName( "Tomcat" );
    engine.setDefaultHost(hostname);
    engine.setRealm(createDefaultRealm());
    service.setContainer(engine);
    return engine;
}
```
Engine是容器继承了Container接口。改接口有4个子接口，分别是Engine、Host、Context、Wrapper。
Engine是最高级别的容器，其子容器是Host，Host的子容器是Context，Wrapper是Context的子容器，
所以这4个容器的关系就是父子关系。也就是：Engine > Host > Context > Wrapper
Tomcat.java
```
// 连接器(Connector)是设置在Service下的，而且是可以设置多个连接器(Connector)
public void setConnector(Connector connector) {
    Service service = getService();
    boolean found = false;
    for (Connector serviceConnector : service.findConnectors()) {
        if (connector == serviceConnector) {
            found = true;
        }
    }
    if (!found) {
        service.addConnector(connector);
    }
}

public Service getService() {
    return getServer().findServices()[0];
}

// Tomcat的最顶层就是Server，也就是Tomcat的实例，一个Tomcat一个Server
public Server getServer() {

    if (server != null) {
        return server;
    }

    System.setProperty("catalina.useNaming", "false");

    server = new StandardServer();

    initBaseDir();

    // Set configuration source
    ConfigFileLoader.setSource(new CatalinaBaseConfigurationSource(new File(basedir), null));

    server.setPort( -1 );

    Service service = new StandardService();
    service.setName("Tomcat");
    server.addService(service);
    return server;
}

// Server下面是Service,而且是多个，一个Service代表我们部署的一个应用，还可以知道，Engine容器，一个Service只有一个
public Engine getEngine() {
    Service service = getServer().findServices()[0];
    if (service.getContainer() != null) {
        return service.getContainer();
    }
    Engine engine = new StandardEngine();
    engine.setName( "Tomcat" );
    engine.setDefaultHost(hostname);
    engine.setRealm(createDefaultRealm());
    service.setContainer(engine);
    return engine;
}

// host容器有多个
public void setHost(Host host) {
    Engine engine = getEngine();
    boolean found = false;
    for (Container engineHost : engine.findChildren()) {
        if (engineHost == host) {
            found = true;
        }
    }
    if (!found) {
        engine.addChild(host);
    }
}

// Context也是多个
public Context addContext(Host host, String contextPath, String contextName,
        String dir) {
    silence(host, contextName);
    Context ctx = createContext(host, contextPath);
    ctx.setName(contextName);
    ctx.setPath(contextPath);
    ctx.setDocBase(dir);
    ctx.addLifecycleListener(new FixContextListener());

    if (host == null) {
        getHost().addChild(ctx);
    } else {
        host.addChild(ctx);
    }
    return ctx;
}

// Wrapper容器也是多个， 也暗示了wrapper和Servlet是一层意思
public static Wrapper addServlet(Context ctx,
                                  String servletName,
                                  Servlet servlet) {
    // will do class for name and set init params
    Wrapper sw = new ExistingStandardWrapper(servlet);
    sw.setName(servletName);
    ctx.addChild(sw);

    return sw;
}

```
一个Tomcat是一个Server，一个Server下有多个Service，也就是我们部署的多个应用。
一个应用下有多个连接器(Connector)和一个容器（Container）,而容器下又有多个子容器，
按照父子关系分别为：Engine、Host、Context、Wrapper，其中除了Engine外，其余的容器都是可以有多个。