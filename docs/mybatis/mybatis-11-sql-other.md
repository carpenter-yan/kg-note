### 1.selectMap实现
DefaultSqlSession.java
```
  public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
    final List<? extends V> list = selectList(statement, parameter, rowBounds);
    final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<>(mapKey,
            configuration.getObjectFactory(), configuration.getObjectWrapperFactory(), configuration.getReflectorFactory());
    final DefaultResultContext<V> context = new DefaultResultContext<>();
    for (V o : list) {
      context.nextResultObject(o);
      mapResultHandler.handleResult(context);
    }
    return mapResultHandler.getMappedResults();
  }
```
和selectList不同，selectMap多了一个参数mapKey，mapKey就是用来指定返回类型中作为key的那个字段名，具体的核心逻辑委托给了selectList方法，
只是在返回结果后，mapResultHandler进行了二次处理。DefaultMapResultHandler是众多ResultHandler的实现之一。
DefaultMapResultHandler.handleResult()的功能就是把List转换为Map<object.prop1,object>格式。

### 2.update/insert/delete实现
DefaultSqlSession.java
```
  public int update(String statement, Object parameter) {
    try {
      dirty = true;
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
首先设置了字段dirty=true(dirty主要用在非自动提交模式下，用于判断是否需要提交或回滚，
在强行提交模式下，如果dirty=true，则需要提交或者回滚，代表可能有pending的事务)

BaseExecutor.java
```
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 清空本地缓存, 与本地出参缓存
    clearLocalCache();
    // 调用具体执行器实现的doUpdate方法
    return doUpdate(ms, parameter);
  }
```

SimpleExecutor.java
```
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
```
其中的逻辑可以发现，和selectList的实现非常相似，先创建语句处理器，然后创建Statement实例，最后调用语句处理的update，
语句处理器里面调用jdbc对应update的方法execute()。
和selectList的不同之处在于：
在创建语句处理器期间，会根据需要调用KeyGenerator.processBefore生成前置id；
在执行完成execute()方法后，会根据需要调用KeyGenerator.processAfter生成后置id；
通过分析delete/insert，我们会发现他们内部都委托给update实现了，所以我们就不做重复的分析了。

### 3.SQL语句执行方式二 SqlSession.getMapper实现
DefaultSqlSession.java
```
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```

Configuration.java
```
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

MapperRegistry.java
```
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

MapperProxyFactory.java
```
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```
MapperProxyFactory首先为Mapper接口创建了一个实现了InvocationHandler方法调用处理器接口的代理类MapperProxy，
并实现invoke接口（其中为mapper各方法执行sql的具体逻辑），最后才调用JDK的java.lang.reflect.Proxy为Mapper接口创建动态代理类并返回
这样当我们应用层执行List users = mapper.getUser2(293);的时候，JVM会首先调用MapperProxy.invoke

MapperProxy.java
```
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

