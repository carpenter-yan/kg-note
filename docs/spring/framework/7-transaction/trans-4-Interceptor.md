## 事务增强器
TransactionInterceptor类继承自MethodInterceptor，所以调用该类是从其invoke方法开始的
TransactionInterceptor.java
```
public Object invoke(final MethodInvocation invocation) throws Throwable {
    // Work out the target class: may be {@code null}.
    // The TransactionAttributeSource should be passed the target class
    // as well as the method, which may be from an interface.
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // Adapt to TransactionAspectSupport's invokeWithinTransaction...
    return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
        @Override
        public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();
        }
    });
}
```

TransactionAspectSupport.java
```
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
        throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    // 1.获取对应事务属性
    final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
    // 2.加载配置中配置的TransactionManager
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    // 构造方法唯一标识(类.方法，如service.UserServiceImpl.save)
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // 3. 不同的事务处理方式使用不同的逻辑。
    /** 对于声明式事务的处理与编程式事务的处理，第一点区别在于事务属性上， 因为编程式的事务处理是不需要有事务属性的，
     * 第二点区别就是在TransactionManager上，CallbackPreferringPlatformTransactionManager实现PlatformTransactionManager接口，
      暴露出一个方法用于执行事务处理中的回调。所以，这两种方式都可以用作事务处理方式的判断。*/
    // 声明式事务处理
    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        // 4. 在目标方法执行前获取事务并收集事务信息。
        /** 事务信息与事务属性并不相同，也就是TransactionInfo与TransactionAttribute并不相同，
         * TransactionInfo中包含TransactionAttribute信息，但是，除了TransactionAttribute外还有其他事务信息，
         * 例如PlatformTransactionManager以及TransactionStatus相关信息。*/
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
        Object retVal = null;
        try {
            // This is an around advice: Invoke the next interceptor in the chain.
            // This will normally result in a target object being invoked.
            // 5. 执行目标方法。
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // target invocation exception
            // 6. 一旦出现异常，尝试异常处理。并不是所有异常，Spring都会将其回滚，默认只对RuntimeException回滚。
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // 7. 提交事务前的事务信息清除。
            cleanupTransactionInfo(txInfo);
        }
        // 8. 提交事务。
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
    // 编程式事务处理
    else {
        // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
        try {
            Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
                    new TransactionCallback<Object>() {
                        @Override
                        public Object doInTransaction(TransactionStatus status) {
                            TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
                            try {
                                return invocation.proceedWithInvocation();
                            }
                            catch (Throwable ex) {
                                if (txAttr.rollbackOn(ex)) {
                                    // A RuntimeException: will lead to a rollback.
                                    if (ex instanceof RuntimeException) {
                                        throw (RuntimeException) ex;
                                    }
                                    else {
                                        throw new ThrowableHolderException(ex);
                                    }
                                }
                                else {
                                    // A normal return value: will lead to a commit.
                                    return new ThrowableHolder(ex);
                                }
                            }
                            finally {
                                cleanupTransactionInfo(txInfo);
                            }
                        }
                    });

            // Check result: It might indicate a Throwable to rethrow.
            if (result instanceof ThrowableHolder) {
                throw ((ThrowableHolder) result).getThrowable();
            }
            else {
                return result;
            }
        }
        catch (ThrowableHolderException ex) {
            throw ex.getCause();
        }
    }
}
```
