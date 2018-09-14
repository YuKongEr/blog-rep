---
title: 【SpringBoot2.0系列11】SpringBoot之@Elasticsearch完成CURD
date: 2018-09-04 18:37:47
tags: [SpringBoot, ElasticSearch]
categories: SpringBoot
---

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于[云计算](https://baike.baidu.com/item/%E4%BA%91%E8%AE%A1%E7%AE%97/9969353)中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。
<!-- more-->
如果在springboot使用Easticsearch呢。在这里我们使用spring-boot-starter-data-elasticsearch。
它提供一系列简单的api给我们使用，让我们有种操作关系数据库的感觉。
好了话不多说，先说一下环境。
- spring boot2.x
- jdk8
- elasticsearch5.x(6.x也可以)

## 依赖
引入依赖
```xml
  <dependencies>

        <!-- lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.22</version>
        </dependency>
        <!-- elasticsearch -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>

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

在这里我们分别引入elasticsearch跟lombok的依赖，关于lombok的介绍大家可以看看[这篇文章](https://blog.csdn.net/motui/article/details/79012846) 讲的很详细。我这简单的介绍一下在项目中使用Lombok可以减少很多重复代码的书写。比如说getter/setter/toString等方法的编写。

## 配置es地址
在下文中我将用es代替elasticsearch。我们打开application.yml文件 配置如下
```yml
  spring:
  data:
    elasticsearch:
      # 集群的名字
      cluster-name: wali
      # 节点的ip与端口 注意端口是9300不是9200
      cluster-nodes: 127.0.0.1:9300
```
## 构建文档对象
假设这是一个商品索引goods，他有一个类型是电脑computer。
分别有四个字段

- id  唯一标识
- name 商品名称
- number 商品数量
- desc 商品具体描述
  我们根据上面的描述，编写出对应的实体类
```java
@Data
@ToString
@Accessors(chain = true)
@Document(indexName = "goods", type = "computer")
public class Good {

    /**
     * 主键,注意这个搜索是id类型是string，与我们常用的不同
     * Id注解加上后，在Elasticsearch里相应于该列就是主键了，在查询时就可以直接用主键查询
     */
    @Id
    private String id;

    private String name;

    private String desc;

    private Integer number;

}
```
其中`@Data
@ToString
@Accessors(chain = true)` 是属于lombok注解。
- @Data 会自动上传get/set方法
- @ToString 会生成tostring方法
- @Accessors(chain = true) 会让我们set方法可以链式调用
  @Document注解

`@Document`注解里面的几个属性，类比mysql的话是这样： 
indexName –> 索引库的名称，建议以项目的名称命名，就相当于数据库DB
type –> 类型，建议以实体的名称命名Table ，就相当于数据库中的表table
Document –> row 就相当于某一个具体对象

## jpa构建文档库
接着，我们可以通过jpa构建文档库，来操作我们的goods对应的文档。
因为我们引入的是spring data的elasticsearch所以它遵循spring data的接口，也就是说操作elasticSearch与操作spring data jpa的方法是完全一样的，我们只将文档库继承ElasticsearchRepository即可。
```java
public interface GoodRepository extends ElasticsearchRepository<Good, String> {

   /**
     * 根据商品名称查询 分页
     * @param name
     * @param pageable
     * @return
     */
    Page<Good> findByName(String name, Pageable pageable);

}
```
然后创建对应的测试类。前面说过快捷键`ctrl+shift+t`
并且编写测试方法，我们分别需要测试添加 删除 修改  查询 分页查询方法。
```java
@Component
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class GoodRepositoryTest extends Chapter10ApplicationTests {

    @Autowired
    private GoodRepository goodRepository;
    
    @Test
    public void save(){}

    @Test
    public void update(){}

    @Test
    public void select(){}

    @Test
    public void delete(){}

    @Test
    public void findByName() {
       
    }
}
```
上面代码我们是通过基础`主测试类`然后使用`@Component`注解就可以，这样就不需要每个测试都要`@SpringTest注解与@RunWith注解了`
另外`@FixMethodOrder(MethodSorters.NAME_ASCENDING)
`这个注解是表示按照方法名的顺序来排序，不然它不会按照我们方法书写的顺序执行，那么就有可能导致，还没save就select，这样就会失败了。
接下来继续编写方法体。`goodRepository`跟我们直接`data-jpa`的`respository`用法基本一致。都有继承`save,delete,find`方法的。
```java
@Test
    public void save(){
        Good good = new Good();
        good.setId("100")
                .setName("联想e541")
                .setDesc("联想e系列")
                .setNumber(10);
        Good result = goodRepository.save(good);
        Assert.assertNotNull(result);
    }

    @Test
    public void select(){
        // 需要注意find方法返回的死Optional对象 需要调用get方法返回具体的实体类对象
        Good result = goodRepository.findById("100").get();
        Assert.assertNotNull(result);

    }

    @Test
    public void update(){
        Good result = goodRepository.findById("100").get();
        result.setNumber(300);
        // 更新也是调用save方法
        Good good = goodRepository.save(result);
        Assert.assertNotNull(good);

    }



    @Test
    public void delete(){
        goodRepository.deleteById("100");
    }
```
我们首先测试增删改查方法。并且通过Assert断言来判断。

![image.png](https://upload-images.jianshu.io/upload_images/5338436-7cfb53c208dcaab7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
测试通过，
接下来我测试一下分页查询方法，首页我们看一下es中、goods索引computer类别下有哪些文档。
```bash
curl -XGET  'http://127.0.0.1:9200/goods/computer/_search?pretty'
```

结果如下
```json
{
  "took" : 21,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "goods",
        "_type" : "computer",
        "_id" : "_search",
        "_score" : 1.0,
        "_source" : {
          "id" : "_search",
          "name" : "macbook",
          "number" : 20,
          "desc" : "macbook air"
        }
      },
      {
        "_index" : "goods",
        "_type" : "computer",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "id" : "2",
          "name" : "think pad",
          "number" : 20,
          "desc" : "联想旗下thinkpad系列"
        }
      },
      {
        "_index" : "goods",
        "_type" : "computer",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "id" : "1",
          "name" : "macbook",
          "number" : 20,
          "desc" : "macbook pro"
        }
      }
    ]
  }
}
```
我看到在computer类别中存有三条文档，name分别是 `macbook ` `think pad` `macbook`,所以我们查询一下name=macbook的文档，pageSize=1,pageNum=1.
```java
@Test
    public void findByName() {
        String name = "macbook";
        Pageable pageable = new PageRequest(1,1);
        Page<Good> goods = goodRepository.findByName(name, pageable);
        System.out.println(goods.getContent());
        Assert.assertEquals(1, goods.getSize());

    }
```
我们查询name为macbook的数据，并且限制每页一条，所以我们查询的结果总数应该是一条。结果如下。
![](https://upload-images.jianshu.io/upload_images/5338436-f6e5193ac482182a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这节，我们了解了springboot与es的curd操作，都是比较简单的，那么下节我们会详细了解springboot对es如何进行复杂查询，与聚合查询。
最后本节的配套代码地址为[https://github.com/YuKongEr/SpringBoot-Study/tree/master/chapter10](https://github.com/YuKongEr/SpringBoot-Study/tree/master/chapter10)


最后欢迎大家关注一下我的个人公众号。一起交流一起学习，有问必答。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)