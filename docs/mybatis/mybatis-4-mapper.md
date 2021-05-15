前面说过mybatis mapper文件的加载主要有两大类，通过package加载和明确指定的方式。
对于简单语句来说，使用注解代码会更加清晰，然而Java注解对于复杂语句比如同时包含了构造器、鉴别器、resultMap来说就会非常混乱，应该限制使用，
此时应该使用XML文件，因为注解不像XML/Gradle一样能够很好的表示嵌套关系。mybatis完整的注解列表可参考org.apache.ibatis.annotations包。

### 注解示例
```
@Select("select *from User where id=#{id} and userName like #{name}")
public User retrieveUserByIdAndName(@Param("id")int id,@Param("name")String names);

@Insert("INSERT INTO user(userName,userAge,userAddress) VALUES(#{userName},#{userAge},#{userAddress})")
public void addNewUser(User user);

@Insert("insert into table3 (id, name) values(#{nameId}, #{name})")
@SelectKey(statement="call next value for TestSequence", keyProperty="nameId", before=true, resultType=int.class)
int insertTable3(Name name);

@Results(id = "userResult", value = {
  @Result(property = "id", column = "uid", id = true),
  @Result(property = "firstName", column = "first_name"),
  @Result(property = "lastName", column = "last_name")
})
@TypeDiscriminator(column = "type",
   cases={   
       @Case(value="1",type=RegisterEmployee.class,results={
          @Result(property="salay")
       }),
       @Case(value="2",type=TimeEmployee.class,results={
          @Result(property="time")
       })
   }
)
@Select("select * from users where id = #{id}")
User getUserById(Integer id);

@Results(id = "companyResults")
@ConstructorArgs({
  @Arg(property = "id", column = "cid", id = true),
  @Arg(property = "name", column = "name")
})
@Select("select * from company where id = #{id}")
Company getCompanyById(Integer id);

@ResultMap(id = "xmlUserResults")
@SelectProvider(type = UserSqlBuilder.class, method = "buildGetUsersByName")
List<User> getUsersByName(String name);

// 注：建议尽可能避免使用SqlBuild的模式生成的,如果因为功能需要必须动态生成SQL的话，也是直接写SQL拼接返回，
而不是一堆类似SELECT()、FROM()的函数调用，这只会让维护成为噩梦，这思路的设计者不是知道怎么想的, 
此处仅用于演示XXXProvider功能，但是XXXProvider模式本身的设计在关键时候还是比较清晰的。
class UserSqlBuilder {
  public String buildGetUsersByName(final String name) {
    return new SQL(){{
      SELECT("*");
      FROM("users");
      if (name != null) {
        WHERE("name like #{value} || '%'");
      }
      ORDER_BY("id");
    }}.toString();
  }
}
```

### xml配置示例
```
<?xml version="1.0" encoding="UTF-8" ?>   
<!DOCTYPE mapper   
PUBLIC "-//ibatis.apache.org//DTD Mapper 3.0//EN"  
"http://ibatis.apache.org/dtd/ibatis-3-mapper.dtd"> 

<!-- mapper 为根元素节点， 一个namespace对应一个dao -->
<mapper namespace="com.dy.dao.UserDao">
    <insert id="insertUser" parameterType="com.demo.User" flushCache="true" statementType="PREPARED" 
      keyProperty="" keyColumn="" useGeneratedKeys="false" timeout="20">

      <!-- 1. id （必须配置）
        id是命名空间中的唯一标识符，可被用来代表这条语句。 一个命名空间（namespace）对应一个dao接口, 
        这个id也应该对应dao里面的某个方法（相当于方法的实现），因此id 应该与方法名一致 -->      
      
      <!-- 2. parameterType （可选配置, 默认为mybatis自动选择处理）
        将要传入语句的参数的完全限定类名或别名， 如果不配置，mybatis会通过ParameterHandler 根据参数类型默认选择合适的typeHandler进行处理
        parameterType 主要指定参数类型，可以是int, short, long, string等类型，也可以是复杂类型（如对象） -->
      
      <!-- 3. flushCache （可选配置，默认配置为true）
        将其设置为 true，任何时候只要语句被调用，都会导致本地缓存和二级缓存都会被清空，默认值：true（对应插入、更新和删除语句） -->
      
      <!-- 4. statementType （可选配置，默认配置为PREPARED）
        STATEMENT，PREPARED 或 CALLABLE 的一个。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 -->
      
      <!-- 5. keyProperty (可选配置， 默认为unset)
        （仅对 insert 和 update 有用）唯一标记一个属性，MyBatis 会通过 getGeneratedKeys 的返回值或者通过 insert 语句的 selectKey 子元素设置它的键值，默认：unset。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。 -->
      
      <!-- 6. keyColumn     (可选配置)
        （仅对 insert 和 update 有用）通过生成的键值设置表中的列名，这个设置仅在某些数据库（像 PostgreSQL）是必须的，当主键列不是表中的第一列的时候需要设置。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。 -->
      
      <!-- 7. useGeneratedKeys (可选配置， 默认为false)
        （仅对 insert 和 update 有用）这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系数据库管理系统的自动递增字段），默认值：false。  -->
      
      <!-- 8. timeout  (可选配置， 默认为unset, 依赖驱动)
        这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为 unset（依赖驱动）。 -->

    <update
      id="updateUser"
      parameterType="com.demo.User"
      flushCache="true"
      statementType="PREPARED"
      timeout="20">

    <delete
      id="deleteUser"
      parameterType="com.demo.User"
      flushCache="true"
      statementType="PREPARED"
      timeout="20">
</mapper>
```


