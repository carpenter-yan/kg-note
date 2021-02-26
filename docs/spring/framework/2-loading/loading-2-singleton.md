##缓存中获取单例Bean
```
DefaultSingletonBeanRegistry.getSingleton
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //1.尝试从单例缓存中获取bean实例
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        //2.如果为空且正在创建中则在该对象上加锁并处理
        synchronized (this.singletonObjects) {
            //3.尝试从创建中单例缓存队列中获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            //4.如果未获取到并且允许提前应用则创建
            if (singletonObject == null && allowEarlyReference) {
                //5.从缓存中获取单例工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    //6.如果存在单例工厂，则创建。创建后加入创建中单例缓存，并删除工厂缓存
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

- singletonObjects：用于保存单例BeanName和创建bean实例之间的关系，beanName->beanInstance。
- singletonFactories：用于保存BeanName和创建bean的工厂之间的关系，beanName->ObjectFactory。
- earlySingletonObjects：也是保存BeanName和创建bean实例之间的关系，与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了，其目的是用来检测循环引用。
- registeredSingletons：用来保存当前所有巳注册的bean。