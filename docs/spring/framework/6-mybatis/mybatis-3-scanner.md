##MapperScannerConfigurer

###使用示例
<bean class="org.mybatis.Spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="test.mybatis.dao"/>
</bean>
你可以使用分号或逗号作为分隔符设置多于一个的包路径。被发现的映射器将会使用Spring对自动侦测组件默认的命名策略来命名。
也就是说，如果没有发现注解，它就会使用映射器的非大写的非完全限定类名。但是如果发现了@Component或JSR-330@Named注解，它会获取名称。

