##创建AOP代理

AnnotationAwareAspectJAutoProxyCreator实现了BeanPostProcessor接口，而实现BeanPostProcessor后，
当Spring 加载这个Bean时会在实例化前调用其postProcessAfterInitialization方法，而我们对于AOP逻辑的分析也由此开始。

父类AbstractAutoProxyCreator.postProcessAfterInitialization
```
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean != null) {
        //根据给定的bean的class和name构建一个key， 格式：beanClassName_beanName
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            //如果它适合被代理， 则需要封装指定bean
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //1. 如果已经处理过
    if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    
    //2. 无须增强
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    
    //3. 给定的bean类是否代表一个基础设施类，基础设施类不应代理，或者配置了指定bean不需要自动代理
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    //4. 如果存在增强方法则创建代理
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    //5. 如果获取到了增强则需要针对增强创建代理
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        //6. 创建代理
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```
创建代理主要包含了两个步骤。
1. 获取增强方法或者增强器。
2. 根据获取的增强进行代理。
AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean
```
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //1. 获取所有的增强
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    //2. 寻找所有增强中适用于bean的增强并应用
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```