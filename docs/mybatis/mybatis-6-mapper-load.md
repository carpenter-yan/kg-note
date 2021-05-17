## 解析mapper文件XMLMapperBuilder

### 加载xml文件
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
package自动加载时，从classpath下加载接口的完整名，如org.mybatis.mapper.BlogMapper，就加载org/mybatis/mapper/BlogMapper.xml。
对于从package和class进来的mapper，如果找不到对应的文件，就忽略，因为这种情况下是允许SQL语句作为注解打在接口上的，所以xml文件不是必须的，
而对于直接声明的xml mapper文件，如果找不到的话会抛出IOException异常而终止，这在使用注解模式的时候需要注意。
加载到对应的mapper.xml文件后，调用XMLMapperBuilder进行解析。创建XMLMapperBuilder时，用到了configuration.getSqlFragments()，
这就是我们在mapper文件中经常使用的可以被包含在其他语句中的SQL片段，但是我们并没有初始化过，所以很有可能它是在解析过程中动态添加的，
创建了XMLMapperBuilder之后，在调用其parse()接口进行具体xml的解析，这和mybatis-config的逻辑基本上是一致的思路。

### 创建XMLMapperBuilder
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

### 解析mapper配置文件
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
      // 4. 解析resultMap
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // 5. 解析sql节点
      sqlElement(context.evalNodes("/mapper/sql"));
      // 6. 解析CRUD语句
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
```
  private void resultMapElements(List<XNode> list) throws Exception {
    for (XNode resultMapNode : list) {
      try {
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }
  
  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList(), null);
  }
  
  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    String extend = resultMapNode.getStringAttribute("extends");
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    Class<?> typeClass = resolveClass(type);
    if (typeClass == null) {
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      //4.1. 解析constructor子元素
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        //4.2. 解析discriminator子元素
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      // 4.3 构建解析器后解析
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }

```

### 4.1. 解析constructor子元素
resultMap里面可以包含多种子节点，每个节点都有具体的方法进行解析，这也体现了单一职责原则。
在resultMapElement中，主要是解析resultMap节点本身并循环遍历委托给具体的方法处理。
下面来看构造器的解析。构造器主要用于没有默认构造器或者有多个构造器的情况
```
  <constructor>
   <idArg column="id" javaType="int"/>
   <arg column="username" javaType="String"/>
   <arg column="age" javaType="_int"/>
  </constructor>
  
  private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    List<XNode> argChildren = resultChild.getChildren();
    for (XNode argChild : argChildren) {
      List<ResultFlag> flags = new ArrayList<>();
      flags.add(ResultFlag.CONSTRUCTOR);
      if ("idArg".equals(argChild.getName())) {
        flags.add(ResultFlag.ID);
      }
      resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
  }
```

buildResultMappingFromContext是个公共工具方法，会被反复使用，我们来看下它的具体实现
```
  private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    String property;
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
      property = context.getStringAttribute("name");
    } else {
      property = context.getStringAttribute("property");
    }
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    String nestedResultMap = context.getStringAttribute("resultMap",
        processNestedResultMappings(context, Collections.<ResultMapping> emptyList(), resultType));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
  }
```
上述过程主要用于获取各个属性，其中唯一值得注意的是processNestedResultMappings，它用于解析包含的association或collection复合类型,
这些复合类型可以使用外部定义的公用resultMap或者内嵌resultMap, 所以这里的处理逻辑是如果是外部resultMap就获取对应resultMap的名称,
如果没有,那就动态生成一个。如果自动生成的话，其resultMap id通过调用XNode.getValueBasedIdentifier()来获得。
由于colletion和association、discriminator里面还可以包含复合类型，所以将进行递归解析直到所有的子元素都为基本列位置，
它在使用层面的目的在于将关系模型映射为对象树模型。


```
<resultMap id="blogResult" type="Blog">
  <id property="id" column="blog_id" />
  <result property="title" column="blog_title"/>
  <association property="author" column="blog_author_id" javaType="Author" resultMap="authorResult"/>
  <collection property="posts" javaType="ArrayList" column="id" ofType="Post" select="selectPostsForBlog"/>
</resultMap>
  private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings, Class<?> enclosingType) throws Exception {
    if ("association".equals(context.getName())
        || "collection".equals(context.getName())
        || "case".equals(context.getName())) {
      if (context.getStringAttribute("select") == null) {
        validateCollection(context, enclosingType);
        ResultMap resultMap = resultMapElement(context, resultMappings, enclosingType);
        return resultMap.getId();
      }
    }
    return null;
  }
```
对于其中的每个非select属性映射，调用resultMapElement进行递归解析。其中的case节点主要用于鉴别器情况，
得到各属性之后，调用builderAssistant.buildResultMapping最后创建ResultMap
MapperBuilderAssistant.java
```
  public ResultMapping buildResultMapping(
      Class<?> resultType,
      String property,
      String column,
      Class<?> javaType,
      JdbcType jdbcType,
      String nestedSelect,
      String nestedResultMap,
      String notNullColumn,
      String columnPrefix,
      Class<? extends TypeHandler<?>> typeHandler,
      List<ResultFlag> flags,
      String resultSet,
      String foreignColumn,
      boolean lazy) {
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    List<ResultMapping> composites = parseCompositeColumnName(column);
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
  }
```

