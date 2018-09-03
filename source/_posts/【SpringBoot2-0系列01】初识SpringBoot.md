---
title: 【SpringBoot2.0系列01】初识SpringBoot
date: 2018-08-14 00:10:06
categories:
    SpringBoot
tags: [SpringBoot]
---
想必大家都一定用过spring框架，每次整合spring框架的时候总是会有无穷无尽的xml配置文件，第一次写配置文件的时候，大家还会抱着学习的心态认真读每一个配置，但是当我们每次在构建项目都要写同样的配置文件大家应该会觉得厌烦，尽管只是复制粘贴。那么现在你就不用担心了，使用springboot让你更简单的构建spring应用。
<!-- more -->
# 一、介绍


springboot让我们更加简单快速的构建spring应用，并且内置web容器（tomcat、jetty等）支持jar包的方式启动一个web应用。

#### SpringBoot主要优点：

* 1. 为所有Spring开发者更快的入门
* 2. 开箱即用，提供各种默认配置来简化项目配置
* 3. 内嵌式容器简化Web项目
* 4. 没有冗余代码生成和XML配置的要求
	



# 二、目标

本节主要目标是通过构建一个简单的SpringBoot项目，实现一个hello word接口。通过这样一个简单的例子让我们对springboot有一个了解。

# 三、实现

## 3.1、环境

* java8 及以上
* SpringBoot 2.0.4

 ## 3.2 构建项目

### 3.2.1 spring官网 SPRING INITIALIZR 

1、访问 `http://start.spring.io/ `

2、选择`maven`构建、使用`java`语法、并且`SpringBoot`版本为2.0.4

具体如图：

![image.png](https://upload-images.jianshu.io/upload_images/5338436-d2c142a6633ec39a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3、点击`Generate Project ` 下载构建成功的项目压缩包

解压完成之后我们就可以通过IntelliJ IDEA开发工具导入到工程，

* 1. 菜单中选择File–>New–>Project from Existing Sources...

* 2. 选择解压后的项目文件夹，点击OK

* 3. 点击Import project from external model并选择Maven，点击Next到底为止。

* 4. 如果你的环境有多个版本的JDK，注意到选择Java SDK的时候请选择系统安装1.8版本

 ### 3.2.2 使用idea构建
   1、打开idea选择`new` -> `project` -> `SPRING INITIALIZR `
![idea构建](https://upload-images.jianshu.io/upload_images/5338436-190d26d4b8e134f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、在ProjectSDK选择`jdk8`
3、点击`next` 输入`gourpId artifactId` 然后继续`next` 一直到完成就可以
![image.png](https://upload-images.jianshu.io/upload_images/5338436-305f00648d01bedc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其实大家看的出来，两种方式都是一样的，只不过用idea省去了我们手动下载介绍并且导入的步骤，稍显方便。
### 3.2.3、项目结构

![image.png](https://upload-images.jianshu.io/upload_images/5338436-cea92176799a5647.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关键目录解释
 * 1、`/src/main/java/`  存放项目所有源代码目录
* 2、`/src/main/resources/ ` 存放项目所有资源文件以及配置文件目录
* 3、`/src/test/ ` 存放测试代码目录

其中生成的Chapter1Application和Chapter1ApplicationTests类都可以直接运行来启动当前创建的项目。



## 3.3、添加web模块

打开pom.xml  添加spring-boot-starter-web 即可

```java
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

## 3.4、编写hello world服务

在`src/main/java`目录下的`chapter1`包下新建一个`HelloWorldController.java`文件

```java
@RestController
@RequestMapping("/hello")
public class HelloWorldController {

    @GetMapping
    public String hello() {
        return "hello world";
    }
}
```

其中

* `@RequestController`是一个复合注解 ，大家按住ctrl看源码会发现 `@RequestController = @Controller +@ResponseBody` 那么它的作用就不言而喻了，代表当前类是一个控制器并且返回所有的方法将会返回`json`数据

## 3.5 、启动

双击打开`Chapter1Application`文件启动。

![image.png](https://upload-images.jianshu.io/upload_images/5338436-9bea1db3ec2ac607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以看出日志打印启动tomcat服务的端口是8080 ，代表启动成功。

打开浏览器访问`http://localhost:8080/hello`

![image.png](https://upload-images.jianshu.io/upload_images/5338436-dc90c849f6a2ab3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么有时候我们的8080端口被占用了，那么就会启动失败，提示端口被占用，如何改变`springboot`启动的默认端口呢？

打开`application.yml`  这时候大家就会问了 不是`application.properties`吗 其实yml文件也是配置文件的一种，它的书写简洁，我比较喜欢。

`yml写法`

```yml
server:
  port: 9090
```

`properties写法`

```properties
server.port=9090
```

上面的代码就是设置web服务启动的端口了。



# 四、总结

通过这次学习，我们了解了`springboot`如何启动一个`web`服务，并且如何更改`web`服务启动的默认端口。

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。
