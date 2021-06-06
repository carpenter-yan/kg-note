## 执行期主要类总结
mybatis在执行期间，主要有四大核心接口对象：
- 执行器Executor，执行器负责整个SQL执行过程的总体控制。
- 参数处理器ParameterHandler，参数处理器负责PreparedStatement入参的具体设置。
- 语句处理器StatementHandler，语句处理器负责和JDBC层具体交互，包括prepare语句，执行语句，以及调用ParameterHandler.parameterize()设置参数。
- 结果集处理器ResultSetHandler，结果处理器负责将JDBC查询结果映射到java对象。

### 1.执行器Executor
什么是执行器？所有我们在应用层通过sqlSession执行的各类selectXXX和增删改操作在做了动态sql和参数相关的封装处理后，
都被委托给具体的执行器去执行，包括一、二级缓存的管理，事务的具体管理，Statement和具体JDBC层面优化的实现等等。
所以执行器比较像是sqlSession下的各个策略工厂实现，用户通过配置决定使用哪个策略工厂。
只不过执行器在一个mybatis配置下只有一个，这可能无法适应于所有的情况
接口定义#org.apache.ibatis.executor.Executor

mybatis提供了两种类型的执行器，缓存执行器与非缓存执行器（使用哪个执行器是通过配置文件中settings下的属性defaultExecutorType控制的，默认是SIMPLE），
是否使用缓存执行器则是通过执行cacheEnabled控制的，默认是true。
缓存执行器不是真正功能上独立的执行器，而是非缓存执行器的装饰器模式。
非缓存执行器。非缓存执行器又分为三种，这三种类型的执行器都基于基础执行器BaseExecutor，基础执行器完成了大部分的公共功能
org.apache.ibatis.executor.BaseExecutor.java
```
public abstract class BaseExecutor implements Executor {

  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  //1. 执行器在特定的事务上下文下执行；
  protected Transaction transaction;
  protected Executor wrapper;

  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  //2. 具有本地缓存和本地出参缓存
  protected PerpetualCache localCache;
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;

  protected int queryStack;
  private boolean closed;
  ......  
  
  // BaseExecutor实现了大部分通用功能本地缓存管理、事务提交、回滚、超时设置、延迟加载等，但是将下列4个方法留给了具体的子类实现
  protected abstract int doUpdate(MappedStatement ms, Object parameter)
      throws SQLException;

  protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
      throws SQLException;

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
      throws SQLException;
}
```

### 1.1 SIMPLE执行器
SimpleExecutor.java
简单执行器的实现非常的简单，不展开详述了。
```
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
```

### 1.2 REUSE执行器
ReuseExecutor.java
```
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {
      Connection connection = getConnection(statementLog);
      stmt = handler.prepare(connection, transaction.getTimeout());
      putStatement(sql, stmt);
    }
    handler.parameterize(stmt);
    return stmt;
  }
  
  private boolean hasStatementFor(String sql) {
    try {
      return statementMap.keySet().contains(sql) && !statementMap.get(sql).getConnection().isClosed();
    } catch (SQLException e) {
      return false;
    }
  }
```
从实现可以看出，REUSE和SIMPLE在doUpdate/doQuery上有个差别，不再是每执行一个语句就close掉了，而是尽可能的根据SQL文本进行缓存并重用，
但是由于数据库服务器端通常对每个连接以及全局的语句(oracle称为游标)handler的数量有限制，oracle中是open_cursors参数控制，
mysql中是mysql_stmt_close参数控制，这就会导致如果sql都是靠if各种拼接出来，日积月累可能会导致数据库资源耗尽。
其是否有足够价值，视创建Statement语句消耗的资源占整体资源的比例、以及一共有多少完全不同的Statement数量而定，
一般来说，纯粹的OLTP且非自动生成的sqlmap，它会比SIMPLE执行器更好。

