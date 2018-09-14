---
title: 【java并发编程实战3】解密volatile
date: 2018-09-03 19:19:22
tags: [多线程, java, volatile]
categories: java并发编程实战
---

自从jdk1.5以后，`volatile`可谓发生了翻天覆地的变化，从一个一直被吐槽的关键词，变成一个轻量级的线程通信代名词。
<!-- more -->

接下来我们将从以下几个方面来分析以下`volatile`。

- 重排序与`as if serial`的关系

- `volatile`的特点
- `volatile`的内存语义
- `volatile`的使用场景

## 重排序与as if serial的关系

重排序值得是编译器与处理器为了优化程序的性能，而对指令序列进行重新排序的。

但是并不是什么情况下都可以重排序的，

- 数据依赖

  ```java
  a = 1;			// 1
  b = 2;			// 2
  ```

  在这种情况，1、2不存在数据依赖，是可以重排序的。

  ```java
  a = 1;			// 1
  b = a;			// 2	
  ```

  在这种情况，1、2存在数据依赖，是禁止重排序的。

- `as if serial`

  简单的理解就是。不管怎么重排序，在单线程情况下程序的执行结果是一致。

根据 `as if serial`原则，它强调了单线程。那么多线程发生重排序又是怎么样的呢？

请看下面代码

```java
public class VolatileExample1 {

    /**
     * 共享变量 name
     */
    private static String name = "init";


    /**
     * 共享变量 flag
     */
    private  static boolean flag = false;


    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            name = "yukong";			// 1
            flag = true;				// 2
        });
        Thread threadB = new Thread(() -> {
            if (flag) {   				// 3
                System.out.println("flag = " + flag + " name = " +name);  // 4
            };
        });
    }
}
```



上面代码中，name输出一定是`yukong`吗，答案是不一定，根据`happen-before`原则与`as if serial`

原则，由于 1、2不存在依赖关系，可以重排序，操作3、操作4也不存在数据依赖，也可以重排序。

那么就有可能发生下面的情况

![1535967197003.png](https://upload-images.jianshu.io/upload_images/5338436-25e0f588bd579e07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，操作1与操作2发生了重排序，程序运行的时候，线程A先将flag更改成true，然后线程B读取flag变量并且判断，由于此时flag已经是true，线程B将继续读取name的值，由于此时线程name的值还没有被线程A写入，那么线程此时输出的name就是初始值，因为在多线程的情况下，重排序存在线程安全问题。

## volatile的特点

`volatile`变量具有以下的特点。

- 可见性。对于一个`volatile`变量的读，总是能看到任意线程对这个变量的最后的修改。
- 有序性。对于存在指令重排序的情况，`volatile`会禁止部分指令重排序。

这里我先介绍一下`volatile`关键词的特点，接下来我们将会从它的内存语义来解释，为什么它会具有以上的特点，以及它使用的场景。



## volatile的内存语义

- 当写一个`volatile`变量时，JMM会**立即**将本地变量中对应的共享变量值刷新到主内存中。
- 当读一个`volatile`变量时，JMM会将线程本地变量存储的值，置为无效值，线程接下来将从主内存中读取共享变量。

如果一个场景存在对`volatile`变量的读写场景，在读线程B读一个`volatile`变量后，，写线程A在写这个`volatile`变量前所有的所见的共享变量的值都将会立即变得对读线程B可见。

那么这种内存语义是怎么实现的呢？

其实编译器生产字节码的时候，会在指令序列中插入内存屏障来禁止指令排序。下面就是JMM内存屏障插入的策略。

- 在每一个volatile写操作前插入一个StoreStore屏障
- 在每一个volatile写操作后插入一个StoreLoad屏障
- 在每一个volatile写操作后插入一个LoadLoad屏障
- 在每一个volatile读操作后插入一个LoadStore屏障

那么这些策略中，插入这些屏障有什么作用呢？我们逐条逐条分析一下。

- 在每一个volatile写操作前插入一个StoreStore屏障，这条策略保证了volatile写变量与之前的普通变量写不会重排序，即是只有当volatile变量之前的普通变量写完，volatile变量才会写。 这样就保证volatile变量写不会跟它之前的普通变量写重排序
- 在每一个volatile写操作后插入一个StoreLoad屏障，这条策略保证了volatile写变量与之后的volatile写/读不会重排序，即是只有当volatile变量写完之后，你后面的volatile读写才能操作。 这样就保证volatile变量写不会跟它之后的普通变量读重排序
- 在每一个volatile读操作后插入一个LoadLoad屏障，这条策略保证了volatile读变量与之后的普通读不会重排序，即只有当前volatile变量读完，之后的普通读才能读。 这样就保证volatile变量读不会跟它之后的普通变量读重排序
- 在每一个volatile读操作后插入一个LoadStore屏障，这条策略保证了volatile读变量与之后的普通写不会重排序，即只有当前volatile变量读完，之后的普通写才能写。样就保证volatile变量读不会跟它之后的普通变量写重排序

根据这些策略，volatile变量禁止了部分的重排序，这样也是为什么我们会说volatile具有一定的有序的原因。

 根据以上分析的`volatile`的内存语义，大家也就知道了为什么前面我们提到的`happen-before`原则会有一条

- volatile的写happen-before与volaile的读。

那么根据`volatile`的内存语义，我们只需要更改之前的部分代码，只能让它正确的执行。

即把flag定义成一个`volatile`变量即可。

```java
public class VolatileExample1 {

    /**
     * 共享变量 name
     */
    private static String name = "init";


    /**
     * 共享变量 flag
     */
    private  volatile static boolean flag = false;


    public static void main(String[] args) throws InterruptedException {
        Thread threadA = new Thread(() -> {
            name = "yukong";			// 1
            flag = true;				// 2
        });
        Thread threadB = new Thread(() -> {
            if (flag) {   				// 3
                System.out.println("flag = " + flag + " name = " +name);  // 4
            };
        });
    }
}
```



我们来分析一下

- 由于 flag是volatile变量 那么在volatile写之前插入一个storestore内存屏障，所以1,2不会发生重排序,即1happen before 2

- 由于 flag是volatile变量 那么在volatile读之后插入一个loadload内存屏障，所以3,4不会发生重排序,即3happen before 4

- 根据happen-before原则，volatile写happen before volatile读，即是 2happen before 3。

- 根据happen-before的传递性，所以1 happen before4。



![1535969126569.png](https://upload-images.jianshu.io/upload_images/5338436-b533cdf78c24ab64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#  volatile的使用场景

- 标记变量，也就是上面的flag使用
- double check 单例模式中

下面我们看看double check的使用

```java
class Singleton{
    private volatile static Singleton instance = null;
     
    private Singleton() {
         
    }
     
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

至于为何需要这么写请参考：

　　《Java 中的双重检查（Double-Check）》<http://www.iteye.com/topic/652440>


最后大家希望关注一下我的个人公众号。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)