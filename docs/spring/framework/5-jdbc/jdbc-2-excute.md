##execute函数

execute作为数据库操作的核心入口，将大多数数据库操作相同的步骤统一封装，而将个性化的操作使用参数PreparedStatementCallback进行回调。
JdbcTemplate.execute
```
public <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action)
        throws DataAccessException {

    Assert.notNull(psc, "PreparedStatementCreator must not be null");
    Assert.notNull(action, "Callback object must not be null");
    if (logger.isDebugEnabled()) {
        String sql = getSql(psc);
        logger.debug("Executing prepared SQL statement" + (sql != null ? " [" + sql + "]" : ""));
    }

    //1. 获取数据库连接
    Connection con = DataSourceUtils.getConnection(getDataSource());
    PreparedStatement ps = null;
    try {
        Connection conToUse = con;
        if (this.nativeJdbcExtractor != null &&
                this.nativeJdbcExtractor.isNativeConnectionNecessaryForNativePreparedStatements()) {
            conToUse = this.nativeJdbcExtractor.getNativeConnection(con);
        }
        //2. 创建statement
        ps = psc.createPreparedStatement(conToUse);
        //3. 应用用户设定的输入参数
        applyStatementSettings(ps);
        PreparedStatement psToUse = ps;
        if (this.nativeJdbcExtractor != null) {
            psToUse = this.nativeJdbcExtractor.getNativePreparedStatement(ps);
        }
        //4. 调用回调函数
        T result = action.doInPreparedStatement(psToUse);
        //5. 警告处理
        handleWarnings(ps);
        return result;
    }
    catch (SQLException ex) {
        // Release Connection early, to avoid potential connection pool deadlock
        // in the case when the exception translator hasn't been initialized yet.
        if (psc instanceof ParameterDisposer) {
            ((ParameterDisposer) psc).cleanupParameters();
        }
        String sql = getSql(psc);
        psc = null;
        JdbcUtils.closeStatement(ps);
        ps = null;
        DataSourceUtils.releaseConnection(con, getDataSource());
        con = null;
        throw getExceptionTranslator().translate("PreparedStatementCallback", sql, ex);
    }
    finally {
        //6. 释放资源
        if (psc instanceof ParameterDisposer) {
            ((ParameterDisposer) psc).cleanupParameters();
        }
        JdbcUtils.closeStatement(ps);
        DataSourceUtils.releaseConnection(con, getDataSource());
    }
}
```

###1. 获取数据库连接
```
public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    Assert.notNull(dataSource, "No DataSource specified");

    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
        conHolder.requested();
        if (!conHolder.hasConnection()) {
            logger.debug("Fetching resumed JDBC Connection from DataSource");
            conHolder.setConnection(dataSource.getConnection());
        }
        return conHolder.getConnection();
    }
    // Else we either got no holder or an empty thread-bound holder here.

    logger.debug("Fetching JDBC Connection from DataSource");
    Connection con = dataSource.getConnection();

    // 当前线程支持同步
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        logger.debug("Registering transaction synchronization for JDBC Connection");
        // Use same Connection for further JDBC actions within the transaction.
        // Thread-bound object will get removed by synchronization at transaction completion.
        ConnectionHolder holderToUse = conHolder;
        if (holderToUse == null) {
            holderToUse = new ConnectionHolder(con);
        }
        else {
            holderToUse.setConnection(con);
        }
        // 记录数据库连接
        holderToUse.requested();
        TransactionSynchronizationManager.registerSynchronization(
                new ConnectionSynchronization(holderToUse, dataSource));
        holderToUse.setSynchronizedWithTransaction(true);
        if (holderToUse != conHolder) {
            TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
        }
    }

    return con;
}
```
在数据库连接方面，Spring主要考虑的是关于事务方面的处理。基于事务处理的特殊性，Spring需要保证线程中的数据库操作都是使用同一个事务连接。

###3. 应用用户设定的参数
```
protected void applyStatementSettings(Statement stmt) throws SQLException {
    /** setFetchSize的意思是当调用rs.next 时，ResultSet会一次性从服务器上取得多少行数据回来，
     * 这样在下次rs.next 时， 它可以直接从内存中获取数据而不需要网络交互， 提高了效率。*/
    int fetchSize = getFetchSize();
    if (fetchSize != -1) {
        stmt.setFetchSize(fetchSize);
    }
    // setMaxRows讲此Statement对象生成的所有ResultSet对象可以包含的最大行数限制设置为给定数。
    int maxRows = getMaxRows();
    if (maxRows != -1) {
        stmt.setMaxRows(maxRows);
    }
    DataSourceUtils.applyTimeout(stmt, getDataSource(), getQueryTimeout());
}
```

###4. 调用回调函数
处理一些通用方法外的个性化处理，也就是PreparedStatementCallback类型的参数的doInPreparedStatement方法的回调。

###5. 警告处理
```
protected void handleWarnings(Statement stmt) throws SQLException {
    if (isIgnoreWarnings()) {
        if (logger.isDebugEnabled()) {
            SQLWarning warningToLog = stmt.getWarnings();
            while (warningToLog != null) {
                logger.debug("SQLWarning ignored: SQL state '" + warningToLog.getSQLState() + "', error code '" +
                        warningToLog.getErrorCode() + "', message [" + warningToLog.getMessage() + "]");
                warningToLog = warningToLog.getNextWarning();
            }
        }
    }
    else {
        handleWarnings(stmt.getWarnings());
    }
}
```
处理警告的方式，如默认的是忽略警告，当出现警告时只打印警告日志，而另一种方式只直接抛出异常。

###6. 释放资源
DataSourceUtils.releaseConnection
```
public static void releaseConnection(Connection con, DataSource dataSource) {
    try {
        doReleaseConnection(con, dataSource);
    }
    catch (SQLException ex) {
        logger.debug("Could not close JDBC Connection", ex);
    }
    catch (Throwable ex) {
        logger.debug("Unexpected exception on closing JDBC Connection", ex);
    }
}

public static void doReleaseConnection(Connection con, DataSource dataSource) throws SQLException {
    if (con == null) {
        return;
    }
    if (dataSource != null) {
        ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
        if (conHolder != null && connectionEquals(conHolder, con)) {
            // It's the transactional Connection: Don't close it.
            conHolder.released();
            return;
        }
    }
    logger.debug("Returning JDBC Connection to DataSource");
    doCloseConnection(con, dataSource);
}

public static void doCloseConnection(Connection con, DataSource dataSource) throws SQLException {
    if (!(dataSource instanceof SmartDataSource) || ((SmartDataSource) dataSource).shouldClose(con)) {
        con.close();
    }
}
```