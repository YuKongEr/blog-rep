---
title: 【SpringBoot2.0系列04】SpringBoot之使用JPA完成简单的restapi
date: 2018-08-17 11:59:28
categories: SpringBoot
tags: [SpringBoot,SpringDataJpa, Rest]
---
在前面我们已经知道在springboot中如何使用`freemark`与`thymeleaf`之类的视图模板引擎去渲染我们的视图页面，但是没涉及跟数据库交互的东西，所以今天在这里我们将介绍了一下如何在`springboot `中通过`spring data jpa`操作`mysql`数据库，并且构建一套简单的`rest api`接口。
<!-- more -->
# 一、前言
 ## 1.1、Spring Data Jpa 介绍
  Spring Data JPA是Spring基于Hibernate开发的一个JPA框架。如果用过Hibernate或者MyBatis的话，就会知道对象关系映射（ORM）框架有多么方便。但是Spring Data JPA框架功能更进一步，为我们做了 一个数据持久层框架几乎能做的任何事情。并且提供了基础的增删改查方法，具体api请看[官网](https://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/)。
## 2.2 
REST是所有Web应用都应该遵守的架构设计指导原则。 
Representational State Transfer，翻译是”表现层状态转化”。 
面向资源是REST最明显的特征，对于同一个资源的一组不同的操作。资源是服务器上一个可命名的抽象概念，资源是以名词为核心来组织的，首先关注的是名词。REST要求，必须通过统一的接口来对资源执行各种操作。对于每个资源只能执行一组有限的操作。（7个HTTP方法：GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS）
关于`rest api`如何涉及我也是从[阮一峰](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)老师那里学习的。

# 二、目标
首先我们有一个`user`表，我们希望能通过构建出对应的`rest api`对表中的数据完成增删改查操作。
jpa的依赖如下
```xml<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
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
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
    </dependencies>
```
接下来那么第一步就是创表了
1、创表
 由于我们使用的spring data  jpa 而jpa的底层实现是hibernate，用过hibernate的同学知道 hibernate可以通过实体类逆向创建表，只需要配置一下`ddl-auto` 就可以
所以我们需要在`application.yml`配置一下
```yml
server:
  port: 8989
spring:
  datasource:
    username: root
    password: root
    url: jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf8
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```
在上面的配置文件中 我们配置了一个数据源跟tomcat端口还有jpa的配置。其中 `show-sql: true` 代表会在日志中打印我们操作的sql、
而另外 `ddl-auto`有四个值可选，分别是
* `create` 启动时删数据库中的表，然后创建，退出时不删除数据表 
* `create-drop` 启动时删数据库中的表，然后创建，退出时删除数据表 如果表不存在报错 
* `update` 最常用的属性，第一次加载hibernate时根据model类会自动建立起表的结构（前提是先建立好数据库），以后加载hibernate时根据 model类自动更新表结构，即使表结构改变了但表中的行仍然存在不会删除以前的行。要注意的是当部署到服务器后，表结构是不会被马上建立起来的，是要等 应用第一次运行起来后才会。
* `validate` 项目启动表结构进行校验 如果不一致则报错
***
所以这里我们希望当表创建成功后  下次启动数据还在我们就选择了`update`模式，其次我们需要在本地的`mysql`数据库新建一个`test`数据库。
接下来我们需要编写的我们实体类`User.java`了 `hibernate`将会通过实体类的结构在`test`数据库中创建一个对应的`user`表
新建包`entity` 创建`User.java`代码如下：
```java
@JsonIgnoreProperties(value={"hibernateLazyInitializer","handler","fieldHandler"}) 
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    /**
     * 用户名
     */
    @Column(name = "username", nullable = true, length = 32)
    private String username;

    /**
     * 密码
     */
    @Column(name = "password", nullable = true, length = 32)
    private String password;

    /**
     * 年龄
     */
    @Column(name = "age", nullable = true, length = 11)
    private Integer age;

    /**
     * 性别 1=男 2=女 其他=保密
     */
    @Column(name = "sex", nullable = true, length = 11)
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
至此，我们的user表也算是创建好了，当我们的程序第一个启动的时候`jpa`会自动在`test`数据库中创建与之对应的表。
***
2、构建rest api
这里我们需要构建如下的rest api
url|method|介绍
:--:|:--|--:        
`/user/`|`get`|获取所有的用户信息
`/user/id/{id}`|`get`|根据id获取用户信息
`/user/username/{username}`|`get`|根据username获取用户信息
`/user`|`post`|新增用户信息
`/user`|`put`|更新用户信息
`/user/id/{id}`|`delete`|根据id删除用户信息

那么这就是我们需要构建的`rest api`，那么对应的由`mvc`模式可知我们的`rest api`是`controller`层的，所以我们的`service`跟`repository`层(备注在使用 jpa的时候我们喜欢把dao层命名为repository)需要提供对应的接口。
首先我们首先需要编写`UserRepository`接口，并且让它基础`JpaRepository`接口

新建包`repository` 创建`UserRepository.java`代码如下：
```java
public interface UserRepository extends JpaRepository<User, Long> {

    /**
     * 根据用户名查找用户信息
     * @param username
     * @return
     */
    User findUserByUsername(String username);

}
```
解释一下上面的代码，为什么只有一个方法，而前面我们是五个接口，因为是在JpaRepository中提供较为基础的增删改查方法，我们无需编写就看使用。如果大家不信按住`ctrl`点击JpaRepository看源码就知道了。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-722476082a4ab89f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从上面就可以看出JpaRespository提供了哪些基础方法了。怎么样 是不是觉得很方便。那么接下来的第二点就Jpa可以根据你的命名规则来推断你这个方法作用，简单的来说`findUserByUsername` 根据这个方法名，jpa可以知道这个方法是通过用户名去查找用户。 具体的规则大家可以看[文档](https://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/#jpa.query-methods)
![image.png](https://upload-images.jianshu.io/upload_images/5338436-4668bb4e12afbdd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果大家用的idea的话，那么它会用智能提示功能，如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-b07f8706d9b082d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以我们只需要编写方法名就可以轻轻松松的实现我们的查询方法，怎么样jpa是不是特别简单，但是需要注意的是方法名一定要命名规范，不要嫌太长了。
接下来就是编写我们的service层了，
新建`service`包创建`UserService.java`代码如下：
```java
public interface UserService {

    /**
     * 添加用户信息
     * @param user
     * @return
     */
    User saveUser(User user);

    /**
     * 更新用户信息
     * @param user
     */
    User updateUser(User user);

    /**
     * 根据id获取用户
     * @param id
     * @return
     */
    User getById(Long id);

    /**
     * 根据名称获取用户
     * @param username
     * @return
     */
    User getByUserName(String username);

    /**
     * 查询所有用户
     * @return
     */
    List<User> queryAll();

    /**
     * 根据id删除用户信息
     * @param id
     */
    void deleteById(Long id);
}
```
接下里就下service的实现类，带service包下新建`impl`包并且创建实现类`UserServiceImpl.java` 代码如下
```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public User saveUser(User user) {
        return userRepository.save(user);
    }

    @Override
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    @Override
    public User getById(Long id) {
        return userRepository.getOne(id);
    }

    @Override
    public User getByUserName(String username) {
        return userRepository.findUserByUsername(username);
    }

    @Override
    public List<User> queryAll() {
        return userRepository.findAll();
    }

    @Override
    public void deleteById(Long id) {
        userRepository.deleteById(id);
    }
}
```
那么紧接着就是控制层的代码了,新建`controller`包，并且创建`UserController.java`
```java
@RestController
@RequestMapping(value = "/user")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping
    public User save(@RequestBody User user) {
        return userService.saveUser(user);
    }

    @PutMapping
    public User update(@RequestBody User user) {
        return userService.saveUser(user);
    }

    @DeleteMapping(value = "/id/{id}")
    public String delete(@PathVariable  Long id) {
        userService.deleteById(id);
        return "删除成功";
    }

    @GetMapping(value = "/id/{id}")
    public User findById (@PathVariable Long id) {
        return userService.getById(id);
    }

    @GetMapping(value = "/username/{username}")
    public User findByUsername (@PathVariable String username) {
        return userService.getByUserName(username);
    }

    @GetMapping(value = "/")
    public List<User> findAll () {
        return userService.queryAll();
    }
}
```
这样我们就完成了一个简单的`rest api`了啊。
接下来我们来测试一下把。
3、测试
由于我们这里测试的是rest api普通的浏览器是没法支持 `post delet put`方式的访问的，所以这里我们就用`postman`来测试。
1、首先我们看到test数据库中现在是一张表都没有的
![image.png](https://upload-images.jianshu.io/upload_images/5338436-8f6f51c83301ae86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
启动程序，注意观察日志。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-90917244173e9927.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到了日志打印了创建表的`ddl`那么我们再看看数据库中有没有表
![image.png](https://upload-images.jianshu.io/upload_images/5338436-e1bc7ae17a1830b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时我们看到了有两张表，一张是我们user表，而另一张就是主键生成序列表。
接下来就开始我们的`rest api`测试了。
首先测试新增用户
打开postman

![image.png](https://upload-images.jianshu.io/upload_images/5338436-4487994d5bab19f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
选择post模式，输入访问的url，然后选择body中的raw，因为我们使用的@RequestBody注解，所以我们选择raw中的Json，如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-ccb86fab94c7582a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
因为我们的id是自增的，所以我们不要输入，直接点击send访问，如果返回的数据有id那么就是代表新增成功。如下图就是新增成功。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-5c41e9025ec10aa4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
打开数据库中的user表，看看数据有没有保存成功。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-9e610eae5f69ee6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由图可知，保存成功。
接下来我们就多添加几条数据。
那么我们测试一下查询所有数据的方法。操作如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-57914782a7aed98b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们刚刚一共添加三条数据，全部都查询出来了。
我们继续测试一下修改方法把。我们把id为2的数据密码修改为跟用户名一样，具体操作如图，
![image.png](https://upload-images.jianshu.io/upload_images/5338436-8b657e817c2c6831.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击send操作成功，我们用根据id查询的方法来查询一下刚刚id为2的数据有没有修改成功，那么我们查询一下id为2的数据，操作如图。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-37a37a25f1f8fe05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由图中可以看到我们的修改是成功的，用户名跟密码已经一样的，那么代表我们的根据id查询方法也是没问题的。那么另外几个方法我们不测试了，留给大家测试。

# 三、总结
这里我们通过这次选择对于jpa的使用有了一个初步的了解，并且对于rest api的规范也有了个了解。

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。
