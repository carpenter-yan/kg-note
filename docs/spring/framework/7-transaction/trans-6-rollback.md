##回滚处理

TransactionAspectSupport.java
```
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
    // 当抛出异常时首先判断当前是否存在事务，这是基础依据
    if (txInfo != null && txInfo.hasTransaction()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                    "] after exception: " + ex);
        }
        // 这里判断是否回滚默认的依据是抛出的异常是存是RuntimeException或者是Error的类型
        if (txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                // 根据TransactionStatus信息进行回滚处理
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
            catch (Error err) {
                logger.error("Application exception overridden by rollback error", ex);
                throw err;
            }
        }
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            // 如果不满足回滚条件即使抛出异常也同样会提交
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
            catch (Error err) {
                logger.error("Application exception overridden by commit error", ex);
                throw err;
            }
        }
    }
}
```

### 1 . 回滚条件
DefaultTransactionAttribute.java
```
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```
默认情况下Spring 中的事务异常处理机制只对RuntimeException和Error两种情况感兴趣，当然你可以通过扩展来改变，
不过，我们最常用的还是使用事务提供的属性设置，利用注解方式的使用
@Transactional(propagation=Propagation.REQUIRED, rollbackFor=Exception.class)

### 2. 回滚处理
AbstractPlatformTransactionManager.java
```
public final void rollback(TransactionStatus status) throws TransactionException {
    // 如果事务已经完成．那么同次回滚会抛出异常
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    processRollback(defStatus);
}

private void processRollback(DefaultTransactionStatus status) {
    try {
        try {
            // 1. 首先是调用自定义前触发器
            triggerBeforeCompletion(status);
            // 2. 针对不同情况做不同的回滚操作
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Rolling back transaction to savepoint");
                }
                //2.1 如果有保存点，也就是当前事务为单独的线程则会退到保存点
                status.rollbackToHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction rollback");
                }
                //2.2 如果当前事务为独立的新事务，则直接回退
                doRollback(status);
            }
            else if (status.hasTransaction()) {
                if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                    }
                    //2.3 如果当前事务不是独立的事务，那么只能标记状态， 等到事务链执行完毕后统一回滚
                    doSetRollbackOnly(status);
                }
                else {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                    }
                }
            }
            else {
                logger.debug("Should roll back transaction but cannot - no transaction available");
            }
        }
        catch (RuntimeException ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }
        catch (Error err) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw err;
        }
        //3. 调用自定义后触发器
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
    }
    finally {
        // 4.消空记录的资源并将挂起的资源恢复
        cleanupAfterCompletion(status);
    }
}
```

DataSourceTransactionManager.java
```
protected void doRollback(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.rollback();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
    }
}
```

### 3.回滚后信息清除
AbstractPlatformTransactionManager.java
```
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    // 1.设置状态是对事务信息作完成标识以避免重复调用
    status.setCompleted();
    
    // 2.如果当前事务是新的同步状态，需要将绑定到当前线程的事务信息清除
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.clear();
    }
    
    // 3.如果是新事务需要做些清除资源的工作。
    if (status.isNewTransaction()) {
        doCleanupAfterCompletion(status.getTransaction());
    }
    // 4.如果在事务执行前有事务挂起，那么当前事务执行结束后需要将挂起事务恢复。
    if (status.getSuspendedResources() != null) {
        if (status.isDebug()) {
            logger.debug("Resuming suspended transaction after completion of inner transaction");
        }
        resume(status.getTransaction(), (SuspendedResourcesHolder) status.getSuspendedResources());
    }
}
```

DataSourceTransactionManager.java
```
protected void doCleanupAfterCompletion(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

    // Remove the connection holder from the thread, if exposed.
    if (txObject.isNewConnectionHolder()) {
        // 将数据库连接从当前线程中接触绑定
        TransactionSynchronizationManager.unbindResource(this.dataSource);
    }

    // Reset connection.
    // 释放连接
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        if (txObject.isMustRestoreAutoCommit()) {
            // 恢复数据库连接的自功提交属性
            con.setAutoCommit(true);
        }
        // 重置数据库连接
        DataSourceUtils.resetConnectionAfterTransaction(con, txObject.getPreviousIsolationLevel());
    }
    catch (Throwable ex) {
        logger.debug("Could not reset JDBC Connection after transaction", ex);
    }

    if (txObject.isNewConnectionHolder()) {
        if (logger.isDebugEnabled()) {
            logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
        }
        // 如果当前事务是独立的新创建的事务则在事务完成后释放数据库连接
        DataSourceUtils.releaseConnection(con, this.dataSource);
    }

    txObject.getConnectionHolder().clear();
}
```