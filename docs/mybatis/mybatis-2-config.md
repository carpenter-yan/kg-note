#SqlSessionFactoryBuilder

### 0.builder流程
```
static {
    try {
        // 将mybatis配置文件转换为文件输入流reader
        reader = Resources.getResourceAsReader("mybatis/config.xml");
        //private static SqlSessionFactory ssf;
        ssf = new SqlSessionFactoryBuilder().build(reader);
    }
    catch (IOException e) {
        e.printStackTrace();
    }
}
```

SqlSessionFactoryBuilder.java
```
  public SqlSessionFactory build(Reader reader) {
    return build(reader, null, null);
  }

  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      // 1. 构建xml配置转化器
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      // 2. parse()将document转换为Configuration
      // 3. build()通过Configuration构建SqlSessionFactory
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

### 1.构建XMLConfigBuilder
XMLConfigBuilder.java
关键是构建XMLMapperEntityResolver和XPathParser
XMLMapperEntityResolver.java 离线mybatis的DTD解析器，使用本地文件提高获取dtd获取。
```
  public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    // 构建转换器和实体解析器
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
  }
```

### 构建转换器和实体解析器
XPathParser.java
```
  public XPathParser(Reader reader, boolean validation, Properties variables, EntityResolver entityResolver) {
    commonConstructor(validation, variables, entityResolver);
    this.document = createDocument(new InputSource(reader));
  }

  // 1.commonConstructor并没有做什么
  private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
    this.validation = validation;
    this.entityResolver = entityResolver;
    this.variables = variables;
    XPathFactory factory = XPathFactory.newInstance();
    this.xpath = factory.newXPath();
  }
  
  // 2.主要是根据mybatis自身需要创建一个文档解析器，然后调用parse将输入input source解析为DOM XML文档并返回
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
      // 设置解析mybatis.xml文档节点的解析器,也就是上面的XMLMapperEntityResolver
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

DocumentBuilderImpl.java
```
    public Document parse(InputSource is) throws SAXException, IOException {
        if (is == null) {
            throw new IllegalArgumentException(
                DOMMessageFormatter.formatMessage(DOMMessageFormatter.DOM_DOMAIN,
                "jaxp-null-input-source", null));
        }
        if (fSchemaValidator != null) {
            if (fSchemaValidationManager != null) {
                fSchemaValidationManager.reset();
                fUnparsedEntityHandler.reset();
            }
            resetSchemaValidator();
        }
        domParser.parse(is);
        Document doc = domParser.getDocument();
        domParser.dropDocumentReferences();
        return doc;
    }
```
使用domParser解析得到Document。

### 2.解析xml文件并转换为Configuration
XMLConfigBuilder创建完成之后，SqlSessionFactoryBuild调用parser.parse()创建Configuration
XMLConfigBuilder.java
```
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //调用evalNode解析节点信息为XNode实例
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```

1. configuration节点为根节点
2. 其下包括11个子元素，如代码所示
   mybatis中所有环境配置、resultMap集合、sql语句集合、插件列表、缓存、加载的xml列表、类型别名、类型处理器等全部都维护在Configuration中。
   Configuration中包含了一个内部静态类StrictMap，它继承于HashMap，对HashMap的装饰在于增加了put时防重复的处理，get时取不到值时候的异常处理，
   这样核心应用层就不需要额外关心各种对象异常处理,简化应用层逻辑。
XMLConfigBuilder.java
```
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

Configuration.java是xml配置文件里面的所有配置的实现表示(请参考源码)。
```
public class Configuration {

