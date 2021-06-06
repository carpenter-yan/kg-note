## SQL语句的执行流程
### 传统JDBC用法
```
package org.mybatis.internal.example;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.sql.Statement;

public class JdbcHelloWord {
    public static void main(String arg[]) {
        try {
            Connection con = null; //定义一个MYSQL链接对象
            // 1.注册驱动
            Class.forName("com.mysql.jdbc.Driver").newInstance(); //MYSQL驱动
            // 2.获取jdbc连接
            con = DriverManager.getConnection("jdbc:mysql://10.7.12.4:3306/lfBase?useUnicode=true", "lfBase", "eKffQV6wbh3sfQuFIG6M"); //链接本地MYSQL

            // 3.创建参数化预编译SQL
            String updateSql = "UPDATE LfParty SET remark1 = 'mybatis internal example' WHERE lfPartyId = ?";
            PreparedStatement pstmt = con.prepareStatement(updateSql);
            // 4.绑定参数
            pstmt.setString(1, "1");
            // 5.发送SQL给数据库进行执行
            long updateRes = pstmt.executeUpdate();
            System.out.print("UPDATE:" + updateRes);

            String sql = "select lfPartyId,partyName from LfParty where lfPartyId = ?";
            PreparedStatement pstmt2 = con.prepareStatement(sql);
            pstmt2.setString(1, "1");
            ResultSet rs = pstmt2.executeQuery();
            // 6.对于查询，获取结果集到应用
            while (rs.next()) { //循环输出结果集
                String lfPartyId = rs.getString("lfPartyId");
                String partyName = rs.getString("partyName");
                System.out.print("\r\n\r\n");
                System.out.print("lfPartyId:" + lfPartyId + "partyName:" + partyName);
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### mybatis执行SQL语句
```
SqlSession session = SqlSessionFactory.openSession();
try {
    User user = (User) session.selectOne("org.mybatis.internal.example.mapper.UserMapper.getUser", 1);
    System.out.println("sql from xml:" + user.getLfPartyId() + "," + user.getPartyName());
    UserMapper2 mapper = session.getMapper(UserMapper2.class);
    List<User> users = mapper.getUser2(293);
    System.out.println("sql from mapper:" + users.get(0).getLfPartyId() + "," + users.get(0).getPartyName());
} finally {
    session.close();
}
```
同样的，在mybatis中，要执行sql语句，首先要拿到代表JDBC底层连接的一个对象，这在mybatis中的实现就是SqlSession
首先调用SqlSessionFactory.openSession()拿到一个session，然后在session上执行各种CRUD操作。
简单来说，SqlSession就是jdbc连接的代表，openSession()就是获取jdbc连接（当然其背后可能是从jdbc连接池获取）；
session中的各种selectXXX方法或者调用mapper的具体方法就是集合了JDBC调用的第3、4、5、6步

### 1.获取openSession
mybatis提供了两个SqlSessionFactory实现：SqlSessionManager和DefaultSqlSessionFactory，默认返回的是DefaultSqlSessionFactory
```
public interface SqlSessionFactory {

  SqlSession openSession();

  SqlSession openSession(boolean autoCommit);
  SqlSession openSession(Connection connection);
  SqlSession openSession(TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType);
  SqlSession openSession(ExecutorType execType, boolean autoCommit);
  SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
  SqlSession openSession(ExecutorType execType, Connection connection);

  Configuration getConfiguration();
}

