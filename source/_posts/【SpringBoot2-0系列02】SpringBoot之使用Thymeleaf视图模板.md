---
title: 【SpringBoot2.0系列02】SpringBoot之使用Thymeleaf视图模板
date: 2018-08-15 10:20:51
categories: SpringBoot
tags: [SpringBoot,Thymeleaf]
---

Thymeleaf 是Java服务端的模板引擎，与传统的JSP不同，前者可以使用浏览器直接打开，因为可以忽略掉拓展属性，相当于打开原生页面，给前端人员也带来一定的便利。如果你已经厌倦了JSP+JSTL的组合，Thymeleaf或许是个不错的选择！ 
<!-- more -->

# 一、目标

使用`thymeleaf`视图模板，并且于SpringBoot进行整合。

# 二、实现

首先创建一个SpringBoot项目，添加如下依赖

```xml
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
        <!-- springboot web 依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- thymeleaf 依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
    </dependencies>
```

添加完依赖以后，就需要编写对应的`Controller`跟`view`了

在`resource`目录下新建`templates`文件夹并且在该目录下新建文件`index.html`



```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>你好</h1>
    <h1 th:text="${name}">  </h1>
</body>
</html>
```

在`src/main/java/com/yukong/chapter`目录下新建`IndexController`

```java
@Controller
public class IndexController {

    @GetMapping("/hello")
    public String hello(@RequestParam(defaultValue = "world", required = false) String name, Model model) {
        model.addAttribute("name", name);
        return "index";
    }

}

```

注意这里使用的是`@Controller`

然后在`src/resource/application.yml`配置一下`thymeleaf`相关配置

```yml

server:
  port: 8989
spring:
  thymeleaf:
    # 配置视图路径前缀
    prefix: classpath:/templates/
    # 配置视图路径后缀
    suffix: .html
    mode: html
    # 关闭缓存 修改视图 刷新浏览器就显示 开发阶段务必关闭缓存 (=false)
    cache: false

```

启动`Chapter21Application.java`并且访问`http://localhost:8989/hello`

结果如图：

![image.png](https://upload-images.jianshu.io/upload_images/5338436-e60dc8f2cae118dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


再次访问`http://localhost:8989/hello?name=yukong`

结果如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-5fef51e99ae90f3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# 三、总结


最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。