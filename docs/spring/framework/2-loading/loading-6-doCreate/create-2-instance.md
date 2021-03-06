##创建Bean实例
AbstractAutowireCapableBeanFactory.createBeanInstance
```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // Make sure bean class is actually resolved at this point.
    //1. 解析Class
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    //2. 如果工厂方法不为空则使工厂方法初始化策略
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    //3. 判断是否已解析过构造函数参数
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            //一个类有多个构造函数，每个构造函数都有不同的参数，所以调用前需要先根据参数锁定构造函数或对应的工厂方法
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    
    //4. 如果已经解析析过则使用解析好的构造函数方法创建Bean
    if (resolved) {
        if (autowireNecessary) {
            //构造函数自动注入
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            //使用默认构造函数构造
            return instantiateBean(beanName, mbd);
        }
    }

    // Need to determine the constructor...
    //5. 根据参数解析构造函数决定使用哪一个构造函数
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        //构造函数自动注入
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // No special handling: simply use no-arg constructor.
    //使用默认构造函数构造
    return instantiateBean(beanName, mbd);
}
```

ConstructorResolver.autowireConstructor
```
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
        Constructor<?>[] chosenCtors, final Object[] explicitArgs) {

    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);

    Constructor<?> constructorToUse = null;
    ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null;

    //1. 构造函数参数的确定。
    //1.1 根据explicitArgs参数判断。
    /** 如果传人的参数explicitArgs 不为空，那就可以直接确定参数。
     * 因为explicitArgs参数是在调用Bean 的时候用户指定的，在BeanFactory 类中存在这样的方法：
     * Object getBean(String name, Object .. args) throws BeansException;
     * 在获取bean 的时候，用户不但可以指定bean的名称还可以指定bean所对应类的构造函数或者工厂方法的方法参数，
     * 主要用于静态工厂方法的调用，而这里是需要给定完全匹配的参数的，
     * 所以，便可以判断，如果传人参数explicitArgs 不为空，则可以确定构造函数参数就是它。 */
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    }
    else {
        //如果方法调用时未指定方法参数则从配置中解析
        Object[] argsToResolve = null;
        //1.2 从缓存中获取
        /* 除此之外，确定参数的办法如果之前已经分析过， 也就是说构造函数参数已经记录在缓存中，
         * 那么便可以直接拿来使用。而且，这里要提到的是，在缓存中缓存的可能是参数的最终类型也可能是参数的初始类型，
         * 例如：构造函数参数要求的是int类型，但是原始的参数值可能是String 类型的"1" ，那么即使在缓存中得到了参数，
         * 也需要经过类型转换器的过滤以确保参数类型与对应的构造函数参数类型完全对应。
        synchronized (mbd.constructorArgumentLock) {
            constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse != null && mbd.constructorArgumentsResolved) {
                // Found a cached constructor...
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        //解析参数类型，如给定方法的构造函数A(int, int）则通过此方法后就会把配置中的("1","1")转换为(1 , 1)。缓存中可能是原始值也可能是最终值
        if (argsToResolve != null) {
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
        }
    }

    //没有被缓存
    if (constructorToUse == null) {
        // Need to resolve the constructor.
        boolean autowiring = (chosenCtors != null ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
        ConstructorArgumentValues resolvedValues = null;

        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        }
        else {
            //1.3 提取配置文件中的配置的构造函数参数
            /** 如果不能根据传人的参数explicitArgs确定构造函数的参数也无法在缓存中得到相关信息，那么只能开始新一轮的分析了。
             * 分析从获取配置文件中配置的构造函数信息开始，经过之前的分析，
             * 我们知道，Spring中配置文件中的信息经过转换都会通过BeanDefinition实例承载，
             * 也就是参数mbd中包含，那么可以通过调用mbd.getConstructorArgumentValues()来获取配置的构造函数信息。
             * 有了配置中的信息便可以获取对应的参数值信息了，获取参数值的信息包括直接指定值，
             * 如：直接指定构造函数中某个值为原始类型String 类型，或者是一个对其他bean 的引用，
             * 而这一处理委托给resoIveConstructorArguments方法，并返回能解析到的参数的个数。*/
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            //用于承载解析后的构造函数参数的值
            resolvedValues = new ConstructorArgumentValues();
            //能解析到的参数个数
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }

        // Take specified constructors, if any.
        //2. 构造函数的确定。
        /** 经过了第一步后已经确定了构造函数的参数，接下来的任务就是根据构造函数参数在所有构造函数中锁定对应的构造函数，
         * 而匹配的方法就是根据参数个数匹配，所以在匹配之前需要先对构造函数按照public构造函数优先参数数量降序、非public构造函数参数数量降序。
         * 这样可以在遍历的情况下迅速判断排在后面的构造函数参数个数是否符合条件。
         * 由于在配置文件中并不是唯一限制使用参数位置索引的方式去创建，同样还支持指定参数名称进行设定参数值的情况，
         * 如＜constructor- arg name＝"id"，那么这种情况就需要首先确定构造函数中的参数名称。
         * 获取参数名称可以有两种方式，一种是通过注解的方式直接获取，另一种就是使用Spring中提供的工具类ParameterNameDiscoverer来获取。
         * 构造函数、参数名称、参数类型、参数值都确定后就可以锁定构造函数以及转换对应的参数类型了。
        Constructor<?>[] candidates = chosenCtors;
        if (candidates == null) {
            Class<?> beanClass = mbd.getBeanClass();
            try {
                candidates = (mbd.isNonPublicAccessAllowed() ?
                        beanClass.getDeclaredConstructors() : beanClass.getConstructors());
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                        "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
            }
        }
        // 排序给定的构造器方法 按照参数降序排序
        AutowireUtils.sortConstructors(candidates);
        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Constructor<?>> ambiguousConstructors = null;
        LinkedList<UnsatisfiedDependencyException> causes = null;

        for (Constructor<?> candidate : candidates) {
            Class<?>[] paramTypes = candidate.getParameterTypes();

            if (constructorToUse != null && argsToUse.length > paramTypes.length) {
                // Already found greedy constructor that can be satisfied ->
                // do not look any further, there are only less greedy constructors left.
                // 如果已经找到选用的构造函数或者需要的参数个数小于当前的构造函数参数个数则终止，因为已经按照参数个数降序排列
                break;
            }
            if (paramTypes.length < minNrOfArgs) {
                //参数个数不相等
                continue;
            }

            ArgumentsHolder argsHolder;
            if (resolvedValues != null) {
                //根据值构造对应参数类型的参数
                try {
                    // 注释上获取参数名称
                    String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
                    if (paramNames == null) {
                        // 获取参数名称探索器
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        if (pnd != null) {
                            // 获取指定构造器的参数名称
                            paramNames = pnd.getParameterNames(candidate);
                        }
                    }
                    //根据名称和数据类型创建参数持有者
                    //3. 根据确定的构造函数转换对应的参数类型。主要是使用Spring中提供的类型转换器或者用户提供的自定义类型转换器进行转换。
                    argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                            getUserDeclaredConstructor(candidate), autowiring);
                }
                catch (UnsatisfiedDependencyException ex) {
                    if (this.beanFactory.logger.isTraceEnabled()) {
                        this.beanFactory.logger.trace(
                                "Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
                    }
                    // Swallow and try next constructor.
                    if (causes == null) {
                        causes = new LinkedList<UnsatisfiedDependencyException>();
                    }
                    causes.add(ex);
                    continue;
                }
            }
            else {
                // Explicit arguments given -> arguments length must match exactly.
                if (paramTypes.length != explicitArgs.length) {
                    continue;
                }
                //构造函数没有参数的情况下
                argsHolder = new ArgumentsHolder(explicitArgs);
            }

            //4. 构造函数不确定性的验证。
            /** 当然，有时候即使构造函数、参数名称、参数类型、参数值都确定后也不一定会直接锁定构造函数，
             * 不同构造函数的参数为父子关系，所以Spring 在最后又做了一次验证。*/
            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                    argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
            // Choose this constructor if it represents the closest match.
            // 如果它代表着当前最接近的匹配则选择作为构造函数
            if (typeDiffWeight < minTypeDiffWeight) {
                constructorToUse = candidate;
                argsHolderToUse = argsHolder;
                argsToUse = argsHolder.arguments;
                minTypeDiffWeight = typeDiffWeight;
                ambiguousConstructors = null;
            }
            else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                if (ambiguousConstructors == null) {
                    ambiguousConstructors = new LinkedHashSet<Constructor<?>>();
                    ambiguousConstructors.add(constructorToUse);
                }
                ambiguousConstructors.add(candidate);
            }
        }

        if (constructorToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Could not resolve matching constructor " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
        }
        else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous constructor matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousConstructors);
        }

        if (explicitArgs == null) {
            //将解析的构造函数加入缓存
            argsHolderToUse.storeCache(mbd, constructorToUse);
        }
    }

    try {
        Object beanInstance;
        //5. 根据实例化策略以及得到的构造函数及构造函数参数实例化Bean。
        if (System.getSecurityManager() != null) {
            final Constructor<?> ctorToUse = constructorToUse;
            final Object[] argumentsToUse = argsToUse;
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return beanFactory.getInstantiationStrategy().instantiate(
                            mbd, beanName, beanFactory, ctorToUse, argumentsToUse);
                }
            }, beanFactory.getAccessControlContext());
        }
        else {
            beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
                    mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
        }
        //将构建的实例加入BeanWrapper
        bw.setBeanInstance(beanInstance);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean instantiation via constructor failed", ex);
    }
}
```

AbstractAutowireCapableBeanFactory.instantiateBean
```
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
```

SimpleInstantiationStrategy.instantiate
```
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    //如果有需要在覆盖或者动态替换的方法则当然需要使用cglib进行动态代理，否则直接反射就可以了
    if (bd.getMethodOverrides().isEmpty()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                            @Override
                            public Constructor<?> run() throws Exception {
                                return clazz.getDeclaredConstructor((Class[]) null);
                            }
                        });
                    }
                    else {
                        constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
                    }
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Throwable ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // Must generate CGLIB subclass.
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```
程序中，首先判断如果beanDefinition.getMethodOverrides为空也就是用户没有使用replace或者lookup的配置方法，
那么直接使用反射的方式，简单快捷，但是如果使用了这两个特性，在直接使用反射的方式创建实例就不妥了，
因为需要将这两个配置提供的功能切入进去，所以就必须要使用动态代理的方式将包含两个特性所对应的逻辑的拦截增强器设置进去，
这样才可以保证在调用方法的时候会被相应的拦截器增强，返回值为包含拦截器的代理实例。

