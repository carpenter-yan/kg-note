##MapperFactoryBean的创建

###配置示例
<bean id="userMapper" class="org.mybatis.Spring.mapper.MapperFactoryBean">
    <property name="mapperInterface" value="test.mybatis.dao.UserMapper"></property>
    <property name="sqlSessionFactory" ref="sqlSessionFactory" ></property>
</bean>


MapperFactoryBean类实现了InitializingBean与FactoryBean。

###InitializingBean.afterPropertiesSet接口
DaoSupport.afterPropertiesSet
```
public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
    //1. 校验dao的配置
    this.checkDaoConfig();

    try {
        //2. 模板方法留个子类重写
        this.initDao();
    } catch (Exception var2) {
        throw new BeanInitializationException("Initialization of DAO failed", var2);
    }
}
```

MapperFactoryBean.checkDaoConfig
```
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
  
  //SqlSessionDaoSupport.checkDaoConfig
  protected void checkDaoConfig() {
    notNull(this.sqlSessionTemplate, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
  }
```

###InitializingBean.getObject接口
```
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
```