```
主要有多种形式的重载，除了使用默认设置外，可以指定自动提交模式、特定的jdbc连接、事务隔离级别，以及指定的执行器类型。
关于执行器类型，mybatis提供了三种执行器类型：SIMPLE, REUSE, BATCH。
我们以最简单的无参方法切入（按照一般的套路，如果定义了多个重载的方法或者构造器，内部实现一定是设置作者认为最合适的默认值，然后调用多参数的方法）

DefaultSqlSessionFactory.java
```
  public SqlSession openSession() {
    // 使用默认的执行器类型(默认是SIMPLE)，默认隔离级别，非自动提交 委托给openSessionFromDataSource方法
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
  
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 1.1 获取事务管理器, 支持从数据源或者直接获取
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      /** 从数据源创建一个事务,同样,数据源必须配置,mybatis内置了JNDI、POOLED、UNPOOLED三种类型的数据源,
       * 其中POOLED对应的实现为org.apache.ibatis.datasource.pooled.PooledDataSource,
       * 它是mybatis自带实现的一个同步、线程安全的数据库连接池 一般在生产中,我们会使用dbcp或者druid连接池*/
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 1.2 创建执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 1.3 创建sql会话
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  
```

### 1.1 获取事务管理器
DefaultSqlSessionFactory.java
```
  private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
    /**  如果没有配置environment或者environment的事务管理器为空,则使用受管的事务管理器
     * 除非什么都没有配置,否则在mybatis-config里面,至少要配置一个environment，此时事务工厂不允许为空
     * 对于jdbc类型的事务管理器,则返回JdbcTransactionFactory,其内部操作mybatis的JdbcTransaction实现(采用了Facade模式)
     */
    if (environment == null || environment.getTransactionFactory() == null) {
      return new ManagedTransactionFactory();
    }
    return environment.getTransactionFactory();
  }
```

JdbcTransactionFactory.java
```
  public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
    return new JdbcTransaction(ds, level, autoCommit);
  }
```
newTransaction的实现逻辑很简单，但是此时返回的事务不一定是有底层连接的。

### 1.2 创建执行器
Configuration.java
```
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
如果没有配置执行器类型，默认是简单执行器。如果启用了缓存，则使用缓存执行器

### 2.sql语句执行
sql语句执行的入口分为两种：
方式一，调用org.apache.ibatis.session.SqlSession的crud方法比如selectList/selectOne传递完整的语句id直接执行；
方式二，先调用SqlSession的getMapper()方法得到mapper接口的一个实现，然后调用具体的方法。除非早期，现在实际开发中，我们一般采用这种方式。

