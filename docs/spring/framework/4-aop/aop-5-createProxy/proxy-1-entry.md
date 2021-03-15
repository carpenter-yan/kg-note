##创建代理

AbstractAutoProxyCreator.createProxy
```
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    // 获取当前类中相关属性
    proxyFactory.copyFrom(this);

    // 添加代理接口。
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    //1. 封装Advisor并加入到ProxyFactory中*
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    // 设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    // 定制代理
    customizeProxyFactory(proxyFactory);

    // 用来控制代理工厂被配置之后，是否还允许修改通知缺省值为false(即在代理被配置之后，不允许修改代理的配置)
    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    //2. 获取代理类
    return proxyFactory.getProxy(getProxyClassLoader());
}
```


```
protected Advisor[] buildAdvisors(String beanName, Object[] specificInterceptors) {
    // Handle prototypes correctly...
    // 解析注册的所有interceptorName
    Advisor[] commonInterceptors = resolveInterceptorNames();

    List<Object> allInterceptors = new ArrayList<Object>();
    if (specificInterceptors != null) {
        // 加入拦截器
        allInterceptors.addAll(Arrays.asList(specificInterceptors));
        if (commonInterceptors.length > 0) {
            if (this.applyCommonInterceptorsFirst) {
                allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
            }
            else {
                allInterceptors.addAll(Arrays.asList(commonInterceptors));
            }
        }
    }
    if (logger.isDebugEnabled()) {
        int nrOfCommonInterceptors = commonInterceptors.length;
        int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
        logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
                " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
    }

    Advisor[] advisors = new Advisor[allInterceptors.size()];
    for (int i = 0; i < allInterceptors.size(); i++) {
        // 拦截然进行封装转化为Advisor*
        advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
    }
    return advisors;
}

//由于Spring中涉及过多的拦截器、增强器、增强方法等方式来对逻辑进行增强，所以非常有必要统一封装成Advisor来进行代理的创建
public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
    // 如果要封装的对象本身就是Advisor类型的，那么无须再做过多处理
    if (adviceObject instanceof Advisor) {
        return (Advisor) adviceObject;
    }
    
    // 因为此封装方法只对Advisor 与Advice两种类型的数据有效，如果不是将不能封装
    if (!(adviceObject instanceof Advice)) {
        throw new UnknownAdviceTypeException(adviceObject);
    }
    Advice advice = (Advice) adviceObject;
    // 如果是Methodinterceptor类型则使用DefaultPointcutAdvisor封装
    if (advice instanceof MethodInterceptor) {
        // So well-known it doesn't even need an adapter.
        return new DefaultPointcutAdvisor(advice);
    }
    // 如果存在Advisor 的适配illi那么也同样馆要进行封装
    for (AdvisorAdapter adapter : this.adapters) {
        // Check that it is supported.
        if (adapter.supportsAdvice(advice)) {
            return new DefaultPointcutAdvisor(advice);
        }
    }
    throw new UnknownAdviceTypeException(advice);
}
```