MapperMethod.java
```
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        // 主要用于BatchExecutor和CacheExecutor的场景,SimpleExecutor模式不适用
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
对非查询类SQL，首先将请求参数转换为mybatis内部的格式，然后调用sqlSession实例对应的方法，这就和第一种方式的SQL逻辑一样的。
对于查询类SQL，根据返回类型是void/many/map/one/cursor分别调用不同的实现，但主体逻辑都类似，都是调用sqlSession.selectXXX

### 4.动态sql
准确的说,只要mybatis的的crud语句中包含了if/where/sql等标签或者${}之后，就已经算是动态sql了，
所以只要在mybatis加载mapper文件期间被解析为非StaticSqlSource，就会被当做动态sql处理，
在执行selectXXX或者update/insert/delete期间，就会调用对应的SqlNode接口和TextSqlNode.isDynamic()处理各自的标签以及${}，
并最终将每个sql片段处理到StaticTextSqlNode并生成最终的参数化静态SQL语句为止。
所以，可以说，在绝大部分非PK查询的情况下，我们都是在使用动态SQL。

### 5.存储过程与函数调用实现
如果MappedStatement.StatementType类型为CALLABLE，在Executor.doQuery方法中创建语句处理器的时候，
就会返回CallableStatementHandler实例，随后在调用语句处理器的初始化语句和设置参数 方法时，调用jdbc对应存储过程的prepareCall方法
CallableStatementHandler.java
```
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      return connection.prepareCall(sql);
    } else {
      return connection.prepareCall(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
  }
```

### 6.mybatis事务实现
mybatis的事务管理模式分为两种，自动提交和手工提交，DefaultSqlSessionFactory的openSession中重载中，提供了一个参数用于控制是否自动提交事务，
该参数最终被传递给 java.sql.Connection.setAutoCommit()方法用于控制是否自动提交事务(默认情况下，连接是自动提交的)
DefaultSqlSessionFactory.java
```
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```
如上所示，返回的事务传递给了执行器，因为执行器是在事务上下文中执行，所以对于自动提交模式，实际上mybatis不需要去关心。
只有非自动管理模式，mybatis才需要关心事务。对于非自动提交模式，通过sqlSession.commit()或sqlSession.rollback()发起，
在进行提交或者回滚的时候会调用isCommitOrRollbackRequired判断是否应该提交或者回滚事务
```
  private boolean isCommitOrRollbackRequired(boolean force) {
    return (!autoCommit && dirty) || force;
  }
```
只有非自动提交模式且执行过DML操作或者设置强制提交才会认为应该进行事务提交或者回滚操作。
对于不同的执行器，在提交和回滚执行的逻辑不一样，因为每个执行器在一级、二级、语句缓存上的差异： 
- 对于简单执行器，除了清空一级缓存外，什么都不做；
- 对于REUSE执行器，关闭每个缓存的Statement以释放服务器端语句处理器，然后清空缓存的语句；
- 对于批量处理器，则执行每个批处理语句的executeBatch()方法以便真正执行语句，然后关闭Statement；  

上述逻辑执行完成后，会执行提交/回滚操作。对于缓存执行器，在提交/回滚完成之后，
会将TransactionCache中的entriesMissedInCache和entriesToAddOnCommit列表分别移动到语句对应的二级缓存中或清空掉。

### 7.缓存
只要实现org.apache.ibatis.cache.Cache接口的任何类都可以当做缓存
mybatis提供了基本实现org.apache.ibatis.cache.impl.PerpetualCache，内部采用原始HashMap实现。
第二个需要知道的方面是mybatis有一级缓存和二级缓存。一级缓存是SqlSession级别的缓存，
不同SqlSession之间的缓存数据区域（HashMap）是互相不影响，MyBatis默认支持一级缓存，不需要任何的配置，
默认情况下(一级缓存的有效范围可通过参数localCacheScope参数修改，取值为SESSION或者STATEMENT)，
在一个SqlSession的查询期间，只要没有发生commit/rollback或者调用close()方法，
那么mybatis就会先根据当前执行语句的CacheKey到一级缓存中查找，如果找到了就直接返回，不到数据库中执行。
其实现在代码BaseExecutor.query()中
BaseExecutor.java
```
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 如果在一级缓存中就直接获取
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      // 如果设置了一级缓存是STATEMENT级别而非默认的SESSION级别，一级缓存就去掉了
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```

二级缓存是mapper级别的缓存，多个SqlSession去操作同一个mapper的sql语句，多个SqlSession可以共用二级缓存，二级缓存是跨SqlSession。
二级缓存默认不启用，需要通过在Mapper中明确设置cache，它的实现在CachingExecutor的query()方法中
CachingExecutor.java
```
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      // 如果二级缓存中找到了记录就直接返回,否则到DB查询后进行缓存
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
对于一级缓存，commit/rollback都会清空一级缓存。
对于二级缓存，DML操作或者显示设置语句层面的flushCache属性都会使得二级缓存失效。
在二级缓存容器的具体回收策略实现上，有下列几种：
- LRU – 最近最少使用的：移除最长时间不被使用的对象，也是默认的选项，其实现类是org.apache.ibatis.cache.decorators.LruCache。
- FIFO – 先进先出：按对象进入缓存的顺序来移除它们，其实现类是org.apache.ibatis.cache.decorators.FifoCache。
- SOFT – 软引用：移除基于垃圾回收器状态和软引用规则的对象，其实现类是org.apache.ibatis.cache.decorators.SoftCache。
- WEAK – 弱引用：更积极地移除基于垃圾收集器状态和弱引用规则的对象，其实现类是org.apache.ibatis.cache.decorators.WeakCache。  

在缓存的设计上，Mybatis的所有Cache算法都是基于装饰器/Composite模式对PerpetualCache扩展增加功能。
对于模块化微服务系统来说，应该来说mybatis的一二级缓存对业务数据都不适合，
尤其是对于OLTP系统来说，CRM/BI这些不算，如果要求数据非常精确的话，也不是特别合适。
对这些要求数据准确的系统来说，尽可能只使用mybatis的ORM特性比较靠谱。
但是有一部分数据如果前期没有很少的设计缓存的话，是很有价值的，
比如说对于一些配置类数据比如数据字典、系统参数、业务配置项等很少变化的数据。

### 8.插件
插件几乎是所有主流框架提供的一种扩展方式之一，插件可以用于记录日志，统计运行时性能，为核心功能提供额外的辅助支持。
在mybatis中，插件是在内部是通过拦截器实现的。要开发自定义自定义插件，只要实现org.apache.ibatis.plugin.Interceptor接口即可

mybatis提供了为插件配置提供了两个注解：org.apache.ibatis.plugin.Signature和org.apache.ibatis.plugin.Intercepts。
Intercepts注解用来指示当前类是一个拦截器

它有一个类型为Signature数组的value属性，如果没有指定，它会拦截StatementHandler、ResultSetHandler、ParameterHandler
和Executor这四个核心接口对象中的所有方法。如需改变默认行为，可以通过明确设置value的值

在实际使用中，使用最频繁的mybatis插件应该算是分页查询插件了，最流行的应该是com.github.pagehelper.PageHelper了。