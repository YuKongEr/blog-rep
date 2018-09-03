---
title: 【SpringBoot2.0系列10】SpringBoot之@Scheduled任务调度
date: 2018-08-27 20:24:29
categories: SpringBoot
tags: [SpringBoot, Scheduled]
---

相信大家在实际工作场景中会遇到这样的情况，系统之间存在数据交换，为了不影响正常服务器运，我们需要在每天的凌晨来进行数据交换，但是让程序每天凌晨自动执行呢，下面带大家来了解一下springboot定时任务调度。
<!-- more-->



# 实现
其实在springboot中实现定时任务调度十分的，下面我们将实现一个简单的定时任务调度调度。
## 1、依赖
`scheduled` 依赖是`spring-context`这个jar包其中我们的`spring-boot-starter`已经依赖spring的一些核心jar，所以我们只需要添加`spring-boot-starter`即可
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
    </dependencies>
```
## 2、引入`@EnableScheduling`
我们需要在Spring Boot的主类中加入@EnableScheduling注解，开启定时任务的配置
```java
/**
 * @author yukong
 * @data 2018年8月27日19:00:21
 */
@EnableScheduling
@SpringBootApplication
public class Chapter9Application {

    public static void main(String[] args) {
        SpringApplication.run(Chapter9Application.class, args);
    }
}
```
##  3、实现定时任务
我们创建一个名为`ScheduledTask`的任务类
### 3.1.1 @Scheduled(fixedDelay= 5000)
@Scheduled(fixedDelay= 5000) 指的是上一次开始执行时间点之后5秒再执行。可能大家不知道什么意思那么下面代码编写，运行一下就知道了。
```java
/**
 * @author yukong
 * @date 2018/8/27
 * @description
 */
@Component
public class ScheduledTask {

    Logger logger = LoggerFactory.getLogger(ScheduledTask.class);

    private static final SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");


    /**
     * 从当前方法开始执行后5s再次执行
     */
    @Scheduled(fixedDelay= 5000)
    public void scheduledTask2() throws InterruptedException {
        logger.info("当前时间为：{}", simpleDateFormat.format(new Date()));
        Thread.sleep(3000L);
    }
}
```
上面需要注意的是我们通过`@Scheduled`注解表示这个一个定时调度的任务，具体的调度策略是根据注解中的属性决定，在当前代码中fixedDelay= 5000代表从当前方法开始执行**完成**后5s再次执行,注意加粗部分。另外需要注意的我们的类需要用`@Component`注解标识，不然spring是无法感知这些定时任务的。
### 3.1.2测试、结论
运行结果如下
![image.png](https://upload-images.jianshu.io/upload_images/5338436-38b5ebaaeba91759.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中是每隔8s执行一次，但是我们明明设置的5s的间隔，这是怎么回事呢。回头看看我刚刚说的`fixedDelay = 5000`的特点：代表从当前方法开始执行**完成**后5s再次执行。在看看定时调用的方法中`Thread.sleep(3000)`就瞬间明白了。原来`fixedDelay ` = 代表从当前方法开始执行**完成**后间隔一定时间再次执行。那么不需要等待当前方法执行完成又是怎么写的呢？
### 3.2.1 @Scheduled(fixedRate= 5000)
其实很简单的，我们只需要将`fixedDelay = 5000`改成`fixedRate = 5000`即可。
`fixedRate = 5000`代表 从当前方**开始执后**5s再次执行，代码如下：
```java
    /**
     * 从当前方法开始执行后5s再次执行
     */
    @Scheduled(fixedRate = 5000)
    public void scheduledTask1() throws InterruptedException {
        logger.info("当前时间为：{}", simpleDateFormat.format(new Date()));
        Thread.sleep(3000L);
    }
