## 解析mapper文件XMLMapperBuilder

Mapper文件的解析主要由XMLMapperBuilder类完成，以package扫描中的loadXmlResource()为入口开始。
MapperAnnotationBuilder.java
```
  private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
      String xmlResource = type.getName().replace('.', '/') + ".xml";
      // #1347
      InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
      if (inputStream == null) {
        // Search XML mapper that is not in the module but in the classpath.
        try {
          inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
        } catch (IOException e2) {
          // ignore, resource is not required
        }
      }
      if (inputStream != null) {
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        xmlParser.parse();
      }
    }
  }
```

根据package自动搜索加载的时候，约定俗成从classpath下加载接口的完整名，
比如org.mybatis.example.mapper.BlogMapper，就加载org/mybatis/example/mapper/BlogMapper.xml。
对于从package和class进来的mapper，如果找不到对应的文件，就忽略，因为这种情况下是允许SQL语句作为注解打在接口上的，所以xml文件不是必须的，
而对于直接声明的xml mapper文件，如果找不到的话会抛出IOException异常而终止，这在使用注解模式的时候需要注意。
加载到对应的mapper.xml文件后，调用XMLMapperBuilder进行解析。
在创建XMLMapperBuilder时，我们发现用到了configuration.getSqlFragments()，这就是我们在mapper文件中经常使用的可以被包含在其他语句中的SQL片段，
但是我们并没有初始化过，所以很有可能它是在解析过程中动态添加的，
创建了XMLMapperBuilder之后，在调用其parse()接口进行具体xml的解析，这和mybatis-config的逻辑基本上是一致的思路。

XMLMapperBuilder.java
```
  public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments, String namespace) {
    this(inputStream, configuration, resource, sqlFragments);
    this.builderAssistant.setCurrentNamespace(namespace);
  }
  
  public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }
  
  private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }
```
加载的基本逻辑和加载mybatis-config一样的过程，使用XPathParser进行总控，XMLMapperEntityResolver进行具体判断。

```
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }
  
  private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      // 1、解析缓存参照cache-ref
      cacheRefElement(context.evalNode("cache-ref"));
      // 2、解析缓存cache
      cacheElement(context.evalNode("cache"));
      // 3、解析参数映射parameterMap
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

### 1、解析缓存参照cache-ref。参照缓存顾名思义，就是共用其他缓存的设置。
XMLMapperBuilder.java
```
  <cache-ref namespace=”com.someone.application.data.SomeMapper”/>
  
  private void cacheRefElement(XNode context) {
    if (context != null) {
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }
  
```
缓存参考因为通过namespace指向其他的缓存。所以会出现第一次解析的时候指向的缓存还不存在的情况，所以需要在所有的mapper文件加载完成后进行二次处理，
不仅仅是缓存参考，其他的CRUD也一样。所以在XMLMapperBuilder.configuration中有很多的incompleteXXX，
这种设计模式类似于JVM GC中的mark and sweep，标记、然后处理。所以当捕获到IncompleteElementException异常时，没有终止执行，
而是将指向的缓存不存在的cacheRefResolver添加到configuration.incompleteCacheRef中。

### 2、解析缓存cache
```
  private void cacheElement(XNode context) {
    if (context != null) {
      String type = context.getStringAttribute("type", "PERPETUAL");
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      String eviction = context.getStringAttribute("eviction", "LRU");
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      Long flushInterval = context.getLongAttribute("flushInterval");
      Integer size = context.getIntAttribute("size");
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      boolean blocking = context.getBooleanAttribute("blocking", false);
      Properties props = context.getChildrenAsProperties();
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
```

默认情况下，mybatis使用的是永久缓存PerpetualCache，读取或设置各个属性默认值之后，调用builderAssistant.useNewCache构建缓存，
其中的CacheBuilder使用了build模式（在effective里面，建议有4个以上可选属性时，应该为对象提供一个builder便于使用），
只要实现org.apache.ibatis.cache.Cache接口，就是合法的mybatis缓存。

### 3、解析参数映射parameterMap
```
  private void parameterMapElement(List<XNode> list) {
    for (XNode parameterMapNode : list) {
      String id = parameterMapNode.getStringAttribute("id");
      String type = parameterMapNode.getStringAttribute("type");
      Class<?> parameterClass = resolveClass(type);
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<>();
      for (XNode parameterNode : parameterNodes) {
        String property = parameterNode.getStringAttribute("property");
        String javaType = parameterNode.getStringAttribute("javaType");
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        String resultMap = parameterNode.getStringAttribute("resultMap");
        String mode = parameterNode.getStringAttribute("mode");
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        Integer numericScale = parameterNode.getIntAttribute("numericScale");
        ParameterMode modeEnum = resolveParameterMode(mode);
        Class<?> javaTypeClass = resolveClass(javaType);
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        parameterMappings.add(parameterMapping);
      }
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
```
总体来说，目前已经不推荐使用参数映射，而是直接使用内联参数。所以我们这里就不展开细讲了。如有必要，我们后续会补上。

### 4、解析结果集映射resultMap