  protected Environment environment;
  protected boolean safeRowBoundsEnabled;
  protected boolean safeResultHandlerEnabled = true;
  // 是否开启自动驼峰命名规则（camel case）映射，即从经典数据库列名 A_COLUMN 到经典 Java 属性名 aColumn 的类似映射。默认false
  protected boolean mapUnderscoreToCamelCase;
  // 当开启时，任何方法的调用都会加载该对象的所有属性。否则，每个属性会按需加载。默认值false (true in ≤3.4.1)
  protected boolean aggressiveLazyLoading;
  // 是否允许单一语句返回多结果集（需要兼容驱动）。
  protected boolean multipleResultSetsEnabled = true;
  // 允许 JDBC 支持自动生成主键，需要驱动兼容。这就是insert时获取mysql自增主键/oracle sequence的开关。注：一般来说,这是希望的结果,应该默认值为true比较合适。
  protected boolean useGeneratedKeys;
  // 使用列标签代替列名,一般来说,这是希望的结果
  protected boolean useColumnLabel = true;
  // 是否启用缓存
  protected boolean cacheEnabled = true;
  protected boolean callSettersOnNulls;
  protected boolean useActualParamName = true;
  protected boolean returnInstanceForEmptyRow;
  // 指定 MyBatis 增加到日志名称的前缀。
  protected String logPrefix;
  // 指定 MyBatis 所用日志的具体实现，未指定时将自动查找。一般建议指定为slf4j或log4j2
  protected Class <? extends Log> logImpl;
  // 指定VFS的实现, VFS是mybatis提供的用于访问AS内资源的一个简便接口
  protected Class <? extends VFS> vfsImpl;
  // MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 
  // 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 
  // 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  // 当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。
  protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
  // 指定对象的哪个方法触发一次延迟加载。
  protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
  // 设置超时时间，它决定驱动等待数据库响应的秒数。默认不超时
  protected Integer defaultStatementTimeout;
  // 为驱动的结果集设置默认获取数量。
  protected Integer defaultFetchSize;
  // SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新。
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  // 指定 MyBatis应如何自动映射列到字段或属性。NONE表示取消自动映射；PARTIAL只会自动映射没有定义嵌套结果集映射的结果集。FULL会自动映射任意复杂的结果集（无论是否嵌套）。
  protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
  protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;
  // settings下的properties属性
  protected Properties variables = new Properties();
  // 默认的反射器工厂,用于操作属性、构造器方便
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  // 对象工厂, 所有的类resultMap类都需要依赖于对象工厂来实例化
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  // 对象包装器工厂,主要用来在创建非原生对象,比如增加了某些监控或者特殊属性的代理类
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
  // 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。特定关联关系中可通过设置fetchType属性来覆盖该项的开关状态。
  protected boolean lazyLoadingEnabled = false;
  // 指定 Mybatis 创建具有延迟加载能力的对象所用到的代理工具。MyBatis 3.3+使用JAVASSIST
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL
  // MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 databaseId 属性
  protected String databaseId;
  /**
   * Configuration factory class.
   * Used to create Configuration for loading deserialized unread properties.
   *
   * @see <a href='https://code.google.com/p/mybatis/issues/detail?id=300'>Issue 300 (google code)</a>
   */
  // 指定一个提供Configuration实例的类. 这个被返回的Configuration实例是用来加载被反序列化对象的懒加载属性值. 
  // 这个类必须包含一个签名方法static Configuration getConfiguration(). (从 3.2.3 版本开始)
  protected Class<?> configurationFactory;

  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  // mybatis插件列表
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
  // 类型注册器,用于在执行sql语句的出入参映射以及mybatis-config文件里的各种配置比如<transactionManager type="JDBC"/><dataSource type="POOLED">时使用简写
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");

  protected final Set<String> loadedResources = new HashSet<>();
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");

  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();
  /*
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
}

  public Configuration() {
    // 内置别名注册
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);

    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class);

    typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);

    typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
    typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

    languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
    languageRegistry.register(RawLanguageDriver.class);
  }
```
Configuration中包含了一个内部静态类StrictMap，它继承于HashMap，对HashMap的装饰在于增加了put时防重复的处理，get时取不到值时候的异常处理，
这样核心应用层就不需要额外关心各种对象异常处理,简化应用层逻辑。

