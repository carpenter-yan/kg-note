##WebApplicationContext初始化

FrameworkServlet.java
```
protected WebApplicationContext initWebApplicationContext() {
    // 1 .寻找或创建对应的WebApplicationContext实例
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    // 1.1 通过构造函数的注入进行初始化
    if (this.webApplicationContext != null) {
        // A context instance was injected at construction time -> use it
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    cwac.setParent(rootContext);
                }
                // 2. 初始化Spring环境包括加载配置文件等
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    // 1.2 通过contextAttribute进行初始化
    if (wac == null) {
        // No context instance was injected at construction time -> see if one
        // has been registered in the servlet context. If one exists, it is assumed
        // that the parent context (if any) has already been set and that the
        // user has performed any initialization such as setting the context id
        wac = findWebApplicationContext();
    }
    // 1.3 重新创建WebApplicationContext 实例
    if (wac == null) {
        // No context instance is defined for this servlet -> create a local one
        wac = createWebApplicationContext(rootContext);
    }

    // 3. 刷新
    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}


```

### 1 .寻找或创建对应的WebApplicationContext实例
```
// 1.2 通过contextAttribute进行初始化
protected WebApplicationContext findWebApplicationContext() {
    String attrName = getContextAttribute();
    if (attrName == null) {
        return null;
    }
    WebApplicationContext wac =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
    if (wac == null) {
        throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
    }
    return wac;
}

// 1.3 重新创建WebApplicationContext 实例
protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
    return createWebApplicationContext((ApplicationContext) parent);
}

protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
    // 获取servlet的初始化参数contextClass，如果没有配置默认为XmlWebApplicationContext.class
    Class<?> contextClass = getContextClass();
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException(
                "Fatal initialization error in servlet with name '" + getServletName() +
                "': custom WebApplicationContext class [" + contextClass.getName() +
                "] is not of type ConfigurableWebApplicationContext");
    }
    // 通过反射方式实例化contextClass
    ConfigurableWebApplicationContext wac =
            (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

    wac.setEnvironment(getEnvironment());
    // parent为在ContextLoaderListener中创建的实例 在ContextLoaderListener加载的时候初始化的WebApplicationContext类型实例
    wac.setParent(parent);
    // 获取contextConfigLocation属性，配置在servlet初始化参数中
    String configLocation = getContextConfigLocation();
    if (configLocation != null) {
        wac.setConfigLocation(configLocation);
    }
    // 初始化Spring环境包括加载配置文件等
    configureAndRefreshWebApplicationContext(wac);

    return wac;
}
```

### 2. configureAndRefreshWebApplicationContext
```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // The application context id is still set to its original default value
        // -> assign a more useful id based on available information
        if (this.contextId != null) {
            wac.setId(this.contextId);
        }
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
        }
    }

    wac.setServletContext(getServletContext());
    wac.setServletConfig(getServletConfig());
    wac.setNamespace(getNamespace());
    wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
    }

    postProcessWebApplicationContext(wac);
    applyInitializers(wac);
    // 加载配置文件及整合parent到wac
    wac.refresh();
}
```

