---
title: 【SpringBoot2.0系列05】SpringBoot之整合Mybatis
date: 2018-08-18 13:59:46
categories: SpringBoot
tags: [SpringBoot, Mybatis]
---

上一篇博客中，我们完成了`springboot` 使用`spring data jpa`但是在我们实际工作中，可能大部分的同学还是使用`mybatis`比较多，所以今天我们在这里实现一下`springboot`使用`mybatis`实现对`user`表的增删改查并且进行单元测试

<!-- more -->
# 实现
1、添加`mybaits`依赖
```xml
  <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
   </dependencies>
```
并且在`yml`中配置一下数据源跟mybatis的配置。
```yml
server:
  port: 8989
mybatis:
  mapper-locations: classpath:/mybaits/mapper/*.xml
  config-location:  classpath:/mybatis/config/mybatis-config.xml
spring:
  datasource:
    username: root
    password: root
    url: jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf8
    driver-class-name: com.mysql.jdbc.Driver
```

在上面的配置中
```yml
mybatis:
  mapper-locations: classpath:/mybaits/mapper/*.xml
  config-location:  classpath:/mybatis/config/mybatis-config.xml
```
* ` mapper-locations` 是配置我们mapper.xml文件的位置，我们把它配置在`/src/main/resource/mybatis/mapper`目录下。
* ` config-locations` 是配置`mybatis-confg.xml`文件的位置。我们把它配置在`/src/main/resource/mybatis/config`目录下。

接下来我们在`resource`目录下新建`mybatis/mapper`目录跟`mybatis/config`目录，并且在`config`目录下新建`mybatis-config.xml`
具体代码如下
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <typeAliases>
        <package name="com.yukong.chapter4.entity"/>
    </typeAliases>
</configuration>
```
主要是配置了一下开启mybatis的驼峰命名转换跟entity的别名。
接下来就是建表写实体类。
1、 建表
>如果有读过上一篇[【SpringBoot系列04】SpringBoot之使用JPA完成简单的rest api](https://www.jianshu.com/p/b0ece9f17a2f)并且实际操作过的同学就不用建表了，可以跳过建表这一步，没有的同学需要新建一个`test`数据库并且新建`user`表
```ddl
create table test.user
(
	id bigint not null AUTO_INCREMENT comment '主键' primary key,
	age int null comment '年龄',
	password varchar(32) null comment '密码',
	sex int null comment '性别',
	username varchar(32) null comment '用户名'
);
```

新建entity包 并且编写对应的实体类  `User.java`
```java
public class User {


    private Long id;

    /**
     * 用户名
     */
    private String username;

    /**
     * 密码
     */
    private String password;

    /**
     * 年龄
     */
    private Integer age;

    /**
     * 性别 1=男 2=女 其他=保密
     */
    private Integer sex;


    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Integer getSex() {
        return sex;
    }

    public void setSex(Integer sex) {
        this.sex = sex;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
}
```
3、编写mapper接口
接下里我们新建`repository`包 新建`UserMapper.java`接口  并且定义增删改查的方法声明
```java
public interface UserMapper {

    /**
     * 新增用户
     * @param user
     * @return
     */
    int save (User user);

    /**
     * 更新用户信息
     * @param user
     * @return
     */
    int update (User user);

    /**
     * 根据id删除
     * @param id
     * @return
     */
    int deleteById (int id);

    /**
     * 根据id查询
     * @param id
     * @return
     */
    User selectById (int id);

    /**
     * 查询所有用户信息
     * @return
     */
    List<User> selectAll ();
}

```
跟我们之前定义的mapper.xml文件的位置，我们在`resource/mybatis/mapper`目录下 新建与之对应的`UserMapper.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.yukong.chapter4.repository.UserMapper">

    <resultMap id="SysUserResultMap" type="User">
        <id property="id" column="id" javaType="java.lang.Long" jdbcType="BIGINT"/>
        <result property="username" column="USERNAME" javaType="java.lang.String" jdbcType="VARCHAR"/>
        <result property="password" column="PASSWORD" javaType="java.lang.String" jdbcType="VARCHAR"/>
        <result property="sex" column="SEX" javaType="java.lang.Integer" jdbcType="INTEGER"/>
        <result property="age" column="AGE" javaType="java.lang.Integer" jdbcType="INTEGER"/>
    </resultMap>
    <delete id="deleteById">
        delete from user where id=#{id}
    </delete>

