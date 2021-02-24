##创建BeanDefinition
BeanDefinitionReaderUtils.createBeanDefinition
```
public static AbstractBeanDefinition createBeanDefinition(
        String parentName, String className, ClassLoader classLoader) throws ClassNotFoundException {

    GenericBeanDefinition bd = new GenericBeanDefinition();
    bd.setParentName(parentName);
    if (className != null) {
        //如果classLoader不为空，则使用以传人的classLoader 同－虚拟机加载类对象，否则只是记录className
        if (classLoader != null) {
            bd.setBeanClass(ClassUtils.forName(className, classLoader));
        }
        else {
            bd.setBeanClassName(className);
        }
    }
    return bd;
}
```
BeanDefinition是一个接口，在Spring中存在三种实现：RootBeanDefinition、ChildBeanDefinition以及GenericBeanDefinition。三种实现均继承了AbstractBeanDefinition ，其中BeanDefinition是配置文件＜bean＞元素标签在容器中的内部表示形式。<bean>元素标签拥有class、scope、lazy-init等配置属性，BeanDefinition则提供了相应的beanClass、scope、lazyInit属性，BeanDefinition和<bean>中的属性是－一对应的。其中RootBeanDefinition是最常用的实现类，它对应一般性的<bean<元素标签，GenericBeanDefinition是自2.5版本以后新加入的bean文件配置属性定义类，是一站式服务类。在配置文件中可以定义父<bean>和子<bean>，父<bean>用RootBeanDefinition表示，而子<bean>用ChildBeanDefinition表示，而没有父<bean>的<bean>就使用RootBeanDefinition 表示。
AbstractBeanDefinition对两者共同的类信息进行抽象。Spring通过BeanDefinition将配置文件中的<bean>配置信息转换为容器的内部表示，并将这些BeanDefinition注册到BeanDefinitionRegistry中。Spring容器的BeanDefinitionRegistry就像是Spring配置信息的内存数据库，主要是以map的形式保存，后续操作直接从BeanDefinitionRegistry中读取配置信息。

### AbstractBeanDefinition
```
private volatile Object beanClass;

//bean的作用范围， 对应bean属性scope
private String scope = SCOPE_DEFAULT;

//是否是抽象，对应bean属性abstract
private boolean abstractFlag = false;

//是否延迟加载，对应bean属性lazy-init
private boolean lazyInit = false;

//自动注入模式，对应bean属性autowire
private int autowireMode = AUTOWIRE_NO;

//依赖检查，3.0版本后弃用
private int dependencyCheck = DEPENDENCY_CHECK_NONE;

//表示一个bean的实例化依赖另外一个bean先实例化，对应属性depend-on
private String[] dependsOn;

//对应属性autowire-candidate。属性为false时，容器在查询自动装配对象时将不考虑该bean。但可以通过配置手动注入其它bean。
private boolean autowireCandidate = true;

//对应属性primary。出现多个bean候选时作为首选者。
private boolean primary = false;

//对应子元素qualifier。
private final Map<String, AutowireCandidateQualifier> qualifiers =
        new LinkedHashMap<String, AutowireCandidateQualifier>(0);

//是否允许范围非公开的构造器和方法，程序设置
private boolean nonPublicAccessAllowed = true;

//是否宽松模式解析构造器，程序设置
private boolean lenientConstructorResolution = true;

//对应bean属性factory-bean
private String factoryBeanName;

//对应bean属性factory-method
private String factoryMethodName;

//构造函数注入属性constructor-arg的子元素
private ConstructorArgumentValues constructorArgumentValues;

//普通属性集合
private MutablePropertyValues propertyValues;

//方法重新集合对应lookup-method,replaced-method元素
private MethodOverrides methodOverrides = new MethodOverrides();

//初始化方法对应bean属性init-method
private String initMethodName;

//销毁方法对应bean属性destroy-method
private String destroyMethodName;

//是否执行init-method，程序设置
private boolean enforceInitMethod = true;

//是否执行destroy-method，程序设置
private boolean enforceDestroyMethod = true;

//是否是用户定义的而不是程序本身定义的，创建AOP时为true。程序设置
private boolean synthetic = false;

//定义这个bean的应用,APPLICATION：用户，INFRASTRUCTURE，完全内部使用，与用户无关，SUPPORT某些复杂配置的一部分。程序设置
private int role = BeanDefinition.ROLE_APPLICATION;

//bean的描述
private String description;

//bean定义的资源
private Resource resource;
```



###辅助类
AbstractBeanDefinition
RootBeanDefinition
ChildBeanDefinition
GenericBeanDefinition