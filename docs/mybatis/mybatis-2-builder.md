## SqlSessionFactoryBuilder

### builder流程

SqlSessionFactoryBuilder.java
```
  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      // 1. 构建xml配置解析器
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      // 2. 解析xml文件然后构建SqlSessionFactory
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
```

### 构建XMLConfigBuilder
XMLConfigBuilder.java
```
  public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    // 构建转换器和实体解析器
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
  }
  
  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
  
  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
```

BaseBuilder.java
```
  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```


### 构建转换器和实体解析器
XMLMapperEntityResolver.java
离线mybatis的DTD解析器，使用本地文件提高获取dtd获取。
XPathParser.java
```
  public XPathParser(Reader reader, boolean validation, Properties variables, EntityResolver entityResolver) {
    commonConstructor(validation, variables, entityResolver);
    this.document = createDocument(new InputSource(reader));
  }

  // commonConstructor并没有做什么
  private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
  }
  
  //主要是根据mybatis自身需要创建一个文档解析器，然后调用parse将输入input source解析为DOM XML文档并返回
  private Document createDocument(InputSource inputSource) {
    // important: this must only be called AFTER common constructor
    try {
      DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
      factory.setValidating(validation);

      // 设置由本工厂创建的解析器是否支持XML命名空间
      factory.setNamespaceAware(false);
      factory.setIgnoringComments(true);
      factory.setIgnoringElementContentWhitespace(false);
      // 设置是否将CDATA节点转换为Text节点
      factory.setCoalescing(false);
      // 设置是否展开实体引用节点，这里应该是sql片段引用的关键
      factory.setExpandEntityReferences(true);

      DocumentBuilder builder = factory.newDocumentBuilder();
      // 设置解析mybatis xml文档节点的解析器,也就是上面的XMLMapperEntityResolver
      builder.setEntityResolver(entityResolver);
      builder.setErrorHandler(new ErrorHandler() {
        @Override
        public void error(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void fatalError(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void warning(SAXParseException exception) throws SAXException {
        }
      });
      return builder.parse(inputSource);
    } catch (Exception e) {
      throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }
  }
```

### 解析配置
XMLConfigBuilder.java
```
  public Configuration parse() {
    // 判断有没有解析过配置文件，只有没有解析过才允许解析
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 解析根节点configuration并从根节点开始解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
  
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      // 1. 属性解析propertiesElement
      propertiesElement(root.evalNode("properties"));
      // 2. 加载settings节点settingsAsProperties
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      // 3. 加载自定义VFS loadCustomVfs
      loadCustomVfs(settings);
      // 4. 加载自定义日志实现
      loadCustomLogImpl(settings);
      // 5. 解析类型别名typeAliasesElement
      typeAliasesElement(root.evalNode("typeAliases"));
      // 6. 加载插件pluginElement
      pluginElement(root.evalNode("plugins"));
      // 7. 加载对象工厂objectFactoryElement
      objectFactoryElement(root.evalNode("objectFactory"));
      // 8. 创建对象包装器工厂objectWrapperFactoryElement
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 9. 加载反射工厂
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      // 10. 设置默认配置 
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 11. 加载环境配置environmentsElement
      environmentsElement(root.evalNode("environments"));
      // 12. 数据库厂商标识加载databaseIdProviderElement
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 13. 加载类型处理器typeHandlerElement
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 14. 加载mapper文件mapperElement
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```


Configuration.java 由于是配置类代码请参考源码
mybatis中所有环境配置、resultMap集合、sql语句集合、插件列表、缓存、加载的xml列表、类型别名、类型处理器等全部都维护在Configuration中。
Configuration中包含了一个内部静态类StrictMap，它继承于HashMap，对HashMap的装饰在于增加了put时防重复的处理，get时取不到值时候的异常处理，
这样核心应用层就不需要额外关心各种对象异常处理,简化应用层逻辑。

从Configuration构造器和protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
可以看出，所有我们在mybatis-config和mapper文件中使用的类似int/string/JDBC/POOLED等字面常量最终解析为具体的java类型
都是在typeAliasRegistry构造器和Configuration构造器执行期间初始化的


### 1.属性解析propertiesElement
XMLConfigBuilder.java
```
  private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      Properties defaults = context.getChildrenAsProperties();
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      configuration.setVariables(defaults);
    }
  }
```

### 2. 加载settings节点settingsAsProperties
XMLConfigBuilder.java
```
  private Properties settingsAsProperties(XNode context) {
    if (context == null) {
      return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
      if (!metaConfig.hasSetter(String.valueOf(key))) {
        throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
      }
    }
    return props;
  }
```

### 3. 加载自定义VFS loadCustomVfs
VFS主要用来加载容器内的各种资源，比如jar或者class文件。mybatis提供了2个实现JBoss6VFS和DefaultVFS，
并提供了用户扩展点，用于自定义VFS实现，加载顺序是自定义VFS实现 > 默认VFS实现 取第一个加载成功的。
```
  private void loadCustomVfs(Properties props) throws ClassNotFoundException {
    String value = props.getProperty("vfsImpl");
    if (value != null) {
      String[] clazzes = value.split(",");
      for (String clazz : clazzes) {
        if (!clazz.isEmpty()) {
          @SuppressWarnings("unchecked")
          Class<? extends VFS> vfsImpl = (Class<? extends VFS>)Resources.classForName(clazz);
          configuration.setVfsImpl(vfsImpl);
        }
      }
    }
  }
```

### 4. 加载自定义日志实现
XMLConfigBuilder.java
```
  private void loadCustomLogImpl(Properties props) {
    Class<? extends Log> logImpl = resolveClass(props.getProperty("logImpl"));
    configuration.setLogImpl(logImpl);
  }
```

### 5. 解析类型别名typeAliasesElement
```
  private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```
mybatis主要提供两种类型的别名设置，具体类的别名以及包的别名设置。
类型别名是为Java类型设置一个短的名字，存在的意义仅在于用来减少类完全限定名的冗余。

### 6. 加载插件pluginElement
```
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```
插件在具体实现的时候，采用的是拦截器模式，要注册为mybatis插件，必须实现org.apache.ibatis.plugin.Interceptor接口

### 7. 加载对象工厂objectFactoryElement
```
  private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties properties = context.getChildrenAsProperties();
      ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
      factory.setProperties(properties);
      configuration.setObjectFactory(factory);
    }
  }
```
MyBatis每次创建结果对象的新实例时，它都会使用一个对象工厂（ObjectFactory）实例来完成。 
默认的对象工厂DefaultObjectFactory做的仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。

### 8. 创建对象包装器工厂objectWrapperFactoryElement
```
  private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
      configuration.setObjectWrapperFactory(factory);
    }
  }
```
对象包装器工厂主要用来包装返回result对象，比如说可以用来设置某些敏感字段脱敏或者加密等。
默认对象包装器工厂是DefaultObjectWrapperFactory，也就是不使用包装器工厂。

### 9. 加载反射工厂
```
  private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
       String type = context.getStringAttribute("type");
       ReflectorFactory factory = (ReflectorFactory) resolveClass(type).newInstance();
       configuration.setReflectorFactory(factory);
    }
  }
```

### 10. 设置默认值
```
  private void settingsElement(Properties props) {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
    configuration.setLogPrefix(props.getProperty("logPrefix"));
    configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
  }
```

### 11. 加载环境配置environmentsElement

### 12. 数据库厂商标识加载databaseIdProviderElement

### 13. 加载类型处理器typeHandlerElement

### 14. 加载mapper文件mapperElement
