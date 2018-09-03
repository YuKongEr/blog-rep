---
title: 【SpringBoot2.0系列07】SpringBoot之redis使用（Lettuce版本）
date: 2018-08-20 14:00:04
categories: SpringBoot
tags: [SpringBoot, Redis, Lettuce]
---
前面三节我们讲解了`springboot`与关系型数据库交互，现在我们需要了解一下`springboot`,今天我们就需要学习了与`nosql`数据库交互，今天我们主要讲一下`springboot`如果操作`redis`。 目前java操作redis的客户端有`jedis`跟`Lettuce`。在`springboot1.x`系列中，其中使用的是`jedis`,但是到了`springboot2.x`其中使用的是`Lettuce`。 因为我们的版本是`springboot2.x`系列，所以今天使用的是`Lettuce`。
  关于`jedis`跟`lettuce`的区别：
* Lettuce 和 Jedis 的定位都是Redis的client，所以他们当然可以直接连接redis server。
* Jedis在实现上是直接连接的redis server，如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个Jedis实例增加物理连接
* Lettuce的连接是基于Netty的，连接实例（StatefulRedisConnection）可以在多个线程间并发访问，应为StatefulRedisConnection是线程安全的，所以一个连接实例（StatefulRedisConnection）就可以满足多线程环境下的并发访问，当然这个也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。

<!-- more -->

# 实现
  ## 1、依赖
  新建一个springboot工程，添加如下依赖。
```xml
<dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- redis依赖commons-pool 这个依赖一定要添加 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
    </dependencies>
```
然后在`application.yml`配置一下`redis`服务器的地址
```yml
server:
  port: 8989
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    # 密码 没有则可以不填
    password: 123456
    # 如果使用的jedis 则将lettuce改成jedis即可
    lettuce:
      pool:
        # 最大活跃链接数 默认8
        max-active: 8
        # 最大空闲连接数 默认8
        max-idle: 8
        # 最小空闲连接数 默认0
        min-idle: 0

```

## 2、 redis配置
接下来我们需要配置redis的key跟value的序列化方式，默认使用的`JdkSerializationRedisSerializer` 这样的会导致我们通过`redis desktop manager`显示的我们key跟value的时候显示不是正常字符。 所以我们需要手动配置一下序列化方式 新建一个`config`包，在其下新建一个`RedisConfig.java` 具体代码如下
```java
/**
 * @Auther: yukong
 * @Date: 2018/8/17 14:58
 * @Description: redis配置
 */
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfig {

    /**
     * 配置自定义redisTemplate
     * @return
     */
    @Bean
    RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {

        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);

        //使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值
        Jackson2JsonRedisSerializer serializer = new Jackson2JsonRedisSerializer(Object.class);

        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);

        template.setValueSerializer(serializer);
        //使用StringRedisSerializer来序列化和反序列化redis的key值
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);
        template.afterPropertiesSet();
        return template;
    }

}
```
其中`@Configuration` 代表这个类是一个配置类，然后`@AutoConfigureAfter(RedisAutoConfiguration.class)` 是让我们这个配置类在内置的配置类之后在配置，这样就保证我们的配置类生效，并且不会被覆盖配置。其中需要注意的就是方法名一定要叫`redisTemplate`  因为`@Bean`注解是根据方法名配置这个bean的name的。

## 3、测试

我们需要测试在redis缓存对象的用例，所以我们需要新建一个实体类。
代码如下：
```java
public class User implements Serializable {

    private static final long serialVersionUID = 1222221L;

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

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                ", age=" + age +
                ", sex=" + sex +
                '}';
    }
}
我们在`Chapter6Application.java`测试一下
```
代码如下
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class Chapter6ApplicationTests {

    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void redisTest() {
        // redis存储数据
        String key = "name";
        redisTemplate.opsForValue().set(key, "yukong");
        // 获取数据
        String value = (String) redisTemplate.opsForValue().get(key);
        System.out.println("获取缓存中key为" + key + "的值为：" + value);

        User user = new User();
        user.setUsername("yukong");
        user.setSex(18);
        user.setId(1L);
        String userKey = "yukong";
        redisTemplate.opsForValue().set(userKey, user);
        User newUser = (User) redisTemplate.opsForValue().get(userKey);
        System.out.println("获取缓存中key为" + userKey + "的值为：" + newUser);

    }

}
```

首先我们引入`springboot `的测试环境，然后注入一个`RedisTemplate`对象，这里我解释一下为什么我前面让大家在配置`redis`的时候 放回RedisTemplate的方法的方法名一定要叫`redisTemplate`因为
 * 如果查询结果刚好为一个，就将该bean装配给@Autowired指定的数据

* 如果查询的结果不止一个，那么@Autowired会根据名称来查找。

* 如果查询的结果为空，那么会抛出异常。解决方法时，使用required=false

如果我们没有把哪个bean命名为`redisTemplate` 而这里又这么注入，会导致我们使用的`springboot`内置的tempalte而不是我们配置的，导致我们的配置不生效，从而埋坑，所以最好的方法就是把方法命名为`redisTemplate`从而覆盖系统内置的。

接下来我们运行测试类。结果如下。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-517e376b5248a6d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时查看redis中缓存的结果。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-14c10f7d55f58fc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/5338436-ecf99f33a27ec3eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


中文成功显示，并且对象在redis以json方式存储，代表我们配置成功。

下列的就是Redis其它类型所对应的操作方式

- opsForValue： 对应 String（字符串）
- opsForZSet： 对应 ZSet（有序集合）
- opsForHash： 对应 Hash（哈希）
- opsForList： 对应 List（列表）
- opsForSet： 对应 Set（集合）

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。


