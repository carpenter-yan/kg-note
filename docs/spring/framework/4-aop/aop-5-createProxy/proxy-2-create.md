##创建代理

AbstractAutoProxyCreator.createProxy
对于代理类的创建及处理，Spring 委托给了ProxyFactory去处理，而在此函数中主要是对
ProxyFactory的初始化操作，进而对真正的创建代理做准备
```
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    //1. 获取当前类中相关属性
    proxyFactory.copyFrom(this);

    if (!proxyFactory.isProxyTargetClass()) {
        // 决定对于给定的bean是否应该使用targetClass而不是它的接口代理，检查proxyTargeClass设置以及preserveTargetClass属性
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            //2. 添加代理接口
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 3. 封装Advisor并加入到ProxyFactory中。
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    //4. 设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    //5. 定制代理 子类可以在此函数中进行对ProxyFactory的进一步封装
    customizeProxyFactory(proxyFactory);
    // 用来控制代理工厂被配置之后，是否还允许修改通知缺省值为false(即在代理被配置之后，不允许修改代理的配置)
    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    //6. 进行获取代理操作。
    return proxyFactory.getProxy(getProxyClassLoader());
}
```
buildAdvisors参考<proxy-1-entry.md>


ProxyFactory.getProxy
```
public Object getProxy(ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

### 1.创建代理
```
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}
```

DefaultAopProxyFactory.createAopProxy
```
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    //0. 判断使用哪种代理方式
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        //1. 使用cglib代理
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        //2. 使用jdk代理
        return new JdkDynamicAopProxy(config);
    }
}
```
从if中的判断条件可以看到3个方面影响着Spring的判断。
1. optimize：用来控制通过CGLIB创建的代理是否使用激进的优化策略。除非完全了解AOP代理如何处理优化，
   否则不推荐用户使用这个设置。目前这个属性仅用于CGLIB代理，对于JDK动态代理（默认代理）无效。
2. proxyTargetClass：这个属性为true时，目标类本身被代理而不是目标类的接口。
   如果这个属性值被设为true, CGLIB 代理将被创建，设置方式为<aop:aspectj-autoproxy proxy-target-class="true"/>
3. hasNoUserSuppliedProxyInterfaces：是否存在代理接口。
