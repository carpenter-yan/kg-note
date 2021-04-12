## 属性自动化配置实现

@ConditionalOnProperty(prefix="study", name="enable", havingValue="true")
SpringBoot读取配置拼装成study.enabled并作为key，然后尝试使用PropertyResolver来验证对应的属性是否存在，
如果不存在则验证不通过，自然也就不会继续后面的解析流程，因为PropertyResolver中包含了所有的配属性信息。
那么，PropertyResolver又是如何被初始化的呢。同样，这功能并不仅仅供Spring内部使用，
在现在的Spring中我们也可以通过Value注解接将属性赋值给变量。这两个问题都涉及Spring属性处理逻辑。
@Value("${study.testStr"})
private String testStr;

通过搜Value.class进行定位到QualifierAnnotationAutowireCandidateResolver.java
```
private Class<? extends Annotation> valueAnnotationType = Value.class;

protected Object findValue(Annotation[] annotationsToSearch) {
    if (annotationsToSearch.length > 0) {   // qualifier annotations have to be local
        AnnotationAttributes attr = AnnotatedElementUtils.getMergedAnnotationAttributes(
                AnnotatedElementUtils.forAnnotations(annotationsToSearch), this.valueAnnotationType);
        if (attr != null) {
            return extractValue(attr);
        }
    }
    return null;
}

protected Object extractValue(AnnotationAttributes attr) {
    Object value = attr.get(AnnotationUtils.VALUE);
    if (value == null) {
        throw new IllegalStateException("Value annotation must have a value attribute");
    }
    return value;
}
```
表达式对应的值是在哪里被替换的？
表达式替换后的值又是如何与原有的Bean整合的？
AbstractBeanFactory.java
```
public String resolveEmbeddedValue(@Nullable String value) {
    if (value == null) {
        return null;
    }
    String result = value;
    for (StringValueResolver resolver : this.embeddedValueResolvers) {
        result = resolver.resolveStringValue(result);
        if (result == null) {
            return null;
        }
    }
    return result;
}
```
对于属性的解析已经委托给了StringValueResolver对应的实现类，接下来我们就要分析下这个StringValueResolver是如何初始化的？
StringValueResolver功能实现依赖spring的切人点是PropertySourcesPlaceholderConfigurer。从其类结构看，它的关键是实现了
BeanFactoryPostProcessor接口，从而利用实现对外扩展函数postProcessBeanFactory来进行对spring的扩展。
->PropertySourcesPlaceholderConfigurer.postProcessBeanFactory
    ->MutablePropertySources.new
    ->PropertySourcesPropertyResolver.new
    ->PropertySourcesPlaceholderConfigurer.processProperties
        ->StringValueResolver.new
        ->PlaceholderConfigurerSupport.doProcessProperties
            ->ConfigurableBeanFactory.addEmbeddedValueResolver

1. 初始化MutablePropertySources
    首先会通过this.environment来初始化MutablePropertySources，environment是Spring属性加载的基础，里面包含了已经加载的各个属性，
    而之所以使用MutablePropertySources封装，是因为MutablePropertySources还能实现单独加载自定义的额外属性的功能。

2. 初始化PropertySourcesPropertyResolver
   使用PropertySourcesPropertyResolver对MutablePropertySources的操作进行进一步封装， 使得操作多个文件对外部不感知。
   当然PropertySourcesPropertyResolver还提供一个重要的功能就是变量的解析。${key:default}

3. StringValueResolver初始化
PropertySourcesPlaceholderConfigurer.java
```
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
        final ConfigurablePropertyResolver propertyResolver) throws BeansException {

    propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
    propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
    propertyResolver.setValueSeparator(this.valueSeparator);

    /** StringValueResolver存在的目的主要是对解析逻辑的进一步封装，例如通过变ignoreUnresolvablePlaceholders控制是否对变量做解析
     * resolvePlaceholders表示如果变无法解析则忽略，resolveRequiredPlaceholders表示如果变量无法解析则抛异常*/
    StringValueResolver valueResolver = strVal -> {
        String resolved = (this.ignoreUnresolvablePlaceholders ?
                propertyResolver.resolvePlaceholders(strVal) :
                propertyResolver.resolveRequiredPlaceholders(strVal));
        if (this.trimValues) {
            resolved = resolved.trim();
        }
        return (resolved.equals(this.nullValue) ? null : resolved);
    };

    doProcessProperties(beanFactoryToProcess, valueResolver);
}
```

4. StringValueResolver注册
   最后将StringValueResolver实例注册到单例ConfigurableListableBeanFactory中，也就是在真正解析变量时使用StringValueResolver实例
   
### environment实现路径
->ConfigFileApplicationListener.onApplicationEnvironmentPreparedEvent
    ->ConfigFileApplicationListener.postProcessEnvironment
        ->ConfigFileApplicationListener.addPropertySources
            ->ConfigFileApplicationListener.Loader.load()
                ->ConfigFileApplicationListener.addProfileToEnvironment
                    ->ConfigurableEnvironment.addActiveProfile
environment初始化过程并不是之前通用的PostProcessor类型的扩展口上做扩展，通过ConfigFileApplicationListener监听机制完成。

ConfigFileApplicationListener.Loader.load()
```
public void load() {
    this.profiles = new LinkedList<>();
    this.processedProfiles = new LinkedList<>();
    this.activatedProfiles = false;
    this.loaded = new LinkedHashMap<>();
    /** 通过profile标记不同的环境，可以通过设置spring.profiles.active和spring.profiles.default。
     * 如果设置了active，default便失去了作用。如果两个都没有设置，那么带有profiles的bean都不会生成 */
    initializeProfiles();
    while (!this.profiles.isEmpty()) {
        Profile profile = this.profiles.poll();
        if (profile != null && !profile.isDefaultProfile()) {
            addProfileToEnvironment(profile.getName());
        }
        load(profile, this::getPositiveProfileFilter,
                addToLoaded(MutablePropertySources::addLast, false));
        this.processedProfiles.add(profile);
    }
    resetEnvironmentProfiles(this.processedProfiles);
    load(null, this::getNegativeProfileFilter,
            addToLoaded(MutablePropertySources::addFirst, true));
    addLoadedPropertySources();
}

private void load(Profile profile, DocumentFilterFactory filterFactory,
        DocumentConsumer consumer) {
    //SpringBoot默认从4个位置查找application.properties文件就是从getSearchLocations()方法返回：
    // 1. 当前目录下的/config目录
    // 2. 当前目录
    // 3. 类路径下的/config目录
    // 4. 类路径下根目录
    getSearchLocations().forEach((location) -> {
        boolean isFolder = location.endsWith("/");
        Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
        names.forEach(
                (name) -> load(location, name, profile, filterFactory, consumer));
    });
}
```
Spring中的profile机制就是在此实现