```
### 3.2.2测试、结论
执行结果如图，如预期的一样每隔5s秒执行一次。
![image.png](https://upload-images.jianshu.io/upload_images/5338436-571fa0289ba25e0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3.1      @Scheduled(cron = "0,5,15 * * * * ?")
如果你还需要更复杂的定时任务策略，那么你就可能需要用到`cron`表达式。
1.cron表达式格式：
{秒数} {分钟} {小时} {日期} {月份} {星期} {年份(可为空)}
2.cron表达式各占位符解释：
{秒数} ==> 允许值范围: 0~59 ,不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每隔1秒钟触发；
"," 代表在指定的秒数触发，比如"0,15,45"代表0秒、15秒和45秒时触发任务
"-"代表在指定的范围内触发，比如"25-45"代表从25秒开始触发到45秒结束触发，每隔1秒触发1次
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"0")，后面的值代表偏移量，比如"0/20"或者"\*/20"代表从0秒钟开始，每隔20秒钟触发1次，即0秒触发1次，20秒触发1次，40秒触发1次；"5/20"代表5秒触发1次，25秒触发1次，45秒触发1次；"10-45/20"代表在[10,45]内步进20秒命中的时间点触发，即10秒触发1次，30秒触发1次
{分钟} ==> 允许值范围: 0~59 ,不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每隔1分钟触发；
","代表在指定的分钟触发，比如"10,20,40"代表10分钟、20分钟和40分钟时触发任务
"-" 代表在指定的范围内触发，比如"5-30"代表从5分钟开始触发到30分钟结束触 发，每隔1分钟触发
"/"代表触发步进(step)，"/"前面的值代表初始值(\"*"等同"0")，后面的值代表偏移量，比如"0/25"或者"\*/25"代表从0分钟开始，每隔25分钟触发1次，即0分钟触发1次，第25分钟触发1次，第50分钟触发1次；"5/25"代表5分钟触发1次，30分钟触发1次，55分钟触发1次；"10-45/20"代表在[10,45]内步进20分钟命中的时间点触发，即10分钟触发1次，30分钟触发1次
{小时} ==> 允许值范围: 0~23 ,不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每隔1小时触发；
","代表在指定的时间点触发，比如"10,20,23"代表10点钟、20点钟和23点触发任务
"-"代表在指定的时间段内触发，比如"20-23"代表从20点开始触发到23点结束触发，每隔1小时触发
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"0")，后面的值代表偏移量，比如"0/1"或者"\*/1"代表从0点开始触发，每隔1小时触发1次；"1/2"代表从1点开始触发，以后每隔2小时触发一次；"19-20/2"表达式将只在19点触发
{日期} ==> 允许值范围: 1~31 ,不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每天触发；
"?"与{星期}互斥，即意味着若明确指定{星期}触发，则表示{日期}无意义，以免引起 冲突和混乱
"," 代表在指定的日期触发，比如"1,10,20"代表1号、10号和20号这3天触发
"-"代表在指定的日期范围内触发，比如"10-15"代表从10号开始触发到15号结束触发，每隔1天触发
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"1")，后面的值代表偏移量，比如"1/5"或者"\*/5"代表从1号开始触发，每隔5天触发1次；"10/5"代表从10号开始触发，以后每隔5天触发一次；"1-10/2"表达式意味着在[1,10]范围内，每隔2天触发，即1号，3号，5号，7号，9号触发
"L" 如果{日期}占位符如果是"L"，即意味着当月的最后一天触发
"W "意味着在本月内离当天最近的工作日触发，所谓最近工作日，即当天到工作日的前后最短距离，如果当天即为工作日，则距离为0；所谓本月内的说法，就是不能跨月取到最近工作日，即使前/后月份的最后一天/第一天确实满足最近工作日；因此，"LW"则意味着本月的最后一个工作日触发，"W"强烈依赖{月份}
"C" 根据日历触发，由于使用较少，暂时不做解释
{月份} ==> 允许值范围: 1~12 (JAN-DEC),不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每个月都触发；
"," 代表在指定的月份触发，比如"1,6,12"代表1月份、6月份和12月份触发任务
"-"代表在指定的月份范围内触发，比如"1-6"代表从1月份开始触发到6月份结束触发，每隔1个月触发
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"1")，后面的值代表偏移量，比如"1/2"或者"\*/2"代表从1月份开始触发，每隔2个月触发1次；"6/6"代表从6月份开始触发，以后每隔6个月触发一次；"1-6/12"表达式意味着每年1月份触发
{星期} ==> 允许值范围: 1~7 (SUN-SAT),1代表星期天(一星期的第一天)，以此类推，7代表星期六(一星期的最后一天)，不允许为空值，若值不合法，调度器将抛出SchedulerException异常
"\*" 代表每星期都触发；
"?"与{日期}互斥，即意味着若明确指定{日期}触发，则表示{星期}无意义，以免引起冲突和混乱
"," 代表在指定的星期约定触发，比如"1,3,5"代表星期天、星期二和星期四触发
"-"代表在指定的星期范围内触发，比如"2-4"代表从星期一开始触发到星期三结束触发，每隔1天触发
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"1")，后面的值代表偏移量，比如"1/3"或者"\*/3"代表从星期天开始触发，每隔3天触发1次；"1-5/2"表达式意味着在[1,5]范围内，每隔2天触发，即星期天、星期二、星期四触发
"L"如果{星期}占位符如果是"L"，即意味着星期的的最后一天触发，即星期六触发，L= 7或者 L = SAT，因此，"5L"意味着一个月的最后一个星期四触发
"#"用来指定具体的周数，"#"前面代表星期，"#"后面代表本月第几周，比如"2#2"表示本月第二周的星期一，"5#3"表示本月第三周的星期四，因此，"5L"这种形式只不过是"#"的特殊形式而已
"C" 根据日历触发，由于使用较少，暂时不做解释
{年份} ==> 允许值范围: 1970~2099 ,允许为空，若值不合法，调度器将抛出SchedulerException异常
"\*"代表每年都触发；
","代表在指定的年份才触发，比如"2011,2012,2013"代表2011年、2012年和2013年触发任务
"-"代表在指定的年份范围内触发，比如"2011-2020"代表从2011年开始触发到2020年结束触发，每隔1年触发
"/"代表触发步进(step)，"/"前面的值代表初始值("\*"等同"1970")，后面的值代表偏移量，比如"2011/2"或者"\*/2"代表从2011年开始触发，每隔2年触发1次
注意：除了{日期}和{星期}可以使用"?"来实现互斥，表达无意义的信息之外，其他占位符都要具有具体的时间含义，且依赖关系为：年->月->日期(星期)->小时->分钟->秒数
具体cron你可以参考[cron详解](https://blog.csdn.net/li295214001/article/details/52065634)


现在我们实现一个每分钟的第0,5,15秒执行一次，那么对应的cron表达式应为`0,5,15 * * * * ?`
实现的代码如下
```java
@Scheduled(cron = "0,5,15 * * * * ?")
    public void cronTask() {
        logger.info("当前时间为：{}", simpleDateFormat.format(new Date()));
    }
```

### 3.3.2运行
启动，运行结果如下：
![image.png](https://upload-images.jianshu.io/upload_images/5338436-ebd7c85adc5dcd6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
每分钟的第0,5,15秒都执行，如果你还需要其他的规则，只需要更改对应cron表达式，相信cron的强大能够满足所有的业务场景。


# 结语
相信通过本次学习，大家应该知道如何在springboot使用定时任务了。
最后配套教程的代码全部在这里
[github https://github.com/YuKongEr/SpringBoot-Study](https://github.com/YuKongEr/SpringBoot-Study)。麻烦点个star或者fork吧。