### 14. 加载mapper文件mapperElement
mappers节点下，配置我们的mapper映射文件，所谓的mapper映射文件，就是让mybatis用来建立数据表和javabean映射的一个桥梁。
在我们实际开发中，通常一个mapper文件对应一个dao接口，这个mapper可以看做是dao的实现。所以,mappers必须配置。
```
<configuration>
    ......
    <mappers>
      <!-- 第一种方式：通过resource指定 -->
    <mapper resource="com/dy/dao/userDao.xml"/>
    
     <!-- 第二种方式， 通过class指定接口，进而将接口与对应的xml文件形成映射关系
             不过，使用这种方式必须保证 接口与mapper文件同名(不区分大小写)， 
             我这儿接口是UserDao,那么意味着mapper文件为UserDao.xml 
     <mapper class="com.dy.dao.UserDao"/>
      -->
      
      <!-- 第三种方式，直接指定包，自动扫描，与方法二同理 
      <package name="com.dy.dao"/>
      -->
      <!-- 第四种方式：通过url指定mapper文件位置
      <mapper url="file://........"/>
       -->
  </mappers>
    ......
</configuration>
```

## mapper加载与初始化
```
  mapperElement(root.evalNode("mappers"));
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          //1. 如果mappers节点的子节点是package, 那么就扫描package下的文件, 注入进configuration
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          //resource, url, class 三选一
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            //2. mapper映射文件都是通过XMLMapperBuilder解析
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            // 3. 按照resource加载
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            // 4. 按照类型进行加载
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```
对于通过package加载的mapper文件，调用mapperRegistry.addMappers(packageName);进行加载，
其核心逻辑在org.apache.ibatis.binding.MapperRegistry中，
对于每个找到的接口或者mapper文件，最后调用用XMLMapperBuilder进行具体解析。
对于明确指定的mapper文件或者mapper接口，则主要使用XMLMapperBuilder进行具体解析。

### 按照包名加载
Configuration.java
```
  public void addMappers(String packageName) {
    mapperRegistry.addMappers(packageName);
  }
```

MapperRegistry.java
```
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }
  
  public void addMappers(String packageName, Class<?> superType) {
    // mybatis框架提供的搜索classpath下指定package以及子package中符合条件(注解或者继承于某个类/接口)的类，
    // 默认使用Thread.currentThread().getContextClassLoader()返回的加载器,和spring的工具类殊途同归。
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    // 无条件的加载所有的类,因为调用方传递了Object.class作为父类,这也给以后的指定mapper接口预留了余地
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    // 所有匹配的calss都被存储在ResolverUtil.matches字段中
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      //调用addMapper方法进行具体的mapper类/接口解析
      addMapper(mapperClass);
    }
  }
```
通过搜索包下的类文件最终统一为按照类型加载

### 按照类型加载
```
  public <T> void addMapper(Class<T> type) {
    // 对于mybatis mapper接口文件，必须是interface，不能是class
    if (type.isInterface()) {
      // 判重，确保只会加载一次不会被覆盖
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        // 1. 为mapper接口创建一个MapperProxyFactory代理
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        // 2. MapperAnnotationBuilder进行具体的解析
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        //剔除解析出现异常的接口
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```
knownMappers是MapperRegistry的主要字段，维护了Mapper接口和代理类的映射关系,key是mapper接口类，value是MapperProxyFactory

MapperProxyFactory.java
```
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();
  ......
}
```
从定义看出，MapperProxyFactory主要是维护mapper接口的方法与对应mapper文件中具体CRUD节点的关联关系。
其中每个Method与对应MapperMethod维护在一起。MapperMethod是mapper中具体映射语句节点的内部表示。

### 标签解析
MapperAnnotationBuilder.java
```
  static {
    SQL_ANNOTATION_TYPES.add(Select.class);
    SQL_ANNOTATION_TYPES.add(Insert.class);
    SQL_ANNOTATION_TYPES.add(Update.class);
    SQL_ANNOTATION_TYPES.add(Delete.class);

    SQL_PROVIDER_ANNOTATION_TYPES.add(SelectProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(InsertProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(UpdateProvider.class);
    SQL_PROVIDER_ANNOTATION_TYPES.add(DeleteProvider.class);
  }

  public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
    String resource = type.getName().replace('.', '/') + ".java (best guess)";
    this.assistant = new MapperBuilderAssistant(configuration, resource);
    this.configuration = configuration;
    this.type = type;
  }
```
MapperBuilderAssistant和XMLConfigBuilder一样，都是继承于BaseBuilder。
Select.class/Insert.class等注解指示该方法对应的真实sql语句类型分别是select/insert。
SelectProvider.class/InsertProvider.class主要用于动态SQL，它们允许你指定一个类名和一个方法在具体执行时返回要运行的SQL语句。
MyBatis会实例化这个类，然后执行指定的方法。

