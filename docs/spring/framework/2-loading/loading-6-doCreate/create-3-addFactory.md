##记录Bean的ObjectFactory 

AbstractAutowireCapableBeanFactory.doCreateBean中一段代码
```
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        //为避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }
```
earlySingletonExposure：从字面的意思理解就是提早曝光的单例
mbd.isSingleton()：此RootBeanDefinition 代表的是否是单例
this.allowCircularReferences ：是否允许循环依赖
isSingletonCurrentlyInCreation(beanName)：该bean是否在创建中


AbstractAutowireCapableBeanFactory.getEarlyBeanReference
```
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    /** 对bean再一次依赖引用，主要应用SmartInstantiationAwareBeanPostProcessor,
     * 其中我们熟知的AOP就是在这里将advice动态织人bean中，若没有则直接返回bean，不做任何处理 */
    if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                if (exposedObject == null) {
                    return null;
                }
            }
        }
    }
    return exposedObject;
}
```
每个Bean的创建流程:
  开始createBean(记录beanName)
->addSingletonFactory
->populateBean
->结束createBean(移除beanName)
如果有依赖循环则在populateBean时会循环重新开始该流程。以A->B->A循环依赖为例
Spring处理循环依赖的解决办法， 在B中创建依赖A时通过ObjectFactory提供的实例化方法来中断A中的属性填充，
使B中持有的A仅仅是刚刚初始化并没有填充任何属性的A，而这正初始化A的步骤还是在最开始创建A的时候进行的，
但是因为A与B中的A所表示的属性地址是一样的，所以在A中创建好的属性填充自然可以通过B中的A获取，
这样就解决了循环依赖的问题。