### 2.1 sql执行方式一
我们先来看第一种形式的sql语句执行也就是SqlSession.getMapper除外的形式。这里还是以带参数的session.selectOne为例子
DefaultSqlSession.java
```
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }
  
  public <E> List<E> selectList(String statement, Object parameter) {
    // 从定义可知，RowBounds是一个分页查询的参数封装，默认是不分页
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }
  
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 判断configuration.mappedStatements里面是否有这个语句，如果没有将会抛出IllegalArgumentException异常
      MappedStatement ms = configuration.getMappedStatement(statement);
      // wrapCollection主要是判断参数是否为数组或集合类型类型，是的话将他们包装到StrictMap中
      // 2.1.1 执行器执行查询
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

### 2.1.1 执行器执行查询
BaseExecutor.java
```
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 首先根据传递的参数获取BoundSql对象，对于不同类型的SqlSource，对应的getBoundSql实现不同
    BoundSql boundSql = ms.getBoundSql(parameter);
    //2.1.1.1 创建缓存key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //2.1.1.2 委托给重载的query
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }
  
  ###2.1.1.1 创建缓存key
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 根据映射语句id,分页信息,jdbc规范化的预编译sql,所有映射参数的值以及环境id的值,计算出缓存Key
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
  
  ###2.1.1.2 委托给重载的query
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 是否需要刷新缓存
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 如果查询不需要应用结果处理器,则先从缓存获取,这样可以避免数据库查询。
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        //2.1.1.2.1 从缓存中获取参数
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //2.1.1.2.2 不管是因为需要应用结果处理器还是缓存中没有,都从数据库中查询
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
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
  
  ###2.1.1.2.1 
  private void handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql) {
    /** 只处理存储过程和函数调用的出参, 因为存储过程和函数的返回不是通过ResultMap而是ParameterMap来的，
     * 所以只要把缓存的非IN模式参数取出来设置到parameter对应的属性上即可 */
    if (ms.getStatementType() == StatementType.CALLABLE) {
      final Object cachedParameter = localOutputParameterCache.getObject(key);
      if (cachedParameter != null && parameter != null) {
        final MetaObject metaCachedParameter = configuration.newMetaObject(cachedParameter);
        final MetaObject metaParameter = configuration.newMetaObject(parameter);
        for (ParameterMapping parameterMapping : boundSql.getParameterMappings()) {
          if (parameterMapping.getMode() != ParameterMode.IN) {
            final String parameterName = parameterMapping.getProperty();
            final Object cachedValue = metaCachedParameter.getValue(parameterName);
            metaParameter.setValue(parameterName, cachedValue);
          }
        }
      }
    }
  }
  
  ###2.1.1.2.2 从数据库中查询
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 一开始放个占位符进去,这个还真不知道用意是什么???
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      //2.1.2 doQuery是个抽象方法,每个具体的执行器都要自己去实现,我们先看SIMPLE的
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    // 把真正的查询结果放到缓存中去
    localCache.putObject(key, list);
    // 如果是存储过程类型,则把查询参数放到本地出参缓存中, 所以第一次一定为空
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```

### 2.1.2 执行器执行SQL
SimpleExecutor.java
```
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      /** 根据上下文参数和具体的执行器new一个StatementHandler, 其中包含了所有必要的信息,
       * 比如结果处理器、参数处理器、执行器等等,主要有三种类型的语句处理器UNPREPARE、PREPARE、CALLABLE。
       * 默认是PREPARE类型，通过mapper语句上的statementType属性进行设置,一般除了存储过程外不应该设置*/
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //2.1.2.1 这一步是真正和JDBC打交道
      stmt = prepareStatement(handler, ms.getStatementLog());
      //2.1.2.2 执行查询
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 获取JDBC连接
    Connection connection = getConnection(statementLog);
    //2.1.2.1.1 调用语句处理器的prepare方法
    stmt = handler.prepare(connection, transaction.getTimeout());
    //2.1.2.1.2 设置参数
    handler.parameterize(stmt);
    return stmt;
  }
```

### 2.1.2.1.1 调用语句处理器的prepare方法
BaseStatementHandler.java
```
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      // 首先实例化语句，因为PREPARE和非PREPARE不同,所以留给具体子类实现
      statement = instantiateStatement(connection);
      // 设置语句超时时间
      setStatementTimeout(statement, transactionTimeout);
      // 设置fetch大小
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
```

PreparedStatementHandler.java
```
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      return connection.prepareStatement(sql);
    } else {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
  }
```
基本上就是把我们在MAPPER中定义的属性转换为JDBC标准的调用。

### 2.1.2.1.2 设置参数
PreparedStatementHandler.java
```
  public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```
DefaultParameterHandler.java
```
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        // 仅处理非出参
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          /**  计算参数值的优先级是 先判断是不是属于语句的AdditionalParameter；其次参数是不是null；然后判断是不是属于注册类型；
           * 都不是，那估计参数一定是object或者map了,这就要借助于MetaObject获取属性值了；*/
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            // jdbc下标从1开始，由具体的类型处理器进行参数的设置, 对于每个jdbcType, mybatis都提供了一个对应的Handler,
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```

### 2.1.2.1.2 设置参数
PreparedStatementHandler.java
```
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    // 调用结果处理--->>>
    return resultSetHandler.handleResultSets(ps);
  }
```
从上述具体逻辑的实现可以看出，内部调用PreparedStatement完成具体查询后，将ps的结果集传递给对应的结果处理器进行处理。
查询结果的映射是mybatis作为ORM框架提供的最有价值的功能，同时也可以说是最复杂的逻辑之一

