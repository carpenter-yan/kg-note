## 关键对象总结与回顾

### 3.1 SqlSource
SqlSource是XML文件或者注解方法中映射语句的实现时表示，通过SqlSourceBuilder.parse()方法创建，
SqlSourceBuilder中符号解析器将mybatis中的查询参数#{}转换为?，并记录了参数的顺序。
它只有一个方法getBoundSql用于获取映射语句对象的各个组成部分

根据SQL语句的类型不同，mybatis提供了多种SqlSource的具体实现
StaticSqlSource：最终静态SQL语句的封装,其他类型的SqlSource最终都委托给StaticSqlSource
RawSqlSource：原始静态SQL语句的封装,在加载时就已经确定了SQL语句,没有动态标签和${}SQL拼接,比动态SQL语句要快,因为不需要运行时解析SQL节点。
DynamicSqlSource：动态SQL语句的封装，在运行时需要根据参数处理标签或者${}SQL拼接之后才能生成最后要执行的静态SQL语句。
ProviderSqlSource：当SQL语句通过指定的类和方法获取时(使用@XXXProvider注解)，需要使用本类，它会通过反射调用相应的方法得到SQL语句。

### 3.2 SqlNode
SqlNode接口主要用来处理CRUD节点下的各类动态标签，对每个动态标签，mybatis都提供了对应的SqlNode实现，
这些动态标签可以相互嵌套且实现上采用单向链表进行应用，这样后面如果需要增加其他动态标签，就只需要新增对应的SqlNode实现就能支持。
当前版本的SqlNode有下列实现
ChooseSqlNode
ForEachSqlNode
IfSqlNode
StaticTextSqlNode: 静态文本节点不做任何处理，直接将本文本节点的内容追加到已经解析了的SQL文本的后面
TextSqlNode: TextSqlNode主要是用来将${}转换为实际的参数值，并返回拼接后的SQL语句，为了防止SQL注入，可以通过标签来创建OGNL上下文变量
VarDeclSqlNode
TrimSqlNode
SetSqlNode
WhereSqlNode

### 3.3 BaseBuilder
从整个设计角度来说，BaseBuilder的目的是为了统一解析的使用，但在实现上却出入较大。
首先，BaseBuilder是所有解析类的MapperBuilderAssistant、XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuilder等的父类。
BaseBuilder中提供类型处理器、JDBC类型、结果集类型、别名等的解析，因为在mybatis配置文件、mapper文件解析、SQL映射语句解析、
基于注解的mapper文件解析过程中，都会频繁的遇到类型处理相关的解析。但是BaseBuilder也没有定义需要子类实现的负责解析的抽象接口

### 3.4 AdditionalParameter
额外参数主要是维护一些在加载时无法确定的参数，比如标签中的参数在加载时就无法尽最大努力确定，
必须通过运行时执行DynamicSqlSource.getBoundSql()中的SqlNode.apply()才能确定真正要执行的SQL语句，以及额外参数。

### 3.5 TypeHandler
当MyBatis将一个Java对象作为输入/输出参数执行CRUD语句操作时，它会创建一个PreparedStatement对象，并且调用setXXX()为占位符设置相应的参数值。
XXX可以是Int，String，Date等Java内置类型，或用户自定义的类型。在实现上，MyBatis通过使用类型处理器（type handler）来确定XXX是具体什么类型。
MyBatis对于下列类型使用内建的类型处理器：所有的基本数据类型、基本类型的包裹类型、byte[] 、java.util.Date、java.sql.Date、
java,sql.Time、java.sql.Timestamp、java 枚举类型等。对于用户自定义的类型，我们可以创建一个自定义的类型处理器。
要创建自定义类型处理器，只要实现TypeHandler接口即可。

虽然我们可以直接实现TypeHandler接口，但是在实践中，我们一般选择继承BaseTypeHandler，BaseTypeHandler为TypeHandler提供了部分骨架代码，
使得用户使用方便，几乎所有mybatis内置类型处理器都继承于BaseTypeHandler。

### 3.6 对象包装器工厂ObjectWrapperFactory
ObjectWrapperFactory是一个对象包装器工厂,用于对返回的结果对象进行二次处理
典型的下划线转驼峰,我们就可以使用ObjectWrapperFactory来统一处理(在实际中,我们一般不会这么做,而是通过设置mapUnderscoreToCamelCase来实现)
通过实现这个接口，可以判断当object是特定类型时，返回true，然后在下面的getWrapperFor中返回一个可以处理key为驼峰的ObjectWrapper 实现类即可。
ObjectWrapper类可以说是对象反射信息的facade模式
当然，我们不需要从头实现ObjectWrapper接口,可以选择继承BeanWrapper或者MapWrapper。
比如对于Map类型，我们可以继承MapWrapper，让参数useCamelCaseMapping起作用
mybatis提供了一个什么都不做的默认实现DefaultObjectWrapperFactory

