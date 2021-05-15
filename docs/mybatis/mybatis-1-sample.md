[源码分析-环境准备](https://www.cnblogs.com/xrq730/p/6792031.html)  

### SQL准备
```
drop table if exists mail;

create table mail
(
  id          int         auto_increment not null comment '主键id',
  create_time datetime    not null  comment '创建时间',
  modify_time timestamp   not null  comment '修改时间',
  web_id      int         not null  comment '站点id，1表示新浪，2表示QQ，3表示搜狐，4表示火狐',
  mail        varchar(50) not null  comment '邮箱名',
  use_for     varchar(30)           comment '邮箱用途',
  primary key(id),
  index use_for(use_for),
  unique index web_id_mail(web_id, mail)
)charset=utf8 engine=innodb comment='邮箱表';
```

### 建立实体类
```
public class Mail {

    /** 主键id */
    private long id;

    /** 创建时间 */
    private Date createTime;

    /** 修改时间 */
    private Date modifyTime;

    /** 网站id，1表示新浪，2表示QQ，3表示搜狐，4表示火狐 */
    private int webId;

    /** 邮箱 */
    private String mail;

    /** 用途 */
    private String useFor;

    public Mail() {

    }

    public Mail(int webId, String mail, String useFor) {
        this.webId = webId;
        this.mail = mail;
        this.useFor = useFor;
    }

    /** 此处省略get和set方法 */

    @Override
    public String toString() {
        return "MailDO [id=" + id + ", createTime=" + createTime + ", modifyTime=" + modifyTime + ", webId=" + webId + ", mail=" + mail + ", useFor="
                + useFor + "]";
    }

}
```

### 建立数据访问层
- 数据访问层使用Dao命名，它定义了对表的基本增删改查操作
- 数据访问层之上使用Service命名，它的作用是对于数据库的多操作进行组合，比如先查再删、先删再增、先改再查再删等等，
这些操作不会放在Dao层面去操作，而会放在Service层面去进行组合
  
```
public interface MailDao {

    /** 插入一条邮箱信息 */
    public long insertMail(Mail mail);

    /** 删除一条邮箱信息 */
    public int deleteMail(long id);

    /** 更新一条邮箱信息 */
    public int updateMail(Mail mail);

    /** 查询邮箱列表 */
    public List<Mail> selectMailList();

    /** 根据主键id查询一条邮箱信息 */
    public Mail selectMailById(long id);
}

public class MailDaoImpl implements MailDao {

    private static final String NAME_SPACE = "MailMapper.";

    private static SqlSessionFactory ssf;

    private static Reader reader;

    static {
        try {
            reader = Resources.getResourceAsReader("mybatis/config.xml");
            ssf = new SqlSessionFactoryBuilder().build(reader);
        }
        catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public long insertMail(Mail mail) {
        SqlSession ss = ssf.openSession();
        try {
            int rows = ss.insert(NAME_SPACE + "insertMail", mail);
            ss.commit();
            if (rows > 0) {
                return mail.getId();
            }
            return 0;
        } catch (Exception e) {
            ss.rollback();
            return 0;
        } finally {
            ss.close();
        }
    }

    @Override
    public int deleteMail(long id) {
        SqlSession ss = ssf.openSession();
        try {
            int rows = ss.delete(NAME_SPACE + "deleteMail",  id);
            ss.commit();
            return rows;
        } catch (Exception e) {
            ss.rollback();
            return 0;
        } finally {
            ss.close();
        }
    }

    @Override
    public int updateMail(Mail mail) {
        SqlSession ss = ssf.openSession();
        try {
            int rows = ss.update(NAME_SPACE + "updateMail", mail);
            ss.commit();
            return rows;
        } catch (Exception e) {
            ss.rollback();
            return 0;
        } finally {
            ss.close();
        }
    }

    @Override
    public List<Mail> selectMailList() {
        SqlSession ss = ssf.openSession();
        try {
            return ss.selectList(NAME_SPACE + "selectMailList");
        } finally {
            ss.close();
        }
    }

    @Override
    public Mail selectMailById(long id) {
        SqlSession ss = ssf.openSession();
        try {
            return ss.selectOne(NAME_SPACE + "selectMailById", id);
        } finally {
            ss.close();
        }
    }

}
```

### 建立MyBatis配置文件
MyBatis的配置文件有两个，一个是环境的配置config.xml，一个是具体SQL的编写mail.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <properties resource="properties/db.properties" />

    <settings>
        <setting name="cacheEnabled" value="true" />
        <setting name="lazyLoadingEnabled" value="true"/>
        <setting name="useGeneratedKeys" value="true"/>
    </settings>

    <typeAliases>
        <typeAlias alias="Mail" type="org.xrq.mybatis.pojo.Mail"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${driveClass}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${userName}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mybatis/mail.xml"/>
    </mappers>

</configuration>


<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="MailMapper">

    <resultMap type="Mail" id="MailResultMap">
        <result column="id" property="id" />
        <result column="create_time" property="createTime" />
        <result column="modify_time" property="modifyTime" />
        <result column="web_id" property="webId" />
        <result column="mail" property="mail" />
        <result column="use_for" property="useFor" />
    </resultMap>

    <sql id="fields">
        id, create_time, modify_time, web_id, mail, use_for
    </sql>

    <sql id="fields_value">
        null, now(), now(), #{webId}, #{mail}, #{useFor}
    </sql>

    <insert id="insertMail" parameterType="Mail" useGeneratedKeys="true" keyProperty="id">
        insert into mail(
            <include refid="fields" />
        ) values(
            <include refid="fields_value" />
        );
    </insert>

    <delete id="deleteMail" parameterType="java.lang.Long">
        delete from mail where id = #{id};
    </delete>

    <update id="updateMail" parameterType="Mail">
        update mail
        <set>
            <if test="web_id != 0">
                web_id = #{webId}
            </if>
            <if test="mail != null">
                mail = #{mail}
            </if>
            <if test="use_for != null">
                use_for = #{useFor}
            </if>
        </set>
        where id = #{id};
    </update>

    <select id="selectMailList" resultMap="MailResultMap">
        select <include refid="fields" /> from mail;
    </select>

    <select id="selectMailById" resultMap="MailResultMap" parameterType="java.lang.Long">
        select <include refid="fields" /> from mail where id = #{id};
    </select>

</mapper>
```