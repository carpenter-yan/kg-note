## Conditional机制实现

### 1. Conditional使用
@Conditional根据满足的某一个特定条件建一个特定bean。比方说，当某一JAR包在一个类路径下的时候，会自动配置一个或多个bean；
或者只有某个bean被创建后才会创建另外bean。总的来说，就是根据特定条件来控制bean的创建行为，这样我们可以利用这个特性进行一些自动的配置。
当然，Conditional注解有非常多的使用方式，我们仅仅通过ConditionalOnProperty来深入探讨它的运行机制
@Configuration
@ComponentScan({"com.spring study.module"})
@ConditionalOnProperty(prefix="study", name="enable", havingValue="true")
public class HelloServiceAutoConfiguration{
}

### 2. Conditional原理
搜索ConditionalOnProperty.class发现其使用类为OnPropertyCondition.java且只有一个public方法
```
public ConditionOutcome getMatchOutcome(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
    // 1. 扫描出ConditionalOnProperty的注解信息
    List<AnnotationAttributes> allAnnotationAttributes = annotationAttributesFromMultiValueMap(
            metadata.getAllAnnotationAttributes(
                    ConditionalOnProperty.class.getName()));
    List<ConditionMessage> noMatch = new ArrayList<>();
    List<ConditionMessage> match = new ArrayList<>();
    for (AnnotationAttributes annotationAttributes : allAnnotationAttributes) {
        // 2. 根据属性进行判断
        ConditionOutcome outcome = determineOutcome(annotationAttributes,
                context.getEnvironment());
        (outcome.isMatch() ? match : noMatch).add(outcome.getConditionMessage());
    }
    if (!noMatch.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage.of(noMatch));
    }
    return ConditionOutcome.match(ConditionMessage.of(match));
}

private ConditionOutcome determineOutcome(AnnotationAttributes annotationAttributes,
        PropertyResolver resolver) {
    Spec spec = new Spec(annotationAttributes);
    List<String> missingProperties = new ArrayList<>();
    List<String> nonMatchingProperties = new ArrayList<>();
    spec.collectProperties(resolver, missingProperties, nonMatchingProperties);
    // 这个逻辑表明，不匹配有两种情况missingProperties对应属性缺失情况；nonMatchingProperties对应不匹配情况
    if (!missingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(
                ConditionMessage.forCondition(ConditionalOnProperty.class, spec)
                        .didNotFind("property", "properties")
                        .items(Style.QUOTE, missingProperties));
    }
    if (!nonMatchingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(
                ConditionMessage.forCondition(ConditionalOnProperty.class, spec)
                        .found("different value in property",
                                "different value in properties")
                        .items(Style.QUOTE, nonMatchingProperties));
    }
    return ConditionOutcome.match(ConditionMessage
            .forCondition(ConditionalOnProperty.class, spec).because("matched"));
}

private void collectProperties(PropertyResolver resolver, List<String> missing,
        List<String> nonMatching) {
    for (String name : this.names) {
        String key = this.prefix + name;
        if (resolver.containsProperty(key)) {
            if (!isMatch(resolver.getProperty(key), this.havingValue)) {
                nonMatching.add(name);
            }
        }
        else {
            if (!this.matchIfMissing) {
                missing.add(name);
            }
        }
    }
}
```

### 3. 调用切入点
那么现在的问题OnPropertyCondition.getMatchOutcome方法是谁去调用的呢？
或者说这个类是如何与spring整合在一起呢？它又是怎么样影响bean的加载逻辑的呢？
参考<boot-2-starter.md>中factories调用时序
ConfigurationClassParser.processImports
    ->ConfigurationClassParser.processConfigurationClass
```
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    /** 代码的第一行就是整个Conditional逻辑生效的切人点，如果验证不通过会直接忽略掉后面的解析逻辑，
     * 那么这个类的属性以及componentScan之类的配也自然不得到解析了。这个方法会拉取所有的condition属性，
     * onConditionProperty就是这里拉取的 */
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            this.knownSuperclasses.values().removeIf(configClass::equals);
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```

ConditionEvaluator.java
```
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
    if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
        return false;
    }

    if (phase == null) {
        if (metadata instanceof AnnotationMetadata &&
                ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
            return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
        }
        return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
    }

    List<Condition> conditions = new ArrayList<>();
    // 1. condition的获取
    for (String[] conditionClasses : getConditionClasses(metadata)) {
        for (String conditionClass : conditionClasses) {
            Condition condition = getCondition(conditionClass, this.context.getClassLoader());
            conditions.add(condition);
        }
    }

    AnnotationAwareOrderComparator.sort(conditions);

    /** 2. condition 的运行匹配。通过代码condition.matches(this.context, metadata)调用，因为我们的配置为
     * @ConditionalOnProperty(prefix="study", name="enabled", havingValue="true") 
     * 所以此时condition对应的运行态类为OnPropertyCondition */
    for (Condition condition : conditions) {
        ConfigurationPhase requiredPhase = null;
        if (condition instanceof ConfigurationCondition) {
            requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
        }
        if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
            return true;
        }
    }

    return false;
}
```

SpringBootCondition.java
```
public final boolean matches(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
    String classOrMethodName = getClassOrMethodName(metadata);
    try {
        // 1. 调用OnPropertyCondition.getMatchOutcome
        ConditionOutcome outcome = getMatchOutcome(context, metadata);
        logOutcome(classOrMethodName, outcome);
        recordEvaluation(context, classOrMethodName, outcome);
        // 2. 返回是否匹配
        return outcome.isMatch();
    }
    catch (NoClassDefFoundError ex) {
        throw new IllegalStateException(
                "Could not evaluate condition on " + classOrMethodName + " due to "
                        + ex.getMessage() + " not "
                        + "found. Make sure your own configuration does not rely on "
                        + "that class. This can also happen if you are "
                        + "@ComponentScanning a springframework package (e.g. if you "
                        + "put a @ComponentScan in the default package by mistake)",
                ex);
    }
    catch (RuntimeException ex) {
        throw new IllegalStateException(
                "Error processing condition on " + getName(metadata), ex);
    }
}
```