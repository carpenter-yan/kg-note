##创建事务
TransactionAspectSupport.java
```
protected TransactionInfo createTransactionIfNecessary(
        PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

    // If no name specified, apply method identification as transaction name.
    // 1. 使用DelegatingTransactionAttribute封装传人的TransactionAttribute实例。
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }
    // 2. 获取事务。
    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction(txAttr);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                        "] because no transaction manager has been configured");
            }
        }
    }
    // 3. 构建事务信息
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

### 获取事务
AbstractPlatformTransactionManager.java
```
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    // 1. 获取事务
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    // 2. 如果当先线程存在事务则转向嵌套事务的处理。
    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    // 3. 事务超时设置验证。
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    // 4 . 事务propagationBehavior属性的设置验证。
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            // 5. 构建DefaultTransactionStatus 。
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 6. 完善transaction，包括设置ConnectionHolder、隔离级别、timeout，如果是新连接，则绑定到当前线程。
            doBegin(transaction, definition);
            // 新同步事务的设置，针对于当前线程的设置.
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + definition);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}
```

### 1.获取事务
DataSourceTransactionManager.java
```
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    // 如果当前线程已经记录数据库连接则使用原有连接
    ConnectionHolder conHolder =
            (ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
    // false表示非新创建连接
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}

protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        // 1. 尝试获取连接。
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = this.dataSource.getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        // 2 . 设置隔离级别以及只读标识
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);

        // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
        // so we don't want to do it unnecessarily (for example if we've explicitly
        // configured the connection pool to set it already).
        // 3. 更改默认的提交设置
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        // 4. 设置标志位，标识当前连接已经被事务激活
        txObject.getConnectionHolder().setTransactionActive(true);

        // 5. 设置过期时间
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        // 6. 将connectionHolder绑定到当前线程
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, this.dataSource);
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
```
可以说事务是从这个函数开始的，因为在这个函数中已经开始尝试了对数据库连接的获取，
当然，在获取数据库连接的同时， 一些必要的设置也是需要同步设置的。

DataSourceUtils.java
```
public static Integer prepareConnectionForTransaction(Connection con, TransactionDefinition definition)
        throws SQLException {

    Assert.notNull(con, "No Connection specified");

    // Set read-only flag.
    // 设置数据连接的只读标识
    if (definition != null && definition.isReadOnly()) {
        try {
            if (logger.isDebugEnabled()) {
                logger.debug("Setting JDBC Connection [" + con + "] read-only");
            }
            con.setReadOnly(true);
        }
        catch (SQLException ex) {
            Throwable exToCheck = ex;
            while (exToCheck != null) {
                if (exToCheck.getClass().getSimpleName().contains("Timeout")) {
                    // Assume it's a connection timeout that would otherwise get lost: e.g. from JDBC 4.0
                    throw ex;
                }
                exToCheck = exToCheck.getCause();
            }
            // "read-only not supported" SQLException -> ignore, it's just a hint anyway
            logger.debug("Could not set JDBC Connection read-only", ex);
        }
        catch (RuntimeException ex) {
            Throwable exToCheck = ex;
            while (exToCheck != null) {
                if (exToCheck.getClass().getSimpleName().contains("Timeout")) {
                    // Assume it's a connection timeout that would otherwise get lost: e.g. from Hibernate
                    throw ex;
                }
                exToCheck = exToCheck.getCause();
            }
            // "read-only not supported" UnsupportedOperationException -> ignore, it's just a hint anyway
            logger.debug("Could not set JDBC Connection read-only", ex);
        }
    }

    // Apply specific isolation level, if any.
    // 设置数据库连接的隔离级别
    Integer previousIsolationLevel = null;
    if (definition != null && definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
        if (logger.isDebugEnabled()) {
            logger.debug("Changing isolation level of JDBC Connection [" + con + "] to " +
                    definition.getIsolationLevel());
        }
        int currentIsolation = con.getTransactionIsolation();
        if (currentIsolation != definition.getIsolationLevel()) {
            previousIsolationLevel = currentIsolation;
            con.setTransactionIsolation(definition.getIsolationLevel());
        }
    }

    return previousIsolationLevel;
}
```

### 2.处理已存在事务
```
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                    definition.getName() + "]");
        }
        // 新事务的建立
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
        catch (Error beginErr) {
            resumeAfterBeginException(transaction, suspendedResources, beginErr);
            throw beginErr;
        }
    }

    // 嵌入式事务的处理
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions by default - " +
                    "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        // 如果没有可以使用保存点的方式控制事务回滚，那么在嵌入式事务的建立初始建立保存点
        if (useSavepointForNestedTransaction()) {
            // Create savepoint within existing Spring-managed transaction,
            // through the SavepointManager API implemented by TransactionStatus.
            // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
            DefaultTransactionStatus status =
                    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            // Nested transaction through nested begin and commit/rollback calls.
            // Usually only for JTA: Spring synchronization might get activated here
            // in case of a pre-existing JTA transaction.
            // 有些情况是不能使用保存点操作，比如JTA，那么建立新事务
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, null);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }

    // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] specifies isolation level which is incompatible with existing transaction: " +
                        (currentIsolationLevel != null ?
                                isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                "(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}

对于挂起操作的主要目的是记录原有事务的状态，以便于后续操作对事务的恢复
protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
        try {
            Object suspendedResources = null;
            if (transaction != null) {
                suspendedResources = doSuspend(transaction);
            }
            String name = TransactionSynchronizationManager.getCurrentTransactionName();
            TransactionSynchronizationManager.setCurrentTransactionName(null);
            boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
            TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
            Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
            boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
            TransactionSynchronizationManager.setActualTransactionActive(false);
            return new SuspendedResourcesHolder(
                    suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
        }
        catch (RuntimeException ex) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw ex;
        }
        catch (Error err) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw err;
        }
    }
    else if (transaction != null) {
        // Transaction active but no synchronization active.
        Object suspendedResources = doSuspend(transaction);
        return new SuspendedResourcesHolder(suspendedResources);
    }
    else {
        // Neither transaction nor synchronization active.
        return null;
    }
}
```

### 3. 构建事务信息
当已经建立事务连接并完成了事务信息的提取后，我们需要将所有的事务信息统一记录在TransactionInfo类型的实例中，
这个实例包含了目标方法开始前的所有状态信息，一旦事务执行失败，Spring会通过TransactionInfo类型的实例中的信息来进行回滚等后续工作。

TransactionAspectSupport.java
```
protected TransactionInfo prepareTransactionInfo(PlatformTransactionManager tm,
        TransactionAttribute txAttr, String joinpointIdentification, TransactionStatus status) {

    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
    if (txAttr != null) {
        // We need a transaction for this method...
        if (logger.isTraceEnabled()) {
            logger.trace("Getting transaction for [" + txInfo.getJoinpointIdentification() + "]");
        }
        // The transaction manager will flag an error if an incompatible tx already exists.
        // 记录事务状态
        txInfo.newTransactionStatus(status);
    }
    else {
        // The TransactionInfo.hasTransaction() method will return false. We created it only
        // to preserve the integrity of the ThreadLocal stack maintained in this class.
        if (logger.isTraceEnabled())
            logger.trace("Don't need to create transaction for [" + joinpointIdentification +
                    "]: This method isn't transactional.");
    }

    // We always bind the TransactionInfo to the thread, even if we didn't create
    // a new transaction here. This guarantees that the TransactionInfo stack
    // will be managed correctly even if no transaction was created by this aspect.
    txInfo.bindToThread();
    return txInfo;
}
```
