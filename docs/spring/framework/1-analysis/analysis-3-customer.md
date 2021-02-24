	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}
##解析自定义标签
BeanDefinitionParserDelegate.parseCustomElement
```
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

//containingBd为父类Bean，顶层元素解析时设置为null
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    //1.获取自动一命名空间
    String namespaceUri = getNamespaceURI(ele);
    //2.根据命名空间找到对应的NamespaceHandler
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    //3.调用自定义的NamespaceHandle进行解析
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```
在构建ReaderContext时NamespaceHandler默认为DefaultNamespaceHandlerResolver。参考<prepareForAnalysis.md>

###提取自定义标签处理器
```
public NamespaceHandler resolve(String namespaceUri) {
    //1.获取所有已配置的handler映射
    Map<String, Object> handlerMappings = getHandlerMappings();
    //2.根据命名空间找到对应的信息
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    if (handlerOrClassName == null) {
        return null;
    }
    //3.已经解析过直接返回
    else if (handlerOrClassName instanceof NamespaceHandler) {
        return (NamespaceHandler) handlerOrClassName;
    }
    else {
        //4.1未解析则返回类名
        String className = (String) handlerOrClassName;
        try {
            //4.2使用反射加载类
            Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
            if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                        "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
            }
            //4.3初始化类
            NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
            //4.4调用初始化方法
            namespaceHandler.init();
            //4.5记录在缓存中
            handlerMappings.put(namespaceUri, namespaceHandler);
            return namespaceHandler;
        }
        catch (ClassNotFoundException ex) {
            throw new FatalBeanException("NamespaceHandler class [" + className + "] for namespace [" +
                    namespaceUri + "] not found", ex);
        }
        catch (LinkageError err) {
            throw new FatalBeanException("Invalid NamespaceHandler class [" + className + "] for namespace [" +
                    namespaceUri + "]: problem with handler class file or dependent class", err);
        }
    }
}

private Map<String, Object> getHandlerMappings() {
    if (this.handlerMappings == null) {
        synchronized (this) {
            if (this.handlerMappings == null) {
                try {
                    //1.handlerMappingsLocation初始化为META-INF/spring.handlers
                    Properties mappings =
                            PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Loaded NamespaceHandler mappings: " + mappings);
                    }
                    Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
                    //2.将Properties格式的mappings合并到Map格式的handlerMappings中
                    CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
                    this.handlerMappings = handlerMappings;
                }
                catch (IOException ex) {
                    throw new IllegalStateException(
                            "Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
                }
            }
        }
    }
    return this.handlerMappings;
}
```

###标签解析
NamespaceHandlerSupport.parse
```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    return findParserForElement(element, parserContext).parse(element, parserContext);
}

//NamespaceHandlerSupport.findParserForElement
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
    //1.获取元素的localName <tx:advice>其中advice为localName
    String localName = parserContext.getDelegate().getLocalName(element);
    BeanDefinitionParser parser = this.parsers.get(localName);
    if (parser == null) {
        parserContext.getReaderContext().fatal(
                "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
    }
    return parser;
}

//AbstractBeanDefinitionParser.parse
public final BeanDefinition parse(Element element, ParserContext parserContext) {
    //1.调用用户实现的解析方法
    AbstractBeanDefinition definition = parseInternal(element, parserContext);
    if (definition != null && !parserContext.isNested()) {
        try {
            //2.将解析后的BeanDefinition转换为BeanDefinitionHolder
            String id = resolveId(element, definition, parserContext);
            if (!StringUtils.hasText(id)) {
                parserContext.getReaderContext().error(
                        "Id is required for element '" + parserContext.getDelegate().getLocalName(element)
                                + "' when used as a top-level tag", element);
            }
            String[] aliases = null;
            if (shouldParseNameAsAliases()) {
                String name = element.getAttribute(NAME_ATTRIBUTE);
                if (StringUtils.hasLength(name)) {
                    aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
                }
            }
            BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
            //3.注册bean
            registerBeanDefinition(holder, parserContext.getRegistry());
            //4.通知注册的监听器
            if (shouldFireEvents()) {
                BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
                postProcessComponentDefinition(componentDefinition);
                parserContext.registerComponent(componentDefinition);
            }
        }
        catch (BeanDefinitionStoreException ex) {
            parserContext.getReaderContext().error(ex.getMessage(), element);
            return null;
        }
    }
    return definition;
}

//AbstractSingleBeanDefinitionParser.parseInternal
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
    String parentName = getParentName(element);
    if (parentName != null) {
        builder.getRawBeanDefinition().setParentName(parentName);
    }
    //1.获取自定义标签中的class
    Class<?> beanClass = getBeanClass(element);
    if (beanClass != null) {
        builder.getRawBeanDefinition().setBeanClass(beanClass);
    }
    //2.如子类未重新getBeanClass则尝试调用getBeanClassName获取类名
    else {
        String beanClassName = getBeanClassName(element);
        if (beanClassName != null) {
            builder.getRawBeanDefinition().setBeanClassName(beanClassName);
        }
    }
    builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
    //3.获取scope属性
    if (parserContext.isNested()) {
        // Inner bean definition must receive same scope as containing bean.
        builder.setScope(parserContext.getContainingBeanDefinition().getScope());
    }
    //4.获取lazy-init属性
    if (parserContext.isDefaultLazyInit()) {
        // Default-lazy-init applies to custom bean definitions as well.
        builder.setLazyInit(true);
    }
    //5.调用用户实现的方法解析元素
    doParse(element, parserContext, builder);
    return builder.getBeanDefinition();
}
```
