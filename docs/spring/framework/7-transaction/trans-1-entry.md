##spring事务

###使用示例
<tx:annotation-driven transaction-manager＝"transactionManager"/>
<bean id="transactionManager" class="org.Springframework.jdbc.datasource.DataSourceTransactionManager" >
    <property name="dataSource" ref="dataSource" />
</bean>

###分析入口
使用annotation-driven全文搜索，锁定TxNamespaceHandler类
TxNamespaceHandler.init
```
public void init() {
    this.registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
    this.registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
    this.registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
}
```

在遇到诸如＜tx:annotation-driven为开头的配置后，Spring都会使用AnnotationDrivenBeanDefinitionParser类的parse方法进行解析
org.springframework.transaction.config.AnnotationDrivenBeanDefinitionParser.parse
```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    this.registerTransactionalEventListenerFactory(parserContext);
    String mode = element.getAttribute("mode");
    if ("aspectj".equals(mode)) {
        this.registerTransactionAspect(element, parserContext);
    } else {
        AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext);
    }

    return null;
}
```
在解析中存在对于mode属性的判断，根据代码，如果我们需要使用AspectJ的方式进行事务切入（ Spring中的事务是以AOP为基础的），
那么可以使用这样的配置：<tx:annotation driven-transaction-manager="transactionManager" mode="aspectj" />