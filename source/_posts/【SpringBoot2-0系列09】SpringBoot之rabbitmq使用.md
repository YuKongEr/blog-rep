---
title: 【SpringBoot2.0系列09】SpringBoot之rabbitmq使用
date: 2018-08-22 21:44:02
categories: SpringBoot
tags: [SpringBoot, RabbitMQ]
---

>消息队列中间件是分布式系统中重要的组件，主要解决应用耦合，异步消息，流量削锋等问题实现高性能，高可用，可伸缩和最终一致性[架构](http://lib.csdn.net/base/architecture "大型网站架构知识库")
>使用较多的消息队列有ActiveMQ，RabbitMQ，ZeroMQ，Kafka，MetaMQ，RocketMQ。
>今天我们将会了解到在`SpringBoot`中使用`rabbitmq`
><!-- more -->
# 实现
## 1.1 rabbitmq简介
RabbitMQ是由Erlang语言编写的实现了高级消息队列协议（AMQP）的开源消息代理软件（也可称为 面向消息的中间件）。支持Windows、Linux/Unix、MAC OS X操作系统和包括JAVA在内的多种编程语言。



AMQP，即Advanced Message Queuing Protocol，一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受 客户端/中间件 不同产品，不同的开发语言等条件的限制
使用`rabbitmq`主要三种分发模式
### 1.1.1 工作队列模式(Work Queue)
避免立即做一个资源密集型任务，必须等待它完成，而是把这个任务安排到稍后再做。我们将任务封装为消息并将其发送给队列。后台运行的工作进程将弹出任务并最终执行作业。当有多个worker同时运行时，任务将在它们之间共享。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-ee11756e3992594e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 1.1.2 分发模式(Fanout Exchange)
一个生产者，多个消费者，每一个消费者都有自己的一个队列，生产者没有将消息直接发送到队列，而是发送到了交换机，每个队列绑定交换机，生产者发送的消息经过交换机，到达队列，实现一个消息被多个消费者获取的目的。需要注意的是，如果将消息发送到一个没有队列绑定的exchange上面，那么该消息将会丢失，这是因为在rabbitMQ中exchange不具备存储消息的能力，只有队列具备存储消息的能力。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-b80ea0afefc01404.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/5338436-c0cd12035e7f1243.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.1.3  通配符模式（Topic Exchange）
这种模式添加了一个路由键，生产者发布消息的时候添加路由键，消费者绑定队列到交换机时添加键值，这样就可以接收到需要接收的消息。
符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词
![image.png](https://upload-images.jianshu.io/upload_images/5338436-33fd48482df51d9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/5338436-90c336219bd05fab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.2、安装rabbitmq
### 1.2.1 window
因为`rabbitmq`是`erlang`实现，所以我们需要先下载安装`erlang`，然后再下载`rabbitmq`
### 1.2.2 mac
在mac系统中可以直接使用`brew`安装，它会帮我们自动安装管理依赖。
```bash
brew update
brew install rabbitmq
```
这样，我们就可以使用`rabbit-server`启动Rabbit服务了。
### 1.2.3 centos
在centos中可以使用`yum`安装
```bash
sudo yum install rabbitmq
```
## 1.3 springboot整合

 首先新建一个项目名为rabbit-producer 消息生产者工程
并且添加依赖。
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

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>

    </dependencies>
```
在yml配置rabbitmq地址
```yml
# rabbitmq配置
spring:
    rabbitmq:
      addresses: 127.0.0.1
      username: guest
      password: guest:
```

同理创建`rabbit-consumer` 消息消费者工程

### 1、普通工作队列模式
首先在`rabbit-producer`工程中新建`RabbitConfig`文件，用于配置我们rabbitmq相关的资源
代码如下
```java
package com.yukong.rabbitproducer;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author yukong
 * @date 2018/8/22
 * @description rabbitmq配置类
 */
@Configuration
public class RabbitConfig {

    /**
     * 定义队列名
     */
    private final static String STRING = "string";


    /**
     * 定义string队列
     * @return
     */
    @Bean
    public Queue string() {
        return new Queue(STRING);
    
}
```
定义了名为string的队列。然后我们创建生产者`RabbitProducer`
```
package com.yukong.rabbitproducer;

import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.xml.ws.Action;
import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * @author yukong
 * @date 2018/8/22
 * @description rabbit消息生产者
 */
@Component
public class RabbitProducer {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void stringSend() {
        Date date = new Date();
        String dateString = new SimpleDateFormat("YYYY-mm-DD hh:MM:ss").format(date);
        System.out.println("[string] send msg:" + dateString);  
      // 第一个参数为刚刚定义的队列名称
        this.rabbitTemplate.convertAndSend("string", dateString);
    }
}
```
这里注入一个`AmqpTemplate`来发布消息
接下来我们需要在`rabbit-consumer`工程配置一下消费者。
创建`StringConsumer`
```java
package com.yukong.rabbitmqconsumer;

import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * @author yukong
 * @date 2018/8/22
 * @description rabbitmq消费者 @RabbitListener(queues = "simpleMsg") 监听名simpleMsg的队列
 */
@Component
@RabbitListener(queues = "string")
public class StringConsumer {

    @Autowired private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[string] recieved message:" + msg);
    }

}
```
每一个注解的作用代码里面的注释说的很详细了我就不重复说了。
然后我们来测试，
首先在生产者工程新建一个测试类，用于生产消息。
代码如下
```java
package com.yukong.rabbitproducer;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class RabbitProducerApplicationTests {

    @Autowired
    private RabbitProducer producer;

    @Test
    public void testStringSend() {
        for (int i = 0; i < 10; i++) {
            producer.stringSend();
        }
    }

}
```
首先启动生产者工程的测试类。然后再启动消费者工程。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-821cc5478882228f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
消息生产成功，一共十条。
启动消费者工程。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-33df4b482743d068.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
消费者成功消费消息。
2、 fanout模式
fanout属于广播模式，只要跟它绑定的队列都会通知并且接受到消息。
我们同理在`RabbitConfig`中配置一下fanout模式的队列跟交换机。
```java
//=================== fanout 模式  ====================

    @Bean
    public Queue fanoutA() {
        return new Queue("fanout.a");
    }

    @Bean
    public Queue fanoutB() {
        return new Queue("fanout.b");
    }

    @Bean
    public Queue fanoutC() {
        return new Queue("fanout.c");
    }

    /**
     * 定义个fanout交换器
     * @return
     */
    @Bean
    FanoutExchange fanoutExchange() {
        // 定义一个名为fanoutExchange的fanout交换器
        return new FanoutExchange("fanoutExchange");
    }

    /**
     * 将定义的fanoutA队列与fanoutExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingExchangeWithA() {
        return BindingBuilder.bind(fanoutA()).to(fanoutExchange());
    }

    /**
     * 将定义的fanoutB队列与fanoutExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingExchangeWithB() {
        return BindingBuilder.bind(fanoutB()).to(fanoutExchange());
    }

    /**
     * 将定义的fanoutC队列与fanoutExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingExchangeWithC() {
        return BindingBuilder.bind(fanoutC()).to(fanoutExchange());
    }

```
在代码中我们配置了三个队列名、一个fanout交换机，并且将这三个队列绑定到了fanout交换器上。只要我们往这个交换机生产新的消息，那么这三个队列都会收到。
接下来，我们在`RabbitProducer` 中添加fanout的生产方法。
```java
public void fanoutSend() {
        Date date = new Date();
        String dateString = new SimpleDateFormat("YYYY-mm-DD hh:MM:ss").format(date);
        System.out.println("[fanout] send msg:" + dateString);
        // 注意 第一个参数是我们交换机的名称 ，第二个参数是routerKey 我们不用管空着就可以，第三个是你要发送的消息
        this.rabbitTemplate.convertAndSend("fanoutExchange", "", dateString);
    }
```
同理我们需要在消费者工程新建三个消费者的类
代码分别如下
```java
@Component
@RabbitListener(queues = "fanout.a")
public class FanoutAConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[fanout.a] recieved message:" + msg);
    }
}


```
```java
@Component
@RabbitListener(queues = "fanout.b")
public class FanoutBConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[fanout.b] recieved message:" + msg);
    }
}
```
```
@Component
@RabbitListener(queues = "fanout.c")
public class FanoutCConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[fanout.c] recieved message:" + msg);
    }
}
```

然后编写一个名为`testFanout()`的方法启动我们的fanout生产方法，
```
   @Test
    public void testFanoutSend() {
        producer.fanoutSend();
    }
```
![image.png](https://upload-images.jianshu.io/upload_images/5338436-bbb841164cda3703.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后重启消费者工程
![image.png](https://upload-images.jianshu.io/upload_images/5338436-0868d905a90408f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
三个队列的消费都成功接收到消息。
3、topic模式，
同样，配置topic队列跟交换器，注意的是这里需要多配置一个bindingKey
```java
 //#################topic模式########################

    @Bean
    public Queue topiocA() {
        return new Queue("topic.a");
    }

    @Bean
    public Queue topicB() {
        return new Queue("topic.b");
    }

    @Bean
    public Queue topicC() {
        return new Queue("topic.c");
    }

    /**
     * 定义个topic交换器
     * @return
     */
    @Bean
    TopicExchange topicExchange() {
        // 定义一个名为fanoutExchange的fanout交换器
        return new TopicExchange("topicExchange");
    }

    /**
     * 将定义的topicA队列与topicExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingTopicExchangeWithA() {
        return BindingBuilder.bind(topiocA()).to(topicExchange()).with("topic.msg");
    }

    /**
     * 将定义的topicB队列与topicExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingTopicExchangeWithB() {
        return BindingBuilder.bind(topicB()).to(topicExchange()).with("topic.#");
    }

    /**
     * 将定义的topicC队列与topicExchange交换机绑定
     * @return
     */
    @Bean
    public Binding bindingTopicExchangeWithC() {
        return BindingBuilder.bind(topicC()).to(topicExchange()).with("topic.*.z");
    }
```
- topicA的key为topic.msg 那么他只会接收包含topic.msg的消息
- topicB的key为topic.#那么他只会接收topic开头的消息
- topicC的key为topic.*.Z那么他只会接收topic.B.z这样格式的消息
  同理在`RabbitProducer`完成topic生产方法
```java
public void topicTopic1Send() {
        Date date = new Date();
        String dateString = new SimpleDateFormat("YYYY-mm-DD hh:MM:ss").format(date);
        dateString = "[topic.msg] send msg:" + dateString;
        System.out.println(dateString);
        // 注意 第一个参数是我们交换机的名称 ，第二个参数是routerKey topic.msg，第三个是你要发送的消息
        // 这条信息将会被 topic.a  topic.b接收
        this.rabbitTemplate.convertAndSend("topicExchange", "topic.msg", dateString);
    }

    public void topicTopic2Send() {
        Date date = new Date();
        String dateString = new SimpleDateFormat("YYYY-mm-DD hh:MM:ss").format(date);
        dateString = "[topic.good.msg] send msg:" + dateString;
        System.out.println(dateString);
        // 注意 第一个参数是我们交换机的名称 ，第二个参数是routerKey ，第三个是你要发送的消息
        // 这条信息将会被topic.b接收
        this.rabbitTemplate.convertAndSend("topicExchange", "topic.good.msg", dateString);
    }

    public void topicTopic3Send() {
        Date date = new Date();
        String dateString = new SimpleDateFormat("YYYY-mm-DD hh:MM:ss").format(date);
        dateString = "[topic.m.z] send msg:" + dateString;
        System.out.println(dateString);
        // 注意 第一个参数是我们交换机的名称 ，第二个参数是routerKey ，第三个是你要发送的消息
        // 这条信息将会被topic.b、topic.b接收
        this.rabbitTemplate.convertAndSend("topicExchange", "topic.m.z", dateString);
    }
```
然后在消费者工程新建队列队列的消费类
```java
@Component
@RabbitListener(queues = "topic.a")
public class TopicAConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[topic.a] recieved message:" + msg);
    }
}
```
```
@Component
@RabbitListener(queues = "topic.b")
public class TopicBConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[topic.b] recieved message:" + msg);
    }
}
```
```
@Component
@RabbitListener(queues = "topic.c")
public class TopicCConsumer {

    @Autowired
    private AmqpTemplate rabbitmqTemplate;

    /**
     * 消息消费
     * @RabbitHandler 代表此方法为接受到消息后的处理方法
     */
    @RabbitHandler
    public void recieved(String msg) {
        System.out.println("[topic.c] recieved message:" + msg);
    }
}
```
同理为topic新建测试方法
```java
 @Test
    public void testTopic() {
        producer.topicTopic1Send();
        producer.topicTopic2Send();
        producer.topicTopic3Send();
    }
```
![image.png](https://upload-images.jianshu.io/upload_images/5338436-6d3aa443c8c347f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
消息成功发出。
启动消费者工程，看看消息是不是按照规则被发送消息
![image.png](https://upload-images.jianshu.io/upload_images/5338436-a9c817ee93c8e8cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中 队列topic.a只配置topic.msg一条消息，正确
其中 队列topic.b匹配三条消息，因为三条消息都是topic开头的 正确
其中 队列topic.c匹配一条消息，只有一条消息满足（也就是topic.m.z这条消息）

最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。