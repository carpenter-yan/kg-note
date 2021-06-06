### mybatis结果集处理

DefaultResultSetHandler.java
```
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    // 返回jdbc ResultSet的包装形式,主要是将java.sql.ResultSetMetaData做了Facade模式,便于使用
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    // 绝大部分情况下一个查询只有一个ResultMap, 除非多结果集查询
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    // 至少执行一次
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      /** 循环直到处理完所有的结果集，一般情况下，一个execute只会返回一个结果集,
       * 除非语句比如存储过程返回多个resultSet*/
      rsw = getNextResultSet(stmt);
      // 清空嵌套结果集
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    // 处理关联或集合
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      // 如果一个映射语句的resultSet数量比jdbc resultset多的话，多的部分就是嵌套结果集
      while (rsw != null && resultSetCount < resultSets.length) {
        // nextResultMaps初始化的时候为空,它是在处理主查询每行记录的时候写进去的，所以此时就可以得到主记录是哪个属性
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          //1. 得到子结果集的resultMap之后，就可以进行填充了
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
  
  /** 因为调用handleResultSet的只有handleResultSets，按说其中第一个调用永远不会出现parentMapping==null的情况，
   * 只有第二个调用才会出现这种情况，而且应该是连续的，因为第二个调用就是为了处理嵌套resultMap。
   * 所以在handleRowValues处理resultMap的时候，一定是主的先处理，嵌套的后处理，这样整个逻辑就比较清晰了。
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      // 处理嵌套/关联的resultMap(collection或association)
      if (parentMapping != null) {
        //1.1 处理非主记录 resultHandler传递null，RowBounds传递默认值，parentMapping不为空
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          //1.2 处理主记录，resultHander不为空,rowBounds不使用默认值,parentMapping传递null
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }
  
  // 1.1 处理每一行记录,不管是主查询记录还是关联嵌套的查询结果集，他们的入参上通过resultHandler,rowBounds,parentMapping区分
  public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      //1.1.1 含有嵌套resultMap的列处理，通常是collection或者association
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
  
  // 1.2 处理主记录
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    // 设置从指定行开始,这是mybatis进行内部分页,不是依赖服务器端的分页实现
    skipRows(resultSet, rowBounds);
    // 确保没有超过mybatis逻辑分页限制,同时结果集中还有记录没有fetch
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      // 解析鉴别器，得到嵌套的最深的鉴别器对应的ResultMap，如果没有鉴别器，就返回最顶层的ResultMap
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      //1.2.1 这个时候resultMap是非常干净的,没有嵌套任何其他东西了，但是这也是最关键的地方，将ResultSet记录转换为业务层配置的对象类型或者Map类型
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      //1.2.2
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }
  
  //1.2.1
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    /** 有三个createResultObject重载，这三个重载完成的功能从最里面到最外面分别是：
     * 1、使用构造器创建目标对象类型;
     *   先判断是否有目标对象类型的处理器，有的话，直接调用类型处理器(这里为什么一定是原生类型)创建目标对象
     *   如果没有，判断是否有构造器，有的话，使用指定的构造器创建目标对象（构造器里面如果嵌套了查询或者ResultMap，则进行处理）
     *   如果结果类型是接口或者具有默认构造器，则使用ObjectFactory创建默认目标对象
     *   最后判断是否可以应用自动映射，默认是对非嵌套查询，只要没有明确设置AutoMappingBehavior.NONE就可以，对于嵌套查询，AutoMappingBehavior.FULL就可以。
     *   自动映射的逻辑是先找目标对象上具有@AutomapConstructor注解的构造器，然后根据ResultSet返回的字段清单找匹配的构造器，如果找不到，就报错
     * 2、如果此时创建的对象不为空，且不需要应用结果对象处理器，判断有没有延迟加载且具有嵌套查询的属性，
     *   如果有的话，则为对象创建一个代理，额外存储后面fetch的时候进行延迟加载所需的信息。返回对象。
     * 3、如果此时创建的对象不为空，且不需要应用结果对象处理器，如果对象需要自动映射，则先进行自动映射
         （创建自动映射列表的过程为：先找到在ResultSet、不在ResultMap中的列，如果在目标对象上可以找到属性且可以类型可以处理，则标记为可以自动映射；
         然后进行自动映射处理，如果遇到无法处理的属性，则根据autoMappingUnknownColumnBehavior进行处理，默认忽略），
         其次进行属性映射处理*/
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      // 处理属性映射,这里会识别出哪些属性需要nestQuery，哪些是nest ResultMap
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }
  
  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        column = null;
      }
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        final String property = propertyMapping.getProperty();
        if (property == null) {
          continue;
        } else if (value == DEFERRED) {
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }
  
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      // 1.2.1.1 处理嵌套query类型的属性
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      /** 处理resultMap类型的属性,主要是：
       * 1、维护记录cacheKey和父属性/记录对Map的关联关系，便于在处理嵌套ResultMap时很快可以找到所有需要处理嵌套结果集的父属性；
       * 2、维护父属性和对应resultSet的关联关系。这两者都是为了在处理嵌套结果集是方便*/
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
  
  // 1.2.2
  private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
      //1.2.1.1 如果不是主记录,则链接到主记录
      linkToParents(rs, parentMapping, rowValue);
    } else {
      //1.2.1.2 否则让ResultHander(默认是DefaultResultHander处理,直接添加到List<Object>中)
      callResultHandler(resultHandler, resultContext, rowValue);
    }
  }
  
  //1.2.2 否则让ResultHander处理
  private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    resultContext.nextResultObject(rowValue);
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
  }
  
  //1.2.1 如果不是主记录,则链接到主记录
  private void linkToParents(ResultSet rs, ResultMapping parentMapping, Object rowValue) throws SQLException {
    CacheKey parentKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getForeignColumn());
    // 判断当前的主记录是否有待关联的子记录，是通过pendingRelations Map进行维护的，pendingRelations是在addPendingChildRelation中添加主记录的
    List<PendingRelation> parents = pendingRelations.get(parentKey);
    if (parents != null) {
      for (PendingRelation parent : parents) {
        if (parent != null && rowValue != null) {
          linkObjects(parent.metaObject, parent.propertyMapping, rowValue);
        }
      }
    }
  }
  
  /**  集合与非集合的处理逻辑对外封装在一起,这样便于用户使用, 内部通过判断resultMapping中的类型确定是否为集合类型。
   * 无论是否为集合类型，最后都添加到parent的metaObject所封装的原始Object对应的属性上*/
  private void linkObjects(MetaObject metaObject, ResultMapping resultMapping, Object rowValue) {
    final Object collectionProperty = instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject);
    if (collectionProperty != null) {
      final MetaObject targetMetaObject = configuration.newMetaObject(collectionProperty);
      targetMetaObject.add(rowValue);
    } else {
      metaObject.setValue(resultMapping.getProperty(), rowValue);
    }
  }
  
  private Object instantiateCollectionPropertyIfAppropriate(ResultMapping resultMapping, MetaObject metaObject) {
    final String propertyName = resultMapping.getProperty();
    Object propertyValue = metaObject.getValue(propertyName);
    if (propertyValue == null) {
      Class<?> type = resultMapping.getJavaType();
      if (type == null) {
        type = metaObject.getSetterType(propertyName);
      }
      try {
        if (objectFactory.isCollection(type)) {
          propertyValue = objectFactory.create(type);
          metaObject.setValue(propertyName, propertyValue);
          return propertyValue;
        }
      } catch (Exception e) {
        throw new ExecutorException("Error instantiating collection property for result '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
      }
    } else if (objectFactory.isCollection(propertyValue.getClass())) {
      return propertyValue;
    }
    return null;
  }
```

