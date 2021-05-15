## mapper加载和初始化
MapperBuilderAssistant初始化完成之后，就调用build.parse()进行具体的mapper接口文件加载与解析

MapperAnnotationBuilder.java
```
  public void parse() {
    String resource = type.toString();
    //首先根据mapper接口的字符串表示判断是否已经加载,避免重复加载,正常情况下应该都没有加载
    if (!configuration.isResourceLoaded(resource)) {
      // 1、首先加载mapper接口对应的xml文件并解析
      loadXmlResource();
      configuration.addLoadedResource(resource);
      // 每个mapper文件自成一个namespace，通常自动匹配就是这么来的，约定俗成代替人工设置最简化常见的开发(使用接口服务名作为namespace)
      assistant.setCurrentNamespace(type.getName());
      // 2、解析缓存注解；
      parseCache();
      // 3、解析缓存参照注解。
      parseCacheRef();
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          // issue #237
          // 4、解析非桥接方法。
          /** 桥接方法是 JDK 1.5引入泛型后，为了使Java的泛型方法生成的字节码和1.5 版本前的字节码相兼容，由编译器自动生成的方法。
            * 一个子类在继承（或实现）一个父类（或接口）的泛型方法时，在子类中明确指定了泛型类型，那么在编译时编译器会自动生成桥接方法。
            * 只要在实现mybatis mapper接口的时，没有继承根Mapper或者继承了根Mapper但是没有写死泛型类型的时候，是不会成为桥接方法的。*/
          if (!method.isBridge()) {
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    // 5、二次解析pending的方法。
    parsePendingMethods();
  }
/****************************************************************************************************/

  void parseStatement(Method method) {
    /** 获取参数类型,如果有多个参数,这种情况下就返回org.apache.ibatis.binding.MapperMethod.ParamMap.class，
      * ParamMap是一个继承于HashMap的类，否则返回实际类型*/
    Class<?> parameterTypeClass = getParameterType(method);
    // 获取语言驱动器
    LanguageDriver languageDriver = getLanguageDriver(method);
    /** 获取方法的SqlSource对象,只有指定了@Select/@Insert/@Update/@Delete或者对应的Provider的方法才会被当作mapper,
      * 否则只是和mapper文件中对应语句的一个运行时占位符*/
    SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
    if (sqlSource != null) {
      // 获取方法的属性设置，对应<select>中的各种属性
      Options options = method.getAnnotation(Options.class);
      final String mappedStatementId = type.getName() + "." + method.getName();
      Integer fetchSize = null;
      Integer timeout = null;
      StatementType statementType = StatementType.PREPARED;
      ResultSetType resultSetType = null;
      // 获取语句的CRUD类型
      SqlCommandType sqlCommandType = getSqlCommandType(method);
      boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
      boolean flushCache = !isSelect;
      boolean useCache = isSelect;

      KeyGenerator keyGenerator;
      String keyProperty = null;
      String keyColumn = null;
      // 只有INSERT/UPDATE才解析SelectKey选项
      if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
        // first check for SelectKey annotation - that overrides everything else
        SelectKey selectKey = method.getAnnotation(SelectKey.class);
        if (selectKey != null) {
          keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
          keyProperty = selectKey.keyProperty();
        } else if (options == null) {
          keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        } else {
          keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
          keyProperty = options.keyProperty();
          keyColumn = options.keyColumn();
        }
      } else {
        keyGenerator = NoKeyGenerator.INSTANCE;
      }

      if (options != null) {
        if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
          flushCache = true;
        } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
          flushCache = false;
        }
        useCache = options.useCache();
        fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
        timeout = options.timeout() > -1 ? options.timeout() : null;
        statementType = options.statementType();
        resultSetType = options.resultSetType();
      }

      String resultMapId = null;
      // 解析@ResultMap注解,如果有@ResultMap注解,就是用它，否则才解析@Results
      // @ResultMap注解用于给@Select和@SelectProvider注解提供在xml配置的<resultMap>,
      如果一个方法上同时出现@Results或者@ConstructorArgs等和结果映射有关的注解,那么@ResultMap会覆盖后面两者的注解
      ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
      if (resultMapAnnotation != null) {
        String[] resultMaps = resultMapAnnotation.value();
        StringBuilder sb = new StringBuilder();
        for (String resultMap : resultMaps) {
          if (sb.length() > 0) {
            sb.append(",");
          }
          sb.append(resultMap);
        }
        resultMapId = sb.toString();
      } else if (isSelect) {
        resultMapId = parseResultMap(method);
      }

      assistant.addMappedStatement(
          mappedStatementId,
          sqlSource,
          statementType,
          sqlCommandType,
          fetchSize,
          timeout,
          // ParameterMapID
          null,
          parameterTypeClass,
          resultMapId,
          getReturnType(method),
          resultSetType,
          flushCache,
          useCache,
          // TODO gcode issue #577
          false,
          keyGenerator,
          keyProperty,
          keyColumn,
          // DatabaseID
          null,
          languageDriver,
          // ResultSets
          options != null ? nullOrEmpty(options.resultSets()) : null);
    }
  }
  
  /*****************************************************************************/
  private String parseResultMap(Method method) {
    // 获取方法的返回类型
    Class<?> returnType = getReturnType(method);
    // 获取构造器
    ConstructorArgs args = method.getAnnotation(ConstructorArgs.class);
    // 获取@Results注解,也就是注解形式的结果映射
    Results results = method.getAnnotation(Results.class);
    // 获取鉴别器
    TypeDiscriminator typeDiscriminator = method.getAnnotation(TypeDiscriminator.class);
    // 产生resultMapId
    String resultMapId = generateResultMapName(method);
    applyResultMap(resultMapId, returnType, argsIf(args), resultsIf(results), typeDiscriminator);
    return resultMapId;
  }
  
  /******************************************************************************/
  // 如果有resultMap设置了Id，就直接返回类名.resultMapId. 否则返回类名.方法名.以-分隔拼接的方法参数
  private String generateResultMapName(Method method) {
    Results results = method.getAnnotation(Results.class);
    if (results != null && !results.id().isEmpty()) {
      return type.getName() + "." + results.id();
    }
    StringBuilder suffix = new StringBuilder();
    for (Class<?> c : method.getParameterTypes()) {
      suffix.append("-");
      suffix.append(c.getSimpleName());
    }
    if (suffix.length() < 1) {
      suffix.append("-void");
    }
    return type.getName() + "." + method.getName() + suffix;
  }
  
  /******************************************************************************/
  private void applyResultMap(String resultMapId, Class<?> returnType, Arg[] args, Result[] results, TypeDiscriminator discriminator) {
    List<ResultMapping> resultMappings = new ArrayList<>();
    applyConstructorArgs(args, returnType, resultMappings);
    applyResults(results, returnType, resultMappings);
    Discriminator disc = applyDiscriminator(resultMapId, returnType, discriminator);
    // TODO add AutoMappingBehaviour
    assistant.addResultMap(resultMapId, returnType, null, disc, resultMappings, null);
    createDiscriminatorResultMaps(resultMapId, returnType, discriminator);
  }
  
  /*******************************************************************************/
  private Discriminator applyDiscriminator(String resultMapId, Class<?> resultType, TypeDiscriminator discriminator) {
    if (discriminator != null) {
      String column = discriminator.column();
      Class<?> javaType = discriminator.javaType() == void.class ? String.class : discriminator.javaType();
      JdbcType jdbcType = discriminator.jdbcType() == JdbcType.UNDEFINED ? null : discriminator.jdbcType();
      @SuppressWarnings("unchecked")
      Class<? extends TypeHandler<?>> typeHandler = (Class<? extends TypeHandler<?>>)
              (discriminator.typeHandler() == UnknownTypeHandler.class ? null : discriminator.typeHandler());
      Case[] cases = discriminator.cases();
      Map<String, String> discriminatorMap = new HashMap<>();
      for (Case c : cases) {
        String value = c.value();
        String caseResultMapId = resultMapId + "-" + value;
        discriminatorMap.put(value, caseResultMapId);
      }
      return assistant.buildDiscriminator(resultType, column, javaType, jdbcType, typeHandler, discriminatorMap);
    }
    return null;
  }
  
  /*****************************************************************************/
  private void createDiscriminatorResultMaps(String resultMapId, Class<?> resultType, TypeDiscriminator discriminator) {
    if (discriminator != null) {
      for (Case c : discriminator.cases()) {
        String caseResultMapId = resultMapId + "-" + c.value();
        List<ResultMapping> resultMappings = new ArrayList<>();
        // issue #136
        applyConstructorArgs(c.constructArgs(), resultType, resultMappings);
        applyResults(c.results(), resultType, resultMappings);
        // TODO add AutoMappingBehaviour
        assistant.addResultMap(caseResultMapId, c.type(), resultMapId, null, resultMappings, null);
      }
    }
  }
```