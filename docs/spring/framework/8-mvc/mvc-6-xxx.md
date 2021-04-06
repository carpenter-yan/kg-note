### 没找到对应的Handler 的错误处理
DispatcherServlet.java
```
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (pageNotFoundLogger.isWarnEnabled()) {
        pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
    }
    if (this.throwExceptionIfNoHandlerFound) {
        throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
                new ServletServerHttpRequest(request).getHeaders());
    }
    else {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
    }
}
```
每个请求都应该对应着Handler，一旦遇到没有找到Handler的情况（正常情况下如果没有URL匹配的Handler，开发人员可以设置默认的Handler
来处理请求，但是如果默认请求也未设置就会出现Handler为空的情况），就只能通过response向用户返回错误信息。

### 根据当前Handler寻找对应的HandlerAdapter
DispatcherServlet.java
```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
            "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```
对于获取适配器的逻辑无非就是遍历所有适配器来选择合适的适配器并返回它，而某个适配器是否适用于当前的Handler逻辑被封装在具体的适配器中


SimpleControllerHandlerAdapter.java
```
public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {

    return ((Controller) handler).handleRequest(request, response);
}
```
此处以SimpleControllerHandlerAdapter为例。

AbstractController.java
```
public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response)
        throws Exception {

    if (HttpMethod.OPTIONS.matches(request.getMethod())) {
        response.setHeader("Allow", getAllowHeader());
        return null;
    }

    // Delegate to WebContentGenerator for checking and preparing.
    checkRequest(request);
    prepareResponse(response);

    // Execute handleRequestInternal in synchronized block if required.
    // 如果需要在session内的同步执行
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                // 调用业务逻辑处理
                return handleRequestInternal(request, response);
            }
        }
    }

    // 调用业务逻辑处理
    return handleRequestInternal(request, response);
}
```
