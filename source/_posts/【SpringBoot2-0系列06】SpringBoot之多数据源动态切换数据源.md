---
title: 【SpringBoot2.0系列06】SpringBoot之多数据源动态切换数据源
date: 2018-08-19 13:59:55
categories: SpringBoot
tags: [SpringBoot, DynamicDataSource, DataSource]
---

在前面两节我们已经完成springboot操作mysql数据库，但是在实际业务场景中，数据量迅速增长，一个库一个表已经满足不了我们的需求的时候，我们就会考虑分库分表的操作，那么接下来我们就去学习一下，在springboot中如何实现多数据源，动态数据源切换，读写分离等操作。
<!-- more -->
# 实现
## 1、建库建表
首先，我们在本地新建三个数据库名分别为`master`,`slave1`,`slave2`，我们的目前就是写入操作都是在`master`，查询是 `slave1,slave2`
因此我们在上一篇也就是[【SpringBoot2.0系列05】SpringBoot之整合Mybatis](https://www.jianshu.com/p/c44dc639cb93)基础上进行改动，
我们在`master slave1 slave2`中都创建`user`表 其中初始化`salve1`库的`user`表数据为
![image.png](https://upload-images.jianshu.io/upload_images/5338436-55e8c5ec33750c51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
初始化
`slave2`库的`user`表
![image.png](https://upload-images.jianshu.io/upload_images/5338436-654ad5a554b993e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
具体的数据库脚本如下
```sql 
create table master.user
(
	id bigint auto_increment comment '主键'
		primary key,
	age int null comment '年龄',
	password varchar(32) null comment '密码',
	sex int null comment '性别',
	username varchar(32) null comment '用户名'
)
engine=MyISAM collate=utf8mb4_bin
;

create table slave1.user
(
	id bigint auto_increment comment '主键'
		primary key,
	age int null comment '年龄',
	password varchar(32) null comment '密码',
	sex int null comment '性别',
	username varchar(32) null comment '用户名'
)
engine=MyISAM collate=utf8mb4_bin
;

INSERT INTO slave1.user (id, age, password, sex, username) VALUES (2, 22, 'admin', 1, 'admin');

create table slave2.user
(
	id bigint auto_increment comment '主键'
		primary key,
	age int null comment '年龄',
	password varchar(32) null comment '密码',
	sex int null comment '性别',
	username varchar(32) null comment '用户名'
)
engine=MyISAM collate=utf8mb4_bin
;
INSERT INTO slave2.user (id, age, password, sex, username) VALUES (3, 19, 'uuu', 2, 'user');
INSERT INTO slave2.user (id, age, password, sex, username) VALUES (4, 18, 'bbbb', 1, 'zzzz');
```
## 2、配置多数据源
经过上面初始化 我们的`master.user`是一张空表，我们等下的插入与更新操作就在这上面，那么我们的查询操作就是在`slave1.user跟slave2.user`上面了。
上面我们的数据库初始化工作完成了，接下来就是实现动态数据源的过程
首先我们需要在我们的`application.yml`配置我们的三个数据源
```yml
server:
  port: 8989
spring:
  datasource:
    master:
      password: root
      url: jdbc:mysql://127.0.0.1:3306/master?useUnicode=true&characterEncoding=UTF-8
      driver-class-name: com.mysql.jdbc.Driver
      username: root
      type: com.zaxxer.hikari.HikariDataSource
    cluster:
    - key: slave1
      password: root
      url: jdbc:mysql://127.0.0.1:3306/slave1?useUnicode=true&characterEncoding=UTF-8
      idle-timeout: 20000
      driver-class-name: com.mysql.jdbc.Driver
      username: root
      type: com.zaxxer.hikari.HikariDataSource
    - key: slave2
      password: root
      url: jdbc:mysql://127.0.0.1:3306/slave2?useUnicode=true&characterEncoding=UTF-8
      driver-class-name: com.mysql.jdbc.Driver
      username: root
mybatis:
  mapper-locations: classpath:/mybatis/mapper/*.xml
  config-location:  classpath:/mybatis/config/mybatis-config.xml
```
在上面我们配置了三个数据，其中第一个作为默认数据源也就是我们的`master`数据源。主要是写操作，那么读操作交给我们的`slave1跟slave2`
其中  master 数据源是一定要配置 作为我们的默认数据源，其次cluster集群中，其他的数据不配置也不会影响程序的运行（相当于单数据源），如果你想添加新的一个数据源 就在cluster下新增一个数据源即可，其中key为必须项，用于数据源的唯一标识，以及接下来切换数据源的标识。
## 3、注册数据源
在上面我们已经配置了三个数据源，但是这是我们自定义的配置，springboot是无法给我们自动配置，所以需要我们自己注册数据源.
那么就要实现 `EnvironmentAware`用于读取上下文环境变量用于构建数据源，同时也需要实现 `ImportBeanDefinitionRegistrar`接口注册我们构建的数据源。`com.yukong.chapter5.register.DynamicDataSourceRegister `具体代码如下
```java
/**
 * 动态数据源注册
 * 实现 ImportBeanDefinitionRegistrar 实现数据源注册
 * 实现 EnvironmentAware 用于读取application.yml配置
 */
public class DynamicDataSourceRegister implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceRegister.class);


    /**
     * 配置上下文（也可以理解为配置文件的获取工具）
     */
    private Environment evn;

    /**
     * 别名
     */
    private final static ConfigurationPropertyNameAliases aliases = new ConfigurationPropertyNameAliases();

    /**
     * 由于部分数据源配置不同，所以在此处添加别名，避免切换数据源出现某些参数无法注入的情况
     */
    static {
        aliases.addAliases("url", new String[]{"jdbc-url"});
        aliases.addAliases("username", new String[]{"user"});
    }

    /**
     * 存储我们注册的数据源
     */
    private Map<String, DataSource> customDataSources = new HashMap<String, DataSource>();

    /**
     * 参数绑定工具 springboot2.0新推出
     */
    private Binder binder;

    /**
     * ImportBeanDefinitionRegistrar接口的实现方法，通过该方法可以按照自己的方式注册bean
     *
     * @param annotationMetadata
     * @param beanDefinitionRegistry
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        // 获取所有数据源配置
        Map config, defauleDataSourceProperties;
        defauleDataSourceProperties = binder.bind("spring.datasource.master", Map.class).get();
        // 获取数据源类型
        String typeStr = evn.getProperty("spring.datasource.master.type");
        // 获取数据源类型
        Class<? extends DataSource> clazz = getDataSourceType(typeStr);
        // 绑定默认数据源参数 也就是主数据源
        DataSource consumerDatasource, defaultDatasource = bind(clazz, defauleDataSourceProperties);
        DynamicDataSourceContextHolder.dataSourceIds.add("master");
        logger.info("注册默认数据源成功");
        // 获取其他数据源配置
        List<Map> configs = binder.bind("spring.datasource.cluster", Bindable.listOf(Map.class)).get();
        // 遍历从数据源
        for (int i = 0; i < configs.size(); i++) {
            config = configs.get(i);
            clazz = getDataSourceType((String) config.get("type"));
            defauleDataSourceProperties = config;
            // 绑定参数
            consumerDatasource = bind(clazz, defauleDataSourceProperties);
            // 获取数据源的key，以便通过该key可以定位到数据源
            String key = config.get("key").toString();
            customDataSources.put(key, consumerDatasource);
            // 数据源上下文，用于管理数据源与记录已经注册的数据源key
            DynamicDataSourceContextHolder.dataSourceIds.add(key);
            logger.info("注册数据源{}成功", key);
        }
        // bean定义类
        GenericBeanDefinition define = new GenericBeanDefinition();
        // 设置bean的类型，此处DynamicRoutingDataSource是继承AbstractRoutingDataSource的实现类
        define.setBeanClass(DynamicRoutingDataSource.class);
        // 需要注入的参数
        MutablePropertyValues mpv = define.getPropertyValues();
        // 添加默认数据源，避免key不存在的情况没有数据源可用
        mpv.add("defaultTargetDataSource", defaultDatasource);
        // 添加其他数据源
        mpv.add("targetDataSources", customDataSources);
        // 将该bean注册为datasource，不使用springboot自动生成的datasource
        beanDefinitionRegistry.registerBeanDefinition("datasource", define);
        logger.info("注册数据源成功，一共注册{}个数据源", customDataSources.keySet().size() + 1);
    }

    /**
     * 通过字符串获取数据源class对象
     *
     * @param typeStr
     * @return
     */
    private Class<? extends DataSource> getDataSourceType(String typeStr) {
        Class<? extends DataSource> type;
        try {
            if (StringUtils.hasLength(typeStr)) {
                // 字符串不为空则通过反射获取class对象
                type = (Class<? extends DataSource>) Class.forName(typeStr);
            } else {
                // 默认为hikariCP数据源，与springboot默认数据源保持一致
                type = HikariDataSource.class;
            }
            return type;
        } catch (Exception e) {
            throw new IllegalArgumentException("can not resolve class with type: " + typeStr); //无法通过反射获取class对象的情况则抛出异常，该情况一般是写错了，所以此次抛出一个runtimeexception
        }
    }

    /**
     * 绑定参数，以下三个方法都是参考DataSourceBuilder的bind方法实现的，目的是尽量保证我们自己添加的数据源构造过程与springboot保持一致
     *
     * @param result
     * @param properties
     */
    private void bind(DataSource result, Map properties) {
        ConfigurationPropertySource source = new MapConfigurationPropertySource(properties);
        Binder binder = new Binder(new ConfigurationPropertySource[]{source.withAliases(aliases)});
        // 将参数绑定到对象
        binder.bind(ConfigurationPropertyName.EMPTY, Bindable.ofInstance(result));
    }

    private <T extends DataSource> T bind(Class<T> clazz, Map properties) {
        ConfigurationPropertySource source = new MapConfigurationPropertySource(properties);
        Binder binder = new Binder(new ConfigurationPropertySource[]{source.withAliases(aliases)});
        // 通过类型绑定参数并获得实例对象
        return binder.bind(ConfigurationPropertyName.EMPTY, Bindable.of(clazz)).get();
    }

    /**
     * @param clazz
     * @param sourcePath 参数路径，对应配置文件中的值，如: spring.datasource
     * @param <T>
     * @return
     */
    private <T extends DataSource> T bind(Class<T> clazz, String sourcePath) {
        Map properties = binder.bind(sourcePath, Map.class).get();
        return bind(clazz, properties);
    }

    /**
     * EnvironmentAware接口的实现方法，通过aware的方式注入，此处是environment对象
     *
     * @param environment
     */
    @Override
    public void setEnvironment(Environment environment) {
        logger.info("开始注册数据源");
        this.evn = environment;
        // 绑定配置器
        binder = Binder.get(evn);
    }
}

```
上面代码需要注意的是在`springboot2.x`系列中用于绑定的工具类如RelaxedPropertyResolver已经无法现在使用`Binder`代替。上面代码主要是读取application中数据源的配置，先读取`spring.datasource.maste`r构建默认数据源,然后在构建`cluster`中的数据源。
在这里注册完数据源之后，我们需要通过@import注解把我们的数据源注册器导入到spring中 在启动类`Chapter5Application.java`加上如下注解`@Import(DynamicDataSourceRegister.class)`。
其中我们用到了一个`DynamicDataSourceContextHolder` 中的静态变量来保存我们已经注册成功的数据源的`key`,至此我们的数据源注册就已经完成了。
## 4、配置数据源上下文
我们需要新建一个数据源上下文，用户记录当前线程使用的数据源的key是什么，以及记录所有注册成功的数据源的key的集合。对于线程级别的私有变量，我们首先`ThreadLocal`来实现。 
`com.yukong.chapter5.config.DynamicDataSourceContextHolder `代码取下
```java
/**
 * @Auther: yukong
 * @Date: 2018/8/15 10:49
 * @Description: 数据源上下文
 */
public class DynamicDataSourceContextHolder {



    private static Logger logger = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);

    /**
     * 存储已经注册的数据源的key
     */
    public static List<String> dataSourceIds = new ArrayList<>();

    /**
     * 线程级别的私有变量
     */
    private static final ThreadLocal<String> HOLDER = new ThreadLocal<>();

    public static String getDataSourceRouterKey () {
        return HOLDER.get();
    }

    public static void setDataSourceRouterKey (String dataSourceRouterKey) {
        logger.info("切换至{}数据源", dataSourceRouterKey);
        HOLDER.set(dataSourceRouterKey);
    }

    /**
     * 设置数据源之前一定要先移除
     */
    public static void removeDataSourceRouterKey () {
        HOLDER.remove();
    }

    /**
     * 判断指定DataSrouce当前是否存在
     *
     * @param dataSourceId
     * @return
     */
    public static boolean containsDataSource(String dataSourceId){
        return dataSourceIds.contains(dataSourceId);
    }

}
```
## 5、动态数据源路由
前面我们以及新建了数据源上下文，用于存储我们当前线程的数据源key那么怎么通知`spring`用key当前的数据源呢，查阅资料可知，`spring`提供一个接口，名为`AbstractRoutingDataSource`的抽象类，我们只需要重写`determineCurrentLookupKey`方法就可以，这个方法看名字就知道，就是返回当前线程的数据源的key，那我们只需要从我们刚刚的数据源上下文中取出我们的key即可，那么具体代码取下。
`com.yukong.chapter5.config.DynamicRoutingDataSource `
```java
/**
 * @Auther: yukong
 * @Date: 2018/8/15 10:47
 * @Description: 动态数据源路由配置
 */
public class DynamicRoutingDataSource extends AbstractRoutingDataSource {



    private static Logger logger = LoggerFactory.getLogger(DynamicRoutingDataSource.class);

    @Override
    protected Object determineCurrentLookupKey() {
        String dataSourceName = DynamicDataSourceContextHolder.getDataSourceRouterKey();
        logger.info("当前数据源是：{}", dataSourceName);
        return DynamicDataSourceContextHolder.getDataSourceRouterKey();
    }
}
```
## 6、通过aop+注解实现动态数据源的切换
现在spring也已经知道通过key来取对应的数据源，我们现在只需要实现给对应的类或者方法设置他们的数据源的key，并且保存在数据源上下文中即可。这里我们采用注解来设置数据源，通过aop拦截并且保存到数据源上下中。
我们新建一个标识数据源的注解`@DataSource`具体代码取下
`com.yukong.chapter5.annotation.DataSource `
```java
/**
 * 切换数据注解 可以用于类或者方法级别 方法级别优先级 > 类级别
 */
@Target({ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataSource {
    String value() default "master"; //该值即key值
}
```
其中他的默认值是`master`,因为我们默认数据源的key也是`master`。也就是说如果你直接用注解，而不指定value的话，那么默认就使用master默认数据源。
然后我们新建一个aop类来拦截。代码如下
`com.yukong.chapter5.aop`
```java
package com.yukong.chapter5.aop;

import com.yukong.chapter5.annotation.DataSource;
import com.yukong.chapter5.config.DynamicDataSourceContextHolder;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

@Aspect
@Component
public class DynamicDataSourceAspect {
    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceAspect.class);

    @Before("@annotation(ds)")
    public void changeDataSource(JoinPoint point, DataSource ds) throws Throwable {
        String dsId = ds.value();
        if (DynamicDataSourceContextHolder.dataSourceIds.contains(dsId)) {
            logger.debug("Use DataSource :{} >", dsId, point.getSignature());
        } else {
            logger.info("数据源[{}]不存在，使用默认数据源 >{}", dsId, point.getSignature());
            DynamicDataSourceContextHolder.setDataSourceRouterKey(dsId);
        }
    }

    @After("@annotation(ds)")
    public void restoreDataSource(JoinPoint point, DataSource ds) {
        logger.debug("Revert DataSource : " + ds.value() + " > " + point.getSignature());
        DynamicDataSourceContextHolder.removeDataSourceRouterKey();

    }
}
```
通过aop拦截，获取注解上面的value的值key，然后取判断我们注册的keys集合中是否有这个key,如果没有，则使用默认数据源，如果有，则设置上下文中当前数据源的key为注解的value。
7、测试
最后我们在对应的方法上面加上注解来测试一下即可
我们在UserMapper.java上面加上注解，并且进行测试。
```java
/**
 * @Auther: yukong
 * @Date: 2018/8/13 19:47
 * @Description: UserMapper接口
 */
public interface UserMapper {

    /**
     * 新增用户
     * @param user
     * @return
     */
    @DataSource  //默认数据源
    int save(User user);

    /**
     * 更新用户信息
     * @param user
     * @return
     */
    @DataSource  //默认数据源
    int update(User user);

    /**
     * 根据id删除
     * @param id
     * @return
     */
    @DataSource  //默认数据源
    int deleteById(Long id);

    /**
     * 根据id查询
     * @param id
     * @return
     */
    @DataSource("slave1")  //slave1
    User selectById(Long id);

    /**
     * 查询所有用户信息
     * @return
     */
    @DataSource("slave2")  //slave2
    List<User> selectAll();
}
```
上面代码可以知道，我们的新增，修改，删除方法都是在默认数据master上，我们的id查询是在slave1，我们的查询所有在slave2，我们编写测试类来测试把。
```java
/**
 * @Auther: yukong
 * @Date: 2018/8/14 16:34
 * @Description:
 */
@SpringBootTest
@RunWith(SpringJUnit4ClassRunner.class)
public class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void save() {
        User user = new User();
        user.setUsername("master");
        user.setPassword("master");
        user.setSex(1);
        user.setAge(18);
        Assert.assertEquals(1,userMapper.save(user));
    }

    @Test
    public void update() {
        User user = new User();
        user.setId(8L);
        user.setPassword("newpassword");
        // 返回插入的记录数 ，期望是1条 如果实际不是一条则抛出异常
        Assert.assertEquals(1,userMapper.update(user));
    }

    @Test
    public void selectById() {
        User user = userMapper.selectById(2L);
        System.out.println("id:" + user.getId());
        System.out.println("name:" + user.getUsername());
        System.out.println("password:" + user.getPassword());
    }

    @Test
    public void deleteById() {
        Assert.assertEquals(1,userMapper.deleteById(1L));
    }

    @Test
    public void selectAll() {
        List<User> users= userMapper.selectAll();
        users.forEach(user -> {
            System.out.println("id:" + user.getId());
            System.out.println("name:" + user.getUsername());
            System.out.println("password:" + user.getPassword());
        });
    }

}
```

首先测试save方法，它将会把数据存到master库的user表，
现在user表是空的，如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-0452a7603f8e8863.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
运行save方法。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-3bb94c5afb742fa4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

绿色，测试通过，并且日志提示数据源注册成功，一共三个。并且当前使用的master数据源，我们再去master数据库看看有没有数据。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-e924fd683ed754ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图，插入成功。
新增方法测试完成了。我们在测试一下修改与删除。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-6e9602782efba2e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
修改方法也测试通过，查看数据库。![image.png](https://upload-images.jianshu.io/upload_images/5338436-2409a39e052905bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
修改成功，删除方法我就不测试， 我们在测试测试，slave1跟slave2数据源的方法，
首先测试`slave1`的主键查询方法，先看数据库 `slave1`有哪些数据。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-bd1c8f513bb8f3c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`slave1.user`就一条id为2 的数据并且id为2 的数据就slave1才有，我们测试一下能不能查到。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-d9055eab0836a063.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
运行通过，数据源为`slave1`并且数据也正确显示。
最后我们来测试一下`slave2`的selectAll方法把，同样先看看`slave2.user`中有什么数据。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-1c6cefa80e5d4d25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从图中，得知`slave2.user`中有两条数据，id分别为3，4。接下来运行测试方法。
结果如图。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-d51b7d61b5bdced3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
日志提示数据源切换值`slave2`，并且id为3，4的数据也成功打印。
那么至此我们的**多数据源动态数据源**就完成了。

主要的思路就是
1. 配置文件中配置多个数据源 
2. 启动类注册动态数据源 
3. 在需要的方法上使用注解指定数据源


最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。