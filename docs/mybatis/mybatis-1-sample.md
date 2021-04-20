##mybatis使用样例
MybatisHelloWorld.java
```
public class MybatisHelloWorld {
    public static void main(String[] args) {
        String resource = "org/mybatis/internal/example/Configuration.xml";
        Reader reader;
        try {
            reader = Resources.getResourceAsReader(resource);
            SqlSessionFactory sqlMapper = new  SqlSessionFactoryBuilder().build(reader);

            SqlSession session = sqlMapper.openSession();
            try {
                User user = (User) session.selectOne("org.mybatis.internal.example.mapper.UserMapper.getUser", 1);
                System.out.println(user.getLfPartyId() + "," + user.getPartyName());
            } finally {
                session.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

org/mybatis/internal/example/Configuration.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://10.7.12.4:3306/lfBase?useUnicode=true"/>
                <property name="username" value="lfBase"/>
                <property name="password" value="eKffQV6wbh3sfQuFIG6M"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="org/mybatis/internal/example/UserMapper.xml"/>
    </mappers>
</configuration>
```

org/mybatis/internal/example/UserMapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.mybatis.internal.example.mapper.UserMapper">
    <select id="getUser" parameterType="int" resultType="org.mybatis.internal.example.pojo.User">
        select lfPartyId,partyName from LfParty where lfPartyId = #{id}
    </select>
</mapper>
```

UserMapper.java
```
public interface UserMapper {
    public User getUser(int lfPartyId);
}
```
