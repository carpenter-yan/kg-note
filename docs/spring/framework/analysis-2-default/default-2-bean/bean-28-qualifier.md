##解析qualifier元素
BeanDefinitionParserDelegate.parseQualifierElements

```
public void parseQualifierElements(Element beanEle, AbstractBeanDefinition bd) {
    NodeList nl = beanEle.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ELEMENT)) {
            parseQualifierElement((Element) node, bd);
        }
    }
}

public void parseQualifierElement(Element ele, AbstractBeanDefinition bd) {
    //1.获取type属性
    String typeName = ele.getAttribute(TYPE_ATTRIBUTE);
    if (!StringUtils.hasLength(typeName)) {
        error("Tag 'qualifier' must have a 'type' attribute", ele);
        return;
    }
    this.parseState.push(new QualifierEntry(typeName));
    try {
        AutowireCandidateQualifier qualifier = new AutowireCandidateQualifier(typeName);
        qualifier.setSource(extractSource(ele));
        String value = ele.getAttribute(VALUE_ATTRIBUTE);
        if (StringUtils.hasLength(value)) {
            qualifier.setAttribute(AutowireCandidateQualifier.VALUE_KEY, value);
        }
        //2.qualifier_attribute元素解析
        NodeList nl = ele.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ATTRIBUTE_ELEMENT)) {
                Element attributeEle = (Element) node;
                String attributeName = attributeEle.getAttribute(KEY_ATTRIBUTE);
                String attributeValue = attributeEle.getAttribute(VALUE_ATTRIBUTE);
                if (StringUtils.hasLength(attributeName) && StringUtils.hasLength(attributeValue)) {
                    BeanMetadataAttribute attribute = new BeanMetadataAttribute(attributeName, attributeValue);
                    attribute.setSource(extractSource(attributeEle));
                    qualifier.addMetadataAttribute(attribute);
                }
                else {
                    error("Qualifier 'attribute' tag must have a 'name' and 'value'", attributeEle);
                    return;
                }
            }
        }
        //3.将解析结果封装为AutowireCandidateQualifier并保存到qualifiers
        bd.addQualifier(qualifier);
    }
    finally {
        this.parseState.pop();
    }
}
```
见注释