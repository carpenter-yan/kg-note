##解析lookup-method子元素
<bean id="getBeanTest" class="test.lookup.app.GetBeanTest">
    <lookup-method name="getBean" bean＝teacher "/>
</bean>
<bean id="teacher" class="test.lookup.bean.Teacher" />
lookup-method通常我们称它为获取器注入。是一种特殊的方法注入，它是把一个方法声明为返回某种类型的bean ，但实际要返回的bean是在配置文件里面配置的，此方法可用在设计有些可插拔的功能上，解除程序依赖。具体使用例子请参考书籍

BeanDefinitionParserDelegate.parseLookupOverrideSubElements

```
public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
            Element ele = (Element) node;
            String methodName = ele.getAttribute(NAME_ATTRIBUTE);
            String beanRef = ele.getAttribute(BEAN_ELEMENT);
            LookupOverride override = new LookupOverride(methodName, beanRef);
            override.setSource(extractSource(ele));
            overrides.addOverride(override);
        }
    }
}
```