### 1.1.1 嵌套resultMap的处理
DefaultResultSetHandler.java
```
  private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    // mybatis 分页处理
    skipRows(resultSet, rowBounds);
    // 前一次处理的记录,只有在映射语句的结果集无序的情况下有意义
    Object rowValue = previousRowValue;
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      //同主记录,先解析到鉴别器的最底层的ResultMap
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      // 创建当前处理记录的rowKey，规则见下文
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
      // 根据rowKey获取嵌套结果对象map中对应的值，因为在处理主记录时存储进去了，具体见上面addPending的流程图，所以partialObject一般不会为空
      Object partialObject = nestedResultObjects.get(rowKey);
      // issue #577 && #542
      // 根据映射语句的结果集是否有序走不同的逻辑
      if (mappedStatement.isResultOrdered()) {
        // 对于有序结果集的映射语句，如果嵌套结果对象map中不包含本记录，则清空嵌套结果对象，因为此时嵌套结果对象之前的记录已经没有意义了
        if (partialObject == null && rowValue != null) {
          nestedResultObjects.clear();
          // 添加记录到resultHandler的list属性中 或 如果是非主记录,添加到主记录对应属性的list或者object中
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      } else {
        // 正常逻辑走这里
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        if (partialObject == null) {
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
      }
    }
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
      previousRowValue = null;
    } else if (rowValue != null) {
      previousRowValue = rowValue;
    }
  }
  
  /** 为嵌套resultMap创建rowValue，和非嵌套记录的接口分开
   *  getRowValue和applyNestedResultMappings存在递归调用,直到处理到不包含任何嵌套结果集的最后一层resultMap为止*/
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    final String resultMapId = resultMap.getId();
    Object rowValue = partialObject;
    if (rowValue != null) {
      // 此时rowValue不应该空
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      putAncestor(rowValue, resultMapId);
      applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
      ancestorObjects.remove(resultMapId);
    } else {
      final ResultLoaderMap lazyLoader = new ResultLoaderMap();
      rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
      if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        // 判断是否至少找到了一个不为null的属性值
        boolean foundValues = this.useConstructorMappings;
        if (shouldApplyAutomaticMappings(resultMap, true)) {
          foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        putAncestor(rowValue, resultMapId);
        foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
        ancestorObjects.remove(resultMapId);
        foundValues = lazyLoader.size() > 0 || foundValues;
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
      }
      if (combinedKey != CacheKey.NULL_CACHE_KEY) {
        nestedResultObjects.put(combinedKey, rowValue);
      }
    }
    return rowValue;
  }
  
  private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
      final String nestedResultMapId = resultMapping.getNestedResultMapId();
      if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
        // 仅仅处理嵌套resultMap的属性
        try {
          final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
          // 得到嵌套resultMap的实际定义
          final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
          if (resultMapping.getColumnPrefix() == null) {
            // try to fill circular reference only when columnPrefix
            // is not specified for the nested result map (issue #215)
            // 不管循环嵌套resultMap的情况
            Object ancestorObject = ancestorObjects.get(nestedResultMapId);
            if (ancestorObject != null) {
              if (newObject) {
                linkObjects(metaObject, resultMapping, ancestorObject); // issue #385
              }
              continue;
            }
          }
          final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
          final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
          Object rowValue = nestedResultObjects.get(combinedKey);
          // 第一次一定是null
          boolean knownValue = rowValue != null;
          // 为嵌套resultMap属性创建对象或者集合
          instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory
          // 至少有一个字段不为null
          if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw)) {
            // 为嵌套resultMap创建记录
            rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
            if (rowValue != null && !knownValue) {
              //获取到记录,就绑定到主记录
              linkObjects(metaObject, resultMapping, rowValue);
              foundValues = true;
            }
          }
        } catch (SQLException e) {
          throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
        }
      }
    }
    return foundValues;
  }
```
### 1.2.1.1 嵌套查询的属性处理
DefaultResultSetHandler.java
```
  private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    // 获取queryId
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    final String property = propertyMapping.getProperty();
    // 根据嵌套queryId获取映射语句对象
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
      final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
      final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
      final Class<?> targetType = propertyMapping.getJavaType();
      if (executor.isCached(nestedQuery, key)) {
        executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
        value = DEFERRED;
      } else {
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        if (propertyMapping.isLazy()) {
          lazyLoader.addLoader(property, metaResultObject, resultLoader);
          value = DEFERRED;
        } else {
          value = resultLoader.loadResult();
        }
      }
    }
    return value;
  }
  
  private Object prepareParameterForNestedQuery(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    if (resultMapping.isCompositeResult()) {
      return prepareCompositeKeyParameter(rs, resultMapping, parameterType, columnPrefix);
    } else {
      return prepareSimpleKeyParameter(rs, resultMapping, parameterType, columnPrefix);
    }
  }
  
  private Object prepareSimpleKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    final TypeHandler<?> typeHandler;
    if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
      typeHandler = typeHandlerRegistry.getTypeHandler(parameterType);
    } else {
      typeHandler = typeHandlerRegistry.getUnknownTypeHandler();
    }
    return typeHandler.getResult(rs, prependPrefix(resultMapping.getColumn(), columnPrefix));
  }
  
  private Object prepareCompositeKeyParameter(ResultSet rs, ResultMapping resultMapping, Class<?> parameterType, String columnPrefix) throws SQLException {
    final Object parameterObject = instantiateParameterObject(parameterType);
    final MetaObject metaObject = configuration.newMetaObject(parameterObject);
    boolean foundValues = false;
    for (ResultMapping innerResultMapping : resultMapping.getComposites()) {
      final Class<?> propType = metaObject.getSetterType(innerResultMapping.getProperty());
      final TypeHandler<?> typeHandler = typeHandlerRegistry.getTypeHandler(propType);
      final Object propValue = typeHandler.getResult(rs, prependPrefix(innerResultMapping.getColumn(), columnPrefix));
      // issue #353 & #560 do not execute nested query if key is null
      if (propValue != null) {
        metaObject.setValue(innerResultMapping.getProperty(), propValue);
        foundValues = true;
      }
    }
    return foundValues ? parameterObject : null;
  }
```