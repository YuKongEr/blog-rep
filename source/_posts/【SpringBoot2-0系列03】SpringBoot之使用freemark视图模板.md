---
title: 【SpringBoot2.0系列03】SpringBoot之使用freemark视图模板
date: 2018-08-16 13:50:50
categories: SpringBoot
tags: [SpringBoot,FreeMark]
---
freemarker介绍；
       FreeMarker是一款模板引擎： 即一种基于模板和要改变的数据，   并用来生成输出文本（HTML网页、电子邮件、配置文件、源代码等）的通用工具。       它不是面向最终用户的，而是一个Java类库，是一款程序员可以嵌入他们所开发产品的组件。
前面我介绍了如何整合thymeleaf，那么现在我们再来了解一下`SpringBoot`中如何使用`freemark`
<!-- more -->

# 一、目标

使用`freemark`视图模板，并且于SpringBoot进行整合。 使用freemark显示用户（user）的信息

# 二、实现

首先创建一个SpringBoot项目，添加如下依赖

```xml
<dependencies>
        <!-- freemark -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>

        <!-- web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

添加完依赖以后，就需要编写对应的`Controller`跟`view`和`User.java`了
在`src/main/java/com/yukong/chapter22`目录下新建`User`

```java
package com.yukong.chapter22;

import java.util.Date;

/**
 * @Auther: xiongping22369
 * @Date: 2018/8/13 17:53
 * @Description: user类
 */
public class User {

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

}


```

编写`IndexController` 实现将`User`信息传递给前台ftl页面。
```java
@Controller
public class IndexController {

    @GetMapping("/aboutMe")
    public String index(Model model) throws ParseException {
        User user = new User();
        user.setUsername("yukong");
        user.setPassword("abc123");
        user.setAge(18);
        user.setSex(1);
        model.addAttribute("user", user);
        return "index";
    }

}
```
注意这里使用的是`@Controller`
在`resource`目录下新建`templates`文件夹并且在该目录下新建文件`index.ftl`
记住是`ftl` freemark文件的后缀名是`ftl`

`index.ftl`代码
```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>freemark</title>
</head>
<body>
    <p> 你好 ${user.username}</p>
    <p> 密码 ${user.password}</p>
    <p> 性别：
           <#if user.sex==1>
              男
           <#elseif user.sex==2>
              女
           <#else>
              保密
           </#if>
    </p>
    <p> 年龄 ${user.age}</p>
</body>
</html>
```
然后在`src/resource/application.yml`配置一下`thymeleaf`相关配置

```yml

server:
  port: 8989
spring:
  freemarker:
    request-context-attribute: req  #req访问request
    suffix: .ftl  #后缀名
    content-type: text/html
    enabled: true
    cache: false #缓存配置
    template-loader-path: classpath:/templates/ #模板加载路径 按需配置
    charset: UTF-8 #编码格式
    settings:
      number_format: '0.##'   #数字格式化，无小数点

```

启动`Chapter22Application.java`并且访问`http://localhost:8989/aboutMe`

结果如图
![image.png](https://upload-images.jianshu.io/upload_images/5338436-a9fa0d3452084893.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


由上图可知，`freemark`成功接受到了后台传递的数据。并且渲染到页面显示。

# 三、总结

此致我们SpringBoot整合freemark就完毕了。

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。