### 3.7 MetaObject
MetaObject是一个对象包装器，其性质上有点类似ASF提供的commons类库，其中包装了对象的元数据信息，对象本身，对象反射工厂，对象包装器工厂等。
使得根据OGNL表达式设置或者获取对象的属性更为便利，也可以更加方便的判断对象中是否包含指定属性、指定属性是否具有getter、setter等。
主要的功能是通过其ObjectWrapper类型的属性完成的，它包装了操作对象元数据以及对象本身的主要接口，操作标准对象的实现是BeanWrapper。
BeanWrapper类型有个MetaClass类型的属性，MetaClass中有个Reflector属性，其中包含了可读、可写的属性、方法以及构造器信息。

### 3.8 对象工厂ObjectFactory
MyBatis 每次创建结果对象的新实例时，都会使用一个对象工厂（ObjectFactory）实例来完成。 
默认的对象工厂DefaultObjectFactory仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。
如果想覆盖对象工厂的默认行为比如给某些属性设置默认值(有些时候直接修改对象不可行，或者由于不是自己拥有的代码或者改动太大)，
则可以通过创建自己的对象工厂来实现。

要自定义对象工厂类，我们可以实现ObjectFactory这个接口，但是这样我们就需要自己去实现一些在DefaultObjectFactory已经实现好了的东西，
所以也可以继承这个DefaultObjectFactory类，这样可以使得实现起来更为简单。

### 3.9 MappedStatement
mapper文件或者mapper接口中每个映射语句都对应一个MappedStatement实例，它包含了所有运行时需要的信息比如结果映射、参数映射、是否刷新缓存等。
唯一值得注意的是resultMaps被设计为只读,这样应用可以查看但是不能修改。

### 3.10 ParameterMapping
每个参数映射<>标签都被创建为一个ParameterMapping实例，其中包含和结果映射类似的信息

### 3.11 KeyGenerator
```
public interface KeyGenerator {
  // before key generator 主要用于oracle等使用序列机制的ID生成方式
  void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);
  // after key generator 主要用于mysql等使用自增机制的ID生成方式
  void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

### 3.12 各种Registry
mybatis将类型处理器，类型别名，mapper定义，语言驱动器等各种信息包装在Registry中维护
```
public class Configuration {
  ...
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
  ...
}
```

### 3.13 LanguageDriver
从3.2版本开始，mybatis提供了LanguageDriver接口，我们可以使用该接口自定义SQL的解析方式。
```
public interface LanguageDriver {
  /**
   * 创建一个ParameterHandler对象，用于将实际参数赋值到JDBC语句中
   */
  ParameterHandler createParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql);
  /**
   * 将XML中读入的语句解析并返回一个sqlSource对象
   */
  SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType);
  /**
   * 将注解中读入的语句解析并返回一个sqlSource对象
   */
  SqlSource createSqlSource(Configuration configuration, String script, Class<?> parameterType);

}
```

### 3.14 ResultMap
ResultMap类维护了每个标签中的详细信息，比如id映射、构造器映射、属性映射以及完整的映射列表、是否有嵌套的resultMap、是否有鉴别器、是否有嵌套查询
ResultMap除了作为一个ResultMap的数据结构表示外，本身并没有提供额外的功能。

### 3.15 ResultMapping
ResultMapping代表了映射

### 3.16 Discriminator
每个鉴别器节点都表示为一个Discriminator

### 3.17 Configuration
Configuration是mybatis所有配置以及mapper文件的元数据容器。
无论是解析mapper文件还是运行时执行SQL语句，都需要依赖与mybatis的环境和配置信息，比如databaseId、类型别名等。
mybatis实现将所有这些信息封装到Configuration中并提供了一系列便利的接口方便各主要的调用方使用，
这样就避免了各种配置和元数据信息到处散落的凌乱。

### 3.18 ErrorContext
ErrorContext定义了一个mybatis内部统一的日志规范，记录了错误信息、发生错误涉及的资源文件、对象、逻辑过程、SQL语句以及出错原因，但是它不会影响运行

### 3.19 BoundSql
```
 /**
  * SqlSource中包含的SQL处理动态内容之后的实际SQL语句，SQL中会包含?占位符，也就是最终给JDBC的SQL语句，以及他们的参数信息
  */
public class BoundSql {
  // sql文本
  private final String sql;
  // 静态参数说明
  private final List<ParameterMapping> parameterMappings;
  // 运行时参数对象
  private final Object parameterObject;
  // 额外参数，也就是for loops、bind生成的
  private final Map<String, Object> additionalParameters;
  // 额外参数的facade模式包装
  private final MetaObject metaParameters;
  ...
}
```