### 3.刷新
onRefresh是FrameworkServlet类中提供的模板方法，在其子类DispatcherServlet中进行了重写，
主要用于刷新Spring在Web功能实现中所必须使用的全局变量。
DispatcherServlet.java
```
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    // 1. 初始化MultipartResolver
    initMultipartResolver(context);
    // 2. 初始化LocaleResolver
    initLocaleResolver(context);
    // 3. 初始化ThemeResolver
    initThemeResolver(context);
    // 4. 初始化HandlerMappings
    initHandlerMappings(context);
    // 5. 初始化HandlerAdapters
    initHandlerAdapters(context);
    // 6. 初始化HandlerExceptionResolvers
    initHandlerExceptionResolvers(context);
    // 7. 初始化RequestToViewNameTranslator
    initRequestToViewNameTranslator(context);
    // 8. 初始化ViewResolvers
    initViewResolvers(context);
    // 9. 初始化FlashMapManager
    initFlashMapManager(context);
}

// 1. 初始化MultipartResolver 在Spring中， MultipartResolver主要用来处理文件上传
public static final String MULTIPART_RESOLVER_BEAN_NAME = "multipartResolver";
private void initMultipartResolver(ApplicationContext context) {
    try {
        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.multipartResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.multipartResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // Default is no multipart resolver.
        this.multipartResolver = null;
        if (logger.isTraceEnabled()) {
            logger.trace("No MultipartResolver '" + MULTIPART_RESOLVER_BEAN_NAME + "' declared");
        }
    }
}

// 2. 初始化LocaleResolver 在Spring的国际化配置中一共有3种使用方式
基于URL参数的配置 AcceptHeaderLocaleResolver.java
基于session的配置 SessionLocaleResolver.java
基于cookie的国际化配置 CookieLocaleResolver.java
public static final String LOCALE_RESOLVER_BEAN_NAME = "localeResolver";
private void initLocaleResolver(ApplicationContext context) {
    try {
        this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.localeResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.localeResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // We need to use the default.
        this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No LocaleResolver '" + LOCALE_RESOLVER_BEAN_NAME +
                    "': using default [" + this.localeResolver.getClass().getSimpleName() + "]");
        }
    }
}

// 3. 初始化ThemeResolver 通过主题Theme来控制网页风格，这将进一步改善用户体验。
简单地说， 一个主题就是一组静态资源它们可以影响应用程序的视觉效果
Spring中的主题功能和国际化功能非常类似。Spring主题功能的构成主要包括如下内容
主题资源 ThemeSource是Spring中主题资源的接口，ResourceBundleThemeSource是ThemeSource接口默认实现类
主题解析器 FixedThemeResolver用于选择一个固定的主题
         CookieThemeResolver用于实现用户所选的主题，以cookie的形式存放在客户端的机器上
         SessionThemeResolver用于主题保存在用户的HTTP Session中
拦截器 如果需要根据用户请求来改变主题， 那么Spring 提供了一个已经实现的拦截器--ThemeChangelnterceptor
private void initThemeResolver(ApplicationContext context) {
    try {
        this.themeResolver = context.getBean(THEME_RESOLVER_BEAN_NAME, ThemeResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.themeResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.themeResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // We need to use the default.
        this.themeResolver = getDefaultStrategy(context, ThemeResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No ThemeResolver '" + THEME_RESOLVER_BEAN_NAME +
                    "': using default [" + this.themeResolver.getClass().getSimpleName() + "]");
        }
    }
}

// 4. 初始化HandlerMappings
当客户端发出Request时DispatcherServlet会将Request提交给HandlerMapping，
然后HanlerMapping根据WebApplicationContext的配置来回传给DispatcherServlet相应的Controller。
public static final String HANDLER_MAPPING_BEAN_NAME = "handlerMapping";
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;

    // 4.1 如果配置允许使用所有的HandlerMappings，则按接口类型加载所有HandlerMappings
    if (this.detectAllHandlerMappings) {
        // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
        Map<String, HandlerMapping> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // We keep HandlerMappings in sorted order.
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    }
    else {
        try {
            // 4.2 使用配置的唯一HandlerMapping
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default HandlerMapping later.
        }
    }

    // Ensure we have at least one HandlerMapping, by registering
    // a default HandlerMapping if no other mappings are found.
    // 4.3 使用默认HandlerMapping
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}

// 5. 初始化HandlerAdapters
HTTP请求处理器适自己器(HttpRequestHandlerAdapter)
简单控制器处理器适配器(SimpleControllerHandlerAdapter)
注解方法处理器适配器(AnnotationMethodHandlerAdapter)
Spring中所使用的Handler并没有任何特殊的联系，但是为了统一处理，Spring提供了不同情况下的适配器。
public static final String HANDLER_ADAPTER_BEAN_NAME = "handlerAdapter";
private void initHandlerAdapters(ApplicationContext context) {
    this.handlerAdapters = null;

    if (this.detectAllHandlerAdapters) {
        // Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
        Map<String, HandlerAdapter> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerAdapters = new ArrayList<>(matchingBeans.values());
            // We keep HandlerAdapters in sorted order.
            AnnotationAwareOrderComparator.sort(this.handlerAdapters);
        }
    }
    else {
        try {
            HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
            this.handlerAdapters = Collections.singletonList(ha);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default HandlerAdapter later.
        }
    }

    // Ensure we have at least some HandlerAdapters, by registering
    // default HandlerAdapters if no other adapters are found.
    if (this.handlerAdapters == null) {
        this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}

// 6. 初始化HandlerExceptionResolvers
Spring 会搜索所有注册在其环境中的实现了HandlerExceptionResolver接口的bean，逐个执行，直到返回了一个ModelAndView对象。
public static final String HANDLER_EXCEPTION_RESOLVER_BEAN_NAME = "handlerExceptionResolver";
private void initHandlerExceptionResolvers(ApplicationContext context) {
    this.handlerExceptionResolvers = null;

    if (this.detectAllHandlerExceptionResolvers) {
        // Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
        Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
                .beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerExceptionResolvers = new ArrayList<>(matchingBeans.values());
            // We keep HandlerExceptionResolvers in sorted order.
            AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
        }
    }
    else {
        try {
            HandlerExceptionResolver her =
                    context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
            this.handlerExceptionResolvers = Collections.singletonList(her);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, no HandlerExceptionResolver is fine too.
        }
    }

    // Ensure we have at least some HandlerExceptionResolvers, by registering
    // default HandlerExceptionResolvers if no other resolvers are found.
    if (this.handlerExceptionResolvers == null) {
        this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerExceptionResolvers declared in servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}

// 7. 初始化RequestToViewNameTranslator
当Controller处理器方法没有返回一个View对象或逻辑视图名称，并且在该方法中没有直接往respons巳的输出流里面写数据的时候，
Spring就会采用约定好的方式提供一个逻辑视图名称。这个逻辑视图名称是通过Spring定义的
org.Springframework.web.servlet.RequestToViewNameTranslator接口的getViewName方法来实现的，
Spring 已经给我们提供了一个它自己的实现，那就是org.Springframework.web.servlet.view.DefaultRequestToViewNameTranslator
public static final String REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME = "viewNameTranslator";
private void initRequestToViewNameTranslator(ApplicationContext context) {
    try {
        this.viewNameTranslator =
                context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.viewNameTranslator.getClass().getSimpleName());
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.viewNameTranslator);
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // We need to use the default.
        this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No RequestToViewNameTranslator '" + REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME +
                    "': using default [" + this.viewNameTranslator.getClass().getSimpleName() + "]");
        }
    }
}

// 8. 初始化ViewResolvers
public static final String VIEW_RESOLVER_BEAN_NAME = "viewResolver";
private void initViewResolvers(ApplicationContext context) {
    this.viewResolvers = null;

    if (this.detectAllViewResolvers) {
        // Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
        Map<String, ViewResolver> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.viewResolvers = new ArrayList<>(matchingBeans.values());
            // We keep ViewResolvers in sorted order.
            AnnotationAwareOrderComparator.sort(this.viewResolvers);
        }
    }
    else {
        try {
            ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
            this.viewResolvers = Collections.singletonList(vr);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Ignore, we'll add a default ViewResolver later.
        }
    }

    // Ensure we have at least one ViewResolver, by registering
    // a default ViewResolver if no other resolvers are found.
    if (this.viewResolvers == null) {
        this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No ViewResolvers declared for servlet '" + getServletName() +
                    "': using default strategies from DispatcherServlet.properties");
        }
    }
}

// 9. 初始化FlashMapManager
SpringMVC Flash attributes提供了一个请求存储属性，可供其他请求使用。在使用重定向时候非常必要，
例如Post/Redirect/Get模式。Flash attributes在重定向之前暂存（就像存在session中）以便重定向之后还能使用，并立即删除。
而FlashMapManager用于存储、检索、管理FlashMap实例。
public static final String FLASH_MAP_MANAGER_BEAN_NAME = "flashMapManager";
private void initFlashMapManager(ApplicationContext context) {
    try {
        this.flashMapManager = context.getBean(FLASH_MAP_MANAGER_BEAN_NAME, FlashMapManager.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.flashMapManager.getClass().getSimpleName());
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.flashMapManager);
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // We need to use the default.
        this.flashMapManager = getDefaultStrategy(context, FlashMapManager.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No FlashMapManager '" + FLASH_MAP_MANAGER_BEAN_NAME +
                    "': using default [" + this.flashMapManager.getClass().getSimpleName() + "]");
        }
    }
}
```

### 获取默认配置
DispatcherServlet.java
根据DispatcherServlet.properties配置获取默认配置Bean信息
```
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
    String key = strategyInterface.getName();
    String value = defaultStrategies.getProperty(key);
    if (value != null) {
        String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
        List<T> strategies = new ArrayList<>(classNames.length);
        for (String className : classNames) {
            try {
                Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
                Object strategy = createDefaultStrategy(context, clazz);
                strategies.add((T) strategy);
            }
            catch (ClassNotFoundException ex) {
                throw new BeanInitializationException(
                        "Could not find DispatcherServlet's default strategy class [" + className +
                        "] for interface [" + key + "]", ex);
            }
            catch (LinkageError err) {
                throw new BeanInitializationException(
                        "Unresolvable class definition for DispatcherServlet's default strategy class [" +
                        className + "] for interface [" + key + "]", err);
            }
        }
        return strategies;
    }
    else {
        return new LinkedList<>();
    }
}

private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";
static {
    // Load default strategy implementations from properties file.
    // This is currently strictly internal and not meant to be customized
    // by application developers.
    try {
        ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
    }
}
```

DispatcherServlet.properties
```
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```