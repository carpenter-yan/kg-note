##DispatcherServlet

在Spring中，ContextLoaderListener只是辅助功能，用于创建WebApplicationContext类型实例，
而真正的逻辑实现其实是在DispatcherServlet中进行的，DispatcherServlet是实现servlet接口的实现类。
我们在其父类HttpServletBean中找到了该方法。
```
public final void init() throws ServletException {

    // Set bean properties from init parameters.
    // 1 .封装及验证初始化参数
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            // 2. 将当前servlet实例转化成BeanWrapper实例，从而能够以Spring的方式来对init-pararm的值进行注入
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            // 3. 注册相对于Resource的属性编辑器，一旦遇到Resource类型的属性将会使用ResourceEditor进行解析
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            // 空实现， 留给子类覆盖
            initBeanWrapper(bw);
            // 4. 属性注入 我们最常用的属性注入无非是contextAttribute、contextClass、nameSpace、contextConfigLocation等
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }

    // Let subclasses do whatever initialization they like.
    // 5 . servletBean的初始化，父类FrameworkServlet覆盖了HttpServletBean中的initServletBean
    initServletBean();
}

```

HttpServletBean.java
封装属性主要是对初始化的参数进行封装，也就是servlet中配置的<init-param>中配置的封装
```
public ServletConfigPropertyValues(ServletConfig config, Set<String> requiredProperties)
        throws ServletException {

    Set<String> missingProps = (!CollectionUtils.isEmpty(requiredProperties) ?
            new HashSet<>(requiredProperties) : null);

    Enumeration<String> paramNames = config.getInitParameterNames();
    while (paramNames.hasMoreElements()) {
        String property = paramNames.nextElement();
        Object value = config.getInitParameter(property);
        addPropertyValue(new PropertyValue(property, value));
        if (missingProps != null) {
            missingProps.remove(property);
        }
    }

    // Fail if we are still missing properties.
    if (!CollectionUtils.isEmpty(missingProps)) {
        throw new ServletException(
                "Initialization from ServletConfig for servlet '" + config.getServletName() +
                "' failed; the following required properties were missing: " +
                StringUtils.collectionToDelimitedString(missingProps, ", "));
    }
}
```

FrameworkServlet.java
```
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    try {
        this.webApplicationContext = initWebApplicationContext();
        // 设计为子类覆盖
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
                "shown which may lead to unsafe logging of potentially sensitive data" :
                "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}
```