### 4.2. 解析discriminator子元素
鉴别器非常容易理解,它的表现很像Java语言中的switch语句。定义鉴别器也是通过column和javaType属性来唯一标识，
column是用来确定某个字段是否为鉴别器， JavaType是需要被用来保证等价测试的合适类型
```
<discriminator javaType="int" column="vehicle_type">
    <case value="1" resultMap="carResult"/>
    <case value="2" resultMap="truckResult"/>
    <case value="3" resultMap="vanResult"/>
    <case value="4" resultMap="suvResult"/>
  </discriminator>
  
  对于上述的鉴别器，如果 vehicle_type=1, 那就会使用下列这个结果映射。
  <resultMap id="carResult" type="Car">
    <result property="doorCount" column="door_count" />
  </resultMap>

  private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    Class<?> javaTypeClass = resolveClass(javaType);
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    Map<String, String> discriminatorMap = new HashMap<>();
    for (XNode caseChild : context.getChildren()) {
      String value = caseChild.getStringAttribute("value");
      String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings, resultType));
      discriminatorMap.put(value, resultMap);
    }
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
  }
```
其逻辑和之前处理构造器的时候类似，同样的使用build建立鉴别器并返回鉴别器实例，鉴别器中也可以嵌套association和collection。
他们的实现逻辑和常规resultMap中的处理方式完全相同，这里就不展开重复讲。和构造器不一样的是，鉴别器中包含了case分支和对应的resultMap的映射。

### 4.3 构建解析器后解析
ResultMapResolver.java
```
  public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
  }
```

MapperBuilderAssistant.java
```
  public ResultMap addResultMap(
      String id,
      Class<?> type,
      String extend,
      Discriminator discriminator,
      List<ResultMapping> resultMappings,
      Boolean autoMapping) {
      
    // 将id/extend填充为完整模式,也就是带命名空间前缀,true不需要和当前resultMap所在的namespace相同,比如extend和cache,否则只能是当前的namespace
    id = applyCurrentNamespace(id, false);
    extend = applyCurrentNamespace(extend, true);

    if (extend != null) {
      // 首先检查继承的resultMap是否已存在,如果不存在则标记为incomplete,会进行二次处理
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
      ResultMap resultMap = configuration.getResultMap(extend);
      List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
      // 剔除所继承的resultMap里已经在当前resultMap中的那个基本映射
      extendedResultMappings.removeAll(resultMappings);
      // Remove parent constructor if this resultMap declares a constructor.
      // 如果本resultMap已经包含了构造器,则剔除继承的resultMap里面的构造器
      boolean declaresConstructor = false;
      for (ResultMapping resultMapping : resultMappings) {
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          declaresConstructor = true;
          break;
        }
      }
      if (declaresConstructor) {
        Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
        while (extendedResultMappingsIter.hasNext()) {
          if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            extendedResultMappingsIter.remove();
          }
        }
      }
      /** 都处理完成之后,将继承的resultMap里面剩下那部分不重复的resultMap子元素添加到当前的resultMap中,
       * 所以这个addResultMap方法的用途在于启动时就创建了完整的resultMap，这样运行时就不需要去检查继承的映射和构造器,有利于性能提升。*/
      resultMappings.addAll(extendedResultMappings);
    }
    
    //4.3.1 构建resultMap
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    configuration.addResultMap(resultMap);
    return resultMap;
  }
```
mybatis源码分析到这里，应该来说上面部分代码不是特别显而易见,它主要是处理extends这个属性,将resultMap各子元素节点的去对象关系，
也就是去重、合并属性、构造器。处理完继承的逻辑之后，调用了ResultMap.Builder.build()进行最后的resultMap构建。
build应该来说是真正创建根resultMap对象的逻辑入口

