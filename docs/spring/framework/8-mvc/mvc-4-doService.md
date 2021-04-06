## DispatcherServlet的逻辑处理

直接查看DispatcherServlet中对于这两个函数的逻辑实现doGet()和doPost()

FrameworkServlet.java
```
protected final void doPost(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    processRequest(request, response);
}

protected final void doPut(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    processRequest(request, response);
}

protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    // 记录当前时间，用于计算web请求的处理时间
    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;
    // 1. 为了保证当前线程的LocaleContext以及RequestAttributes可以在当前请求后还能恢复，提取当前线程的两个属性
    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);
    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

    // 2. 根据当前request创建对应的LocaleContext和RequestAttributes，并绑定到当前线程
    initContextHolders(request, localeContext, requestAttributes);

    try {
        // 3. 委托给doService方法进一步处理
        doService(request, response);
    }
    catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    }
    catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    }

    finally {
        // 4. 请求处理结束后恢复线程到原始状态
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }
        logResult(request, response, failureCause, asyncManager);
        // 5. 请求处理结束后无论成功与否发布事件通知
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}
```

DispatcherServlet.java
在doService同样是一些准备工作，将已经初始化的功能辅助工具变量，
比如localeResolver, themeResolver等设置在request属性中，而这些属性会在接下来的处理中派上用场
```
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    logRequest(request);

    // Keep a snapshot of the request attributes in case of an include,
    // to be able to restore the original attributes after the include.
    Map<String, Object> attributesSnapshot = null;
    if (WebUtils.isIncludeRequest(request)) {
        attributesSnapshot = new HashMap<>();
        Enumeration<?> attrNames = request.getAttributeNames();
        while (attrNames.hasMoreElements()) {
            String attrName = (String) attrNames.nextElement();
            if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                attributesSnapshot.put(attrName, request.getAttribute(attrName));
            }
        }
    }

    // Make framework objects available to handlers and view objects.
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    if (this.flashMapManager != null) {
        FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        }
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
    }

    try {
        doDispatch(request, response);
    }
    finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            // Restore the original attribute snapshot, in case of an include.
            if (attributesSnapshot != null) {
                restoreAttributesAfterInclude(request, attributesSnapshot);
            }
        }
    }
}
```

### 业务流转主方法
DispatcherServlet.java
```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 1. 如果是MultipartContent类型的request则转换request为MultipartHttpServletRequest类型的request
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            // 2. 根据request信息寻找对应的Handler
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                // 3.如果没有找到对应的handler则通过response反馈错误信息
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            // 4. 根据当前的handler寻找对应的HandlerAdapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            // 4. 如果当前handler支持last-modified头处理
            /**  在研究Spring对缓存处理的功能支持前，我们先了解一个概念：Last-Modified缓存机制。
             * 1. 在客户端第一次输入URL时，服务器端会返回内容和状态码200，表示请求成功，
             * 同时会添加一个"Last-Modified"的响应头，表示此文件在服务器上的最后更新时间，
             * 例如，"Last-Modified:W1时，14Mar2012 10:22:42 GMT"表示最后更新时间为（2012-03-14 10:22 ）。
             * 2. 客户端第二次请求此URL 时，客户端会向服务器发送请求头“If-Modified-Since"，
             * 询问服务器该时间之后当前请求内容是否有被修改过，如"If-Modified-Since:Wed,14 Mar 2012 10:22:42 GMT"，
             * 如果服务器端的内容没有变化，则自动返回HTTP304状态码（只要响应头，内容为空，这样就节省了网络带宽）。
             * Spring 提供的对Last-Modified 机制的支持，只需要实现LastModified接口*/
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 5. 拦截器的preHandler方法的调用
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 6. 真正的激活handler并返回视图
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            // 7. 视图名称转换应用于需要添加前缀后缀的情况
            applyDefaultViewName(processedRequest, mv);
            // 8. 应用所有拦截器的postHandle方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 9. 如果在Handler 实例的处理中返回了view ，那么需要做页面的处理
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        // 10.完成处理激活触发器
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

