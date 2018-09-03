---
title: 【SpringBoot2.0系列08】SpringBoot之redis数据缓存管理
date: 2018-08-21 14:00:17
categories: SpringBoot
tags: [SpringBoot, Redis, SpringCache]
---
  实现数据缓存，如果缓存中没有数据，则从数据库查询，并且写入redis缓存，如果redis缓存中有数据，则直接从redis中读取，同事删除更新等操作也需要维护缓存。本文基于前面两篇文章而来，部分重复就不贴。
  <!-- more -->
# 实现
## 1、依赖
  ```pom
  <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
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

        <!-- redis依赖commons-pool 这个依赖一定要添加 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
    </dependencies>
```

## 2、配置redis跟mybatis
yml配置如下:
```yml
server:
  port: 8989
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf8
    username: root
    password: root
  redis:
    host: 127.0.0.1
    port: 6379
    lettuce:
      pool:
        max-idle: 8
        min-idle: 0
        max-active: 8
mybatis:
  config-location: classpath:/mybatis/config/mybatis-config.xml
  mapper-locations: classpath:/mybatis/mapper/*.xml
logging:
  level:
    com:
      yukong:
        chapter7:
          repository: debug
```
这里就是配置了一下mybati跟redis  但是比之前多了一点就是配置了我们
`com.yukong.chapter.repository`包的日志级别为`debug`这样sql就会打印出来。
然后之前一样的配置一下`RedisConfig.java`
```java
    
/**
 * @author xiongping22369
 * @date 2018/8/20 15:33
 * @description redis配置  配置序列化方式以及缓存管理器 @EnableCaching 开启缓存
 */
@EnableCaching
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfig {


    /**
     * 配置自定义redisTemplate
     *
     * @param connectionFactory
     * @return
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setValueSerializer(jackson2JsonRedisSerializer());
        //使用StringRedisSerializer来序列化和反序列化redis的key值
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(jackson2JsonRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }

    /**
     * json序列化
     * @return
     */
    @Bean
    public RedisSerializer<Object> jackson2JsonRedisSerializer() {
        //使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值
        Jackson2JsonRedisSerializer serializer = new Jackson2JsonRedisSerializer(Object.class);

        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        return serializer;
    }


    /**
     * 配置缓存管理器
     * @param redisConnectionFactory
     * @return
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        // 生成一个默认配置，通过config对象即可对缓存进行自定义配置
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();
        // 设置缓存的默认过期时间，也是使用Duration设置
        config = config.entryTtl(Duration.ofMinutes(1))
                // 设置 key为string序列化
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                // 设置value为json序列化
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer()))
                // 不缓存空值
                .disableCachingNullValues();

        // 设置一个初始化的缓存空间set集合
        Set<String> cacheNames = new HashSet<>();
        cacheNames.add("timeGroup");
        cacheNames.add("user");

        // 对每个缓存空间应用不同的配置
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put("timeGroup", config);
        configMap.put("user", config.entryTtl(Duration.ofSeconds(120)));

        // 使用自定义的缓存配置初始化一个cacheManager
        RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory)
                // 一定要先调用该方法设置初始化的缓存名，再初始化相关的配置
                .initialCacheNames(cacheNames)
                .withInitialCacheConfigurations(configMap)
                .build();
        return cacheManager;
    }
}
```
相比上节，我们多配置了一个缓存管理器，在`springboot2`中配置缓存管理是新的api也就是`builder`模式构建。然后通过`@EnableCaching` 开启缓存注解。然后需要注意的是 你在redistemplate中的配置的key，value序列化方法并不会生效，需要在`RedisCacheConfiguration`中单独配置。
## 3、使用caching注解
一般缓存在service层中使用。
代码如下
```java
/**
 * @author yukong
 * @date 2018/8/20 14:35
 * @description user业务层实现
 */
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;


    @Override
    public User saveUser(User user) {
        userMapper.save(user);
        // 返回用户信息，带id
        return user;
    }

    /**
     * @CacheEvict 应用到删除数据的方法上，调用方法时会从缓存中删除对应key的数据
     *      condition 与unless相反，只有表达式为真才会执行。
     * @param id 主键id
     * @return
     */
    @CacheEvict(value = "user", key = "#root.args[0]", condition = "#result eq true")
    @Override
    public Boolean removeUser(Long id) {
        // 如果删除记录不为1  则是失败
        return userMapper.deleteById(id) == 1;
    }

    /**
     *  @Cacheable 应用到读取数据的方法上，先从缓存中读取，如果没有再从DB获取数据，然后把数据添加到缓存中
     *            key 缓存在redis中的key
     *            unless 表示条件表达式成立的话不放入缓存
     * @param id 主键id
     * @return
     */
    @Cacheable(value = "user", key = "#root.args[0]", unless = "#result eq null ")
    @Override
    public User getById(Long id) {
        return userMapper.selectById(id);
    }

    /**
     *  @CachePut 应用到写数据的方法上，如新增/修改方法，调用方法时会自动把相应的数据放入缓存
     * @param user 用户信息
     * @return
     */
    @CachePut(value = "user", key = "#root.args[0]", unless = "#user eq null ")
    @Override
    public User updateUser(User user) {
        userMapper.update(user);
        return user;
    }
}
```

其中每个注解的作用都写在注释中了。
## 4、测试
然后我们编写一下对应的rest接口来测试
```java
/**
 * @author yukong
 * @date 2018/8/20 15:27
 * @description user控制器
 */
@RequestMapping("/user")
@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping
    public User save(@RequestBody User user) {
        return userService.saveUser(user);
    }

    @PutMapping
    public User update(@RequestBody User user) {
        return userService.updateUser(user);
    }

    @DeleteMapping(value = "/id/{id}")
    public Boolean delete(@PathVariable  Long id) {
        return userService.removeUser(id);
    }

    @GetMapping(value = "/id/{id}")
    public User findById (@PathVariable Long id) {
        return userService.getById(id);
    }

}
```
启动程序。首先访问`http://localhost:8989/user/id/2`
成功请求数据 并且控制台打印sql
![image.png](https://upload-images.jianshu.io/upload_images/5338436-fc93964402cd4bd5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
查看redis是否成功缓存数据。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-350d0d228bcdcf89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

成功缓存数据，并且key为主键id。
再次刷新访问。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-8e093b0c4abb2fd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
并没有打印sql，说明缓存生效，数据是从缓存中去的，而没有去访问数据库。
至于修改跟删除两个注解就交给大家测试。
关于本文，其中mybatis相关的代码并没有贴出，全部代码放在github上，地址如下。
最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。

