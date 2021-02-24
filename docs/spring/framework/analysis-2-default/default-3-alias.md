##解析alias元素
DefaultBeanDefinitionDocumentReader.processAliasRegistration

```
protected void processAliasRegistration(Element ele) {
    //1.beanName和alias正确性校验
    String name = ele.getAttribute(NAME_ATTRIBUTE);
    String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
    boolean valid = true;
    if (!StringUtils.hasText(name)) {
        getReaderContext().error("Name must not be empty", ele);
        valid = false;
    }
    if (!StringUtils.hasText(alias)) {
        getReaderContext().error("Alias must not be empty", ele);
        valid = false;
    }
    if (valid) {
        try {
            //2.注册别名
            getReaderContext().getRegistry().registerAlias(name, alias);
        }
        catch (Exception ex) {
            getReaderContext().error("Failed to register alias '" + alias +
                    "' for bean with name '" + name + "'", ele, ex);
        }
        //3.通知别名注册监听，目前为空实现
        getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
    }
}

```