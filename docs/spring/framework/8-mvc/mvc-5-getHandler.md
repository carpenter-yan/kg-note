DispatcherServlet.java
```
1.MultipartContent类型的request处理
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    // 判断是否是Multipart请求
    if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
        if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
            if (request.getDispatcherType().equals(DispatcherType.REQUEST)) {
                logger.trace("Request already resolved to MultipartHttpServletRequest, e.g. by MultipartFilter");
            }
        }
        else if (hasMultipartException(request) ) {
            logger.debug("Multipart resolution previously failed for current request - " +
                    "skipping re-resolution for undisturbed error rendering");
        }
        else {
            try {
                // 转换为MultipartHttpServletRequest类型
                return this.multipartResolver.resolveMultipart(request);
            }
            catch (MultipartException ex) {
                if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
                    logger.debug("Multipart resolution failed for error dispatch", ex);
                    // Keep processing error dispatch with regular request handle below
                }
                else {
                    throw ex;
                }
            }
        }
    }
    // If not returned before: return original request.
    return request;
}
```

// 2. 根据request信息寻找对应的Handler
DispatcherServlet.java
```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历所有的handerMapping直到获取到handler为止
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

AbstractHandlerMapping.java
```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 1. 根据request获取对应的handler 
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        // 2. 如果没有对应request的handler则使用默认的handler
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    // Bean name or resolved handler?
    // 3. 当查找的controller为String类型时，那就意味着返回的是配置的bean名称，需要根据bean名称查找对应的bean
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }

    // 4. 通过getHandlerExecutionChain方法对返回的Handler进行封装，将handeler组成责任链
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

    if (logger.isTraceEnabled()) {
        logger.trace("Mapped to " + handler);
    }
    else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
        logger.debug("Mapped to " + executionChain.getHandler());
    }

    if (CorsUtils.isCorsRequest(request)) {
        CorsConfiguration globalConfig = this.corsConfigurationSource.getCorsConfiguration(request);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }

    return executionChain;
}
```

AbstractUrlHandlerMapping.java
```
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
    // 1. 截取用于匹配的url有效路径
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    // 2. 根据路径寻找Handler
    Object handler = lookupHandler(lookupPath, request);
    if (handler == null) {
        // We need to care for the default handler directly, since we need to
        // expose the PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE for it as well.
        Object rawHandler = null;
        // 3. 如果请求的路径仅仅是"/"，那么使用RootHandler进行处理
        if ("/".equals(lookupPath)) {
            rawHandler = getRootHandler();
        }
        // 4. 无法找到handler则使用默认handler
        if (rawHandler == null) {
            rawHandler = getDefaultHandler();
        }
        if (rawHandler != null) {
            // Bean name or resolved handler?
            if (rawHandler instanceof String) {
                String handlerName = (String) rawHandler;
                // 5. 根据beanName 获取对应的bean
                rawHandler = obtainApplicationContext().getBean(handlerName);
            }
            validateHandler(rawHandler, request);
            handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
        }
    }
    return handler;
}

protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
    // Direct match?
    // 1. 直接匹配情况的处理
    Object handler = this.handlerMap.get(urlPath);
    if (handler != null) {
        // Bean name or resolved handler?
        if (handler instanceof String) {
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
        validateHandler(handler, request);
        return buildPathExposingHandler(handler, urlPath, urlPath, null);
    }

    // Pattern match?
    // 2. 通配符匹配的处理
    List<String> matchingPatterns = new ArrayList<>();
    for (String registeredPattern : this.handlerMap.keySet()) {
        if (getPathMatcher().match(registeredPattern, urlPath)) {
            matchingPatterns.add(registeredPattern);
        }
        else if (useTrailingSlashMatch()) {
            if (!registeredPattern.endsWith("/") && getPathMatcher().match(registeredPattern + "/", urlPath)) {
                matchingPatterns.add(registeredPattern +"/");
            }
        }
    }

    String bestMatch = null;
    Comparator<String> patternComparator = getPathMatcher().getPatternComparator(urlPath);
    if (!matchingPatterns.isEmpty()) {
        matchingPatterns.sort(patternComparator);
        if (logger.isTraceEnabled() && matchingPatterns.size() > 1) {
            logger.trace("Matching patterns " + matchingPatterns);
        }
        bestMatch = matchingPatterns.get(0);
    }
    if (bestMatch != null) {
        handler = this.handlerMap.get(bestMatch);
        if (handler == null) {
            if (bestMatch.endsWith("/")) {
                handler = this.handlerMap.get(bestMatch.substring(0, bestMatch.length() - 1));
            }
            if (handler == null) {
                throw new IllegalStateException(
                        "Could not find handler for best pattern match [" + bestMatch + "]");
            }
        }
        // Bean name or resolved handler?
        if (handler instanceof String) {
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
        validateHandler(handler, request);
        String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestMatch, urlPath);

        // There might be multiple 'best patterns', let's make sure we have the correct URI template variables
        // for all of them
        Map<String, String> uriTemplateVariables = new LinkedHashMap<>();
        for (String matchingPattern : matchingPatterns) {
            if (patternComparator.compare(bestMatch, matchingPattern) == 0) {
                Map<String, String> vars = getPathMatcher().extractUriTemplateVariables(matchingPattern, urlPath);
                Map<String, String> decodedVars = getUrlPathHelper().decodePathVariables(request, vars);
                uriTemplateVariables.putAll(decodedVars);
            }
        }
        if (logger.isTraceEnabled() && uriTemplateVariables.size() > 0) {
            logger.trace("URI variables " + uriTemplateVariables);
        }
        return buildPathExposingHandler(handler, bestMatch, pathWithinMapping, uriTemplateVariables);
    }

    // No handler found...
    return null;
}

protected Object buildPathExposingHandler(Object rawHandler, String bestMatchingPattern,
        String pathWithinMapping, @Nullable Map<String, String> uriTemplateVariables) {

    HandlerExecutionChain chain = new HandlerExecutionChain(rawHandler);
    chain.addInterceptor(new PathExposingHandlerInterceptor(bestMatchingPattern, pathWithinMapping));
    if (!CollectionUtils.isEmpty(uriTemplateVariables)) {
        chain.addInterceptor(new UriTemplateVariablesHandlerInterceptor(uriTemplateVariables));
    }
    return chain;
}
在函数中我们看到了通过将Handler以参数形式传入，井构建HandlerExecutionChain类型实例，加入了两个拦截器。
此时我们似乎已经了解了Spring这样大番周折的目的。链处理机制


protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
            (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}
getHandlerExecutionChain函数最主要的目的是将配置中的对应拦截器加入到执行链中，以保证这些拦截器可以有效地作用于目标对象。
```