    <select id="selectAll"  resultMap="SysUserResultMap">
        select * from user
    </select>


    <insert id="save" parameterType="User" >
        insert into user
        <trim prefix="(" suffix=")" suffixOverrides="," >
            <if test="id != null" >
                id,
            </if>
            <if test="username != null" >
                username,
            </if>
            <if test="password != null" >
                password,
            </if>

            <if test="sex != null" >
                sex,
            </if>
            <if test="age != null" >
                age,
            </if>
        </trim>
        <trim prefix="values (" suffix=")" suffixOverrides="," >
            <if test="id != null" >
                #{id,jdbcType=BIGINT},
            </if>
            <if test="username != null" >
                #{username,jdbcType=VARCHAR},
            </if>
            <if test="password != null" >
                #{password,jdbcType=VARCHAR},
            </if>
            <if test="sex != null" >
                #{sex,jdbcType=INTEGER},
            </if>
            <if test="age != null" >
                #{age,jdbcType=INTEGER},
            </if>
        </trim>
    </insert>


    <update id="update" parameterType="User" >
        update user
        <set >
            <if test="username != null" >
                username = #{username,jdbcType=VARCHAR},
            </if>
            <if test="password != null" >
                password = #{password,jdbcType=VARCHAR},
            </if>
            <if test="sex != null" >
                sex = #{sex,jdbcType=INTEGER},
            </if>
            <if test="age != null" >
                sex = #{age,jdbcType=INTEGER},
            </if>
        </set>
        where id = #{id,jdbcType=BIGINT}
    </update>


    <select id="selectById" resultMap="SysUserResultMap">
        select
        *
        from user
        where id = #{id,jdbcType=BIGINT}
    </select>
</mapper>
```
这时候mybatis的配置还不算成功，我们需要把mapper的路径暴露给spring 让它来扫描管理，所以我们需要在Chapter4Application.java文件上加上注解`@MapperScan("com.yukong.chapter4.repository")` 扫描mapper的所在位置 
至此。我们mybaits的接口算是编写完毕，接下来我们来测试一下把。
4、测试
这次我们用`spring boot`的单元测试来测试我们的接口。
在`idea`中 使用快捷键新建测试类`ctrl+shift+t`
![image.png](https://upload-images.jianshu.io/upload_images/5338436-d98a67a00a716ab0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
勾选需要测试的方法，并且点击ok
这时候会在`src/test/java`目录下对应的包路径新建对应的测试文件
代码如下
```java
@SpringBootTest
@RunWith(SpringJUnit4ClassRunner.class)
public class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void save() {
        User user = new User();
        user.setUsername("zzzz");
        user.setPassword("bbbb");
        user.setSex(1);
        user.setAge(18);
        // 返回插入的记录数 ，期望是1条 如果实际不是一条则抛出异常
        Assert.assertEquals(1,userMapper.save(user));
    }

    @Test
    public void update() {
        User user = new User();
        user.setId(1L);
        user.setPassword("newpassword");
        // 返回更新的记录数 ，期望是1条 如果实际不是一条则抛出异常
        Assert.assertEquals(1,userMapper.update(user));
    }

    @Test
    public void selectById() {
        Assert.assertNotNull(userMapper.selectById(1L));
    }

    @Test
    public void deleteById() {
        Assert.assertEquals(1,userMapper.deleteById(1L));
    }
}
```

使用`spring boot `测试需要在测试类上解释注解
`@SpringBootTest
@RunWith(SpringJUnit4ClassRunner.class)`
另外我们这里使用`Assert`断言来判断实际结果跟预期结果是否一致，如果不一致则抛出异常。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-eada1f1309cd8b89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

右键类旁边的运行按钮，一次测试所有的方法

结果如下
![image.png](https://upload-images.jianshu.io/upload_images/5338436-4810daae11be5424.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 所有的方法通过测试，springboot于mybatis整合成功。

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。