### 4.3.1 构建resultMap
ResultMap.java
```
    public ResultMap build() {
      if (resultMap.id == null) {
        throw new IllegalArgumentException("ResultMaps must have an id");
      }
      resultMap.mappedColumns = new HashSet<>();
      resultMap.mappedProperties = new HashSet<>();
      resultMap.idResultMappings = new ArrayList<>();
      resultMap.constructorResultMappings = new ArrayList<>();
      resultMap.propertyResultMappings = new ArrayList<>();
      final List<String> constructorArgNames = new ArrayList<>();
      for (ResultMapping resultMapping : resultMap.resultMappings) {
        // 判断是否有嵌套查询
        resultMap.hasNestedQueries = resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
        // 判断是否嵌套了association或者collection
        resultMap.hasNestedResultMaps = resultMap.hasNestedResultMaps || (resultMapping.getNestedResultMapId() != null && resultMapping.getResultSet() == null);
        final String column = resultMapping.getColumn();
        // 获取column属性, 包括复合列
        if (column != null) {
          resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
        } else if (resultMapping.isCompositeResult()) {
          for (ResultMapping compositeResultMapping : resultMapping.getComposites()) {
            final String compositeColumn = compositeResultMapping.getColumn();
            if (compositeColumn != null) {
              resultMap.mappedColumns.add(compositeColumn.toUpperCase(Locale.ENGLISH));
            }
          }
        }
        // 所有映射的属性都被按顺序添加到resultMap.mappedProperties中,ID单独存储
        final String property = resultMapping.getProperty();
        if(property != null) {
          resultMap.mappedProperties.add(property);
        }
        
        // 所有映射的构造器被按顺序添加到resultMap.constructorResultMappings
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          resultMap.constructorResultMappings.add(resultMapping);
          if (resultMapping.getProperty() != null) {
            constructorArgNames.add(resultMapping.getProperty());
          }
        } else {
          resultMap.propertyResultMappings.add(resultMapping);
        }
        
        // 如果本元素具有ID标记, 则添加到ID映射列表resultMap.idResultMappings
        if (resultMapping.getFlags().contains(ResultFlag.ID)) {
          resultMap.idResultMappings.add(resultMapping);
        }
      }
      
      // 如果没有声明ID属性,就把所有属性都作为ID属性
      if (resultMap.idResultMappings.isEmpty()) {
        resultMap.idResultMappings.addAll(resultMap.resultMappings);
      }
      
      // 根据声明的构造器参数名和类型,反射声明的类,检查其中是否包含对应参数名和类型的构造器,如果不存在匹配的构造器,就抛出运行时异常
      if (!constructorArgNames.isEmpty()) {
        final List<String> actualArgNames = argNamesOfMatchingConstructor(constructorArgNames);
        if (actualArgNames == null) {
          throw new BuilderException("Error in result map '" + resultMap.id
              + "'. Failed to find a constructor in '"
              + resultMap.getType().getName() + "' by arg names " + constructorArgNames
              + ". There might be more info in debug log.");
        }
        //构造器参数排序
        Collections.sort(resultMap.constructorResultMappings, (o1, o2) -> {
          int paramIdx1 = actualArgNames.indexOf(o1.getProperty());
          int paramIdx2 = actualArgNames.indexOf(o2.getProperty());
          return paramIdx1 - paramIdx2;
        });
      }
      // lock down collections
      // 为了避免用于无意或者有意事后修改resultMap的内部结构, 克隆一个不可修改的集合提供给用户
      resultMap.resultMappings = Collections.unmodifiableList(resultMap.resultMappings);
      resultMap.idResultMappings = Collections.unmodifiableList(resultMap.idResultMappings);
      resultMap.constructorResultMappings = Collections.unmodifiableList(resultMap.constructorResultMappings);
      resultMap.propertyResultMappings = Collections.unmodifiableList(resultMap.propertyResultMappings);
      resultMap.mappedColumns = Collections.unmodifiableSet(resultMap.mappedColumns);
      return resultMap;
    }
```

### 5. 解析sql节点
XMLMapperBuilder.java
sql元素可以被用来定义可重用的SQL代码段，包含在其他语句中。比如，他常被用来定义重用的列
```
  <sql id=”userColumns”> id,username,password </sql>

  private void sqlElement(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
  }
  
  private void sqlElement(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      String databaseId = context.getStringAttribute("databaseId");
      String id = context.getStringAttribute("id");
      id = builderAssistant.applyCurrentNamespace(id, false);
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
        sqlFragments.put(id, context);
      }
    }
  }
```
sql元素本身的处理很简单，只是简单的过滤出databaseId和当前加载的配置文件相同的语句以备以后再解析crud遇到时进行引用。
不进行解析是因为首先能够作为sql元素子元素的所有节点都可以作为crud的子元素，而且sql元素不会在运行时单独使用，所以也没有必要专门解析一番。