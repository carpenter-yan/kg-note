##Spring生命周期
AbstractAutowireCapableBeanFactory.doCreateBean开始进入
1. 调用createBeanInstance实例化(1)
2. 调用populateBean属性注入(2)
3. 调用initializeBean方法进行初始化(3) 
   3.1 调用invokeAwareMethods对aware接口进行处理(4)
   3.2 调用applyBeanPostProcessorsBeforeInitialization(5)
   3.2 调用invokeInitMethods方法
       3.2.1 调用InitializingBean的afterPropertiesSet方法(6)
       3.2.2 调用用户自定义init-method(7)
   3.3 调用applyBeanPostProcessorsAfterInitialization(8)
4. 调用registerDisposableBeanIfNecessary(9)
其中(5)(8)BeanPostProcessors针对所有Bean处理