### 1.3 BATCH执行器
BatchExecutor.java
```
public class BatchExecutor extends BaseExecutor {

  public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;
  // 存储在一个事务中的批量DML的语句列表
  private final List<Statement> statementList = new ArrayList<Statement>();
  // 存放DML语句对应的参数对象,包括自动/手工生成的key
  private final List<BatchResult> batchResultList = new ArrayList<BatchResult>();

  // 最新提交执行的SQL语句
  private String currentSql;

  // 最新提交执行的语句
  private MappedStatement currentStatement;


  @Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    // 如果最新执行的一条语句和前面一条语句相同,就不创建新的语句了，直接用缓存的语句，只是把参数对象添加到该语句对应的BatchResult中
    // 否则的话，无论是否在未提交之前，还有pending的语句，都新插入一条语句到list中
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
     handler.parameterize(stmt);//fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    //fix Issues 322
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
  // handler.parameterize(stmt);
    // 调用jdbc的addBatch方法
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
  }

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException {
    Statement stmt = null;
    try {
      flushStatements();
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameterObject, rowBounds, resultHandler, boundSql);
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

  @Override
  protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
    flushStatements();
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, null, boundSql);
    Connection connection = getConnection(ms.getStatementLog());
    Statement stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return handler.<E>queryCursor(stmt);
  }

  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
      List<BatchResult> results = new ArrayList<BatchResult>();
      if (isRollback) {
        return Collections.emptyList();
      }
      for (int i = 0, n = statementList.size(); i < n; i++) {
        Statement stmt = statementList.get(i);
        applyTransactionTimeout(stmt);
        BatchResult batchResult = batchResultList.get(i);
        try {
          batchResult.setUpdateCounts(stmt.executeBatch());
          MappedStatement ms = batchResult.getMappedStatement();
          List<Object> parameterObjects = batchResult.getParameterObjects();
          KeyGenerator keyGenerator = ms.getKeyGenerator();
          if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
            Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
            jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
          } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
            for (Object parameter : parameterObjects) {
              keyGenerator.processAfter(this, ms, stmt, parameter);
            }
          }
          // Close statement to close cursor #1109
          closeStatement(stmt);
        } catch (BatchUpdateException e) {
          StringBuilder message = new StringBuilder();
          message.append(batchResult.getMappedStatement().getId())
              .append(" (batch index #")
              .append(i + 1)
              .append(")")
              .append(" failed.");
          if (i > 0) {
            message.append(" ")
                .append(i)
                .append(" prior sub executor(s) completed successfully, but will be rolled back.");
          }
          throw new BatchExecutorException(message.toString(), e, results, batchResult);
        }
        results.add(batchResult);
      }
      return results;
    } finally {
      for (Statement stmt : statementList) {
        closeStatement(stmt);
      }
      currentSql = null;
      statementList.clear();
      batchResultList.clear();
    }
  }
}
```
批量执行器是JDBC Statement.addBatch的实现,对于批量insert而言比如导入大量数据的ETL,驱动器如果支持的话，能够大幅度的提高DML语句的性能
（首先最重要的是，网络交互就大幅度减少了），比如对于mysql而言，在5.1.13以上版本的驱动，在连接字符串上
rewriteBatchedStatements参数也就是jdbc:mysql://192.168.1.100:3306/test?rewriteBatchedStatements=true后，
性能可以提高几十倍，参见 https://www.cnblogs.com/kxdblog/p/4056010.html　
以及　http://blog.sina.com.cn/s/blog_68b4c68f01013yog.html。
因为BatchExecutor对于每个statementList中的语句，都执行executeBatch()方法，因此最差的极端情况是交叉执行不同的DML SQL语句，
这种情况退化为原始的方式。比如下列形式就是最差的情况：
```
for(int i=0;i<100;i++) {
    session.update("insertUser", userReq);
    session.update("insertUserProfile", userReq);
}
```

### 1.4 缓存执行器CachingExecutor的实现
CachingExecutor.java
```
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    // 首先判断是否启用了二级缓存
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 然后判断缓存中是否有对应的缓存条目(正常情况下，执行DML操作会清空缓存，也可以语句层面明确明确设置)，有的话则返回，这样就不用二次查询了
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
  
  // 存储过程不支持二级缓存
  private void ensureNoOutParams(MappedStatement ms, BoundSql boundSql) {
    if (ms.getStatementType() == StatementType.CALLABLE) {
      for (ParameterMapping parameterMapping : boundSql.getParameterMappings()) {
        if (parameterMapping.getMode() != ParameterMode.IN) {
          throw new ExecutorException("Caching stored procedures with OUT params is not supported.  Please configure useCache=false in " + ms.getId() + " statement.");
        }
      }
    }
  }
```
缓存执行器相对于其他执行器的差别在于，首先是在query()方法中判断是否使用二级缓存(也就是mapper级别的缓存)。
虽然mybatis默认启用了CachingExecutor，但是如果在mapper层面没有明确设置二级缓存的话，就退化为SimpleExecutor了。
二级缓存的维护由TransactionalCache(事务化缓存)负责，当在TransactionalCacheManager(事务化缓存管理器)中
调用putObject和removeObject方法的时候并不是马上就把对象存放到缓存或者从缓存中删除，
而是先把这个对象放到entriesToAddOnCommit和entriesToRemoveOnCommit这两个HashMap之中的一个里，
然后当执行commit/rollback方法时再真正地把对象存放到缓存或者从缓存中删除，具体可以参见TransactionalCache.commit/rollback方法。
还有一个差别是使用了TransactionalCacheManager管理事务，其他逻辑就一样了。

### 2.参数处理器ParameterHandler
ParameterHandler只有一个默认实现DefaultParameterHandler

### 3.语句处理器StatementHandler
从接口可以看出，StatementHandler主要包括prepare语句、给语句设置参数、执行语句获取要执行的SQL语句本身。
mybatis包含了三种类型的StatementHandler实现
分别用于JDBC对应的PrepareStatement,Statement以及CallableStatement。
BaseStatementHandler是这三种类型语句处理器的抽象父类，封装了一些实现细节比如设置超时时间、结果集每次提取大小等操作

### 4.结果集处理器ResultSetHandler
结果集处理器,顾名知义,就是用了对查询结果集进行处理的,目标是将JDBC结果集映射为业务对象
```
public interface ResultSetHandler {

  <E> List<E> handleResultSets(Statement stmt) throws SQLException;

  <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

  void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```
接口中定义的三个接口分别用于处理常规查询的结果集,游标查询的结果集以及存储过程调用的出参设置。
和参数处理器一样,结果集处理器也只有一个默认实现DefaultResultSetHandler。
结果集处理器的功能包括对象的实例化、属性自动匹配计算、常规属性赋值、嵌套ResultMap的处理、嵌套查询的处理、鉴别器结果集的处理等，
每个功能我们在分析SQL语句执行selectXXX的时候都详细的讲解过了，具体可以参见selectXXX部分

