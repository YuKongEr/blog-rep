---
title: 【java并发编程实战1】何为线程安全性
date: 2018-08-29 19:17:02
categories: java并发编程实战
tags: [java, 多线程, 并发编程]
---

多线程问题，一直是我们老生常谈的一个问题，在面试中也会被经常问到，如何去学习理解多线程，何为线程安全性，那么大家跟我的脚步一起来学习一下。
<!-- more -->

# 线程安全性

定义：

> 当多个线程访问某个类时，不管运行时环境采用**何种调度方式** 或者这些线程如何交替执行，并且在主调代码中**不需要任何额外的同步或者协同**，这个类都能表现**正确的行为**,那么称这个类时线程安全的。

线程的安全性主要体现在三个方法

- 原子性：即不可分割，提供互斥访问，同一时刻只能有一个线程对它进行操作
- 可见性：一个线程对共享变量的修改，可以及时被其他线程观察到
- 有序性：序在执行的时候，程序的代码执行顺序和语句的顺序是一致的。 

## 1、原子性

1、访问（读/写）某个共享变量的操作从其执行线程以外的线程来看，该操作要么已经执行结果，有么尚未执行，也就是说其他线程不会看到“该操作执行了部分的效果”。

2、访问同一组共享变量的原子操作 不能够被交错的。

在java中实现原子性的两种方式：

- 使用CAS也是atomic包下的类。

- 使用锁

> 在java语言中，除long/double之外的任何类型的变量的写操作都是原子操作。
>
> java语言中任何变量的读操作都是原子操作。
>
> 需要注意的是 原子操作 + 原子操作  != 原子操作
>
> 例如 i++ 先读后写  读跟写都是原子操作，但是 i++并不是原子操作

下面用代码讲一下实现的两种方式



例子

```java
/**
 * @author yukong
 * @date 2018/8/29
 * @description 线程不安全
 */
public class CountExample {

    /**
     * 并发线程数目
     */
    private static int threadNum = 1000;

    /**
     * 闭锁
     */
    private static CountDownLatch countDownLatch  = new CountDownLatch(threadNum);

    private static Integer i = 0;
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int j = 0; j < threadNum; j++) {
            executorService.execute(() -> {
                add();
            });
        }
        // 使用闭锁保证当所有统计线程完成后，主线程输出统计结果。 其实这里也可以使用Thread.sleep() 让主线程阻塞等待一会儿实现
        countDownLatch.await();
        System.out.println(i);
    }

    private static void add() {
        countDownLatch.countDown();
        i++;
    }

}
```

上面这段代码很明显因为i++不是原子性操作，所以不是线程安全的。

那么根据上面讲的，我们可以使用锁，或者atomic包下的类实现。

## 2、可见性

一个线程对共享变量的修改能够及时被其他线程所观察。

这句话怎么理解呢？

在JMM（Java Memory Model）的定义中，所有的变量都需要存储在主体内存中，主内存是共享内存区域，所有的线程都能访问的，但是线程对变量的操作（读、写）必须在工作内存中完成。

1、首先将变量从主内存中拷贝到自己的工作内存。

2、对变量进行读写操作。

3、操作完成，将变量回写到主内存中。

从上面可以得知，线程不能直接操作主内存的变量，必须要在工作内存中操作。

简单了解一下JMM的规定，那么我们就可以很容易的理解可见性了。
![1535527111889.png](https://upload-images.jianshu.io/upload_images/5338436-fea53ade81191d4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


由上图可知 ，在多线程情况下，线程对共享变量的的操作都是拷贝一份副本到自己的工作内存中操作的，然后才写回到主内存中，这就可能存在一个问题，线程1修改了共享变量X的值，但是还未写回主内存，另外一个线程2又对主内存中的同一个共享变量x进行操作，但此时线程1工作内存中的变量x对线程n并不可以，这种工作内存与主内存同步延迟的问题就造成了可见性问题，另外指令重排序也会导致可见性问题。

那么对于可见性问题，使用什么解决方法呢？

- synchronized关键字
- volatile关键词 

为什么synchronized能保证可见性呢？根据JMM关于synchronized的规定

- 线程解锁前，必须把共享变量的最新刷新到主内存。
- 线程加锁时，将清空工作内存中共享变量的值，从而使用共享变量时需要重新从主内存中读取最新值。

那么volatile又是怎么实现可见性的呢？

其实volatile是通过加入**内存屏障**和禁止指令**重排序**优化来实现的。

- 对volatile变量写操作时，会在写操作后加入一条store屏障指令，将工作内存中的共享变量值刷新到主内存中
- 对于volatile变量读操作时，会在读操作前加入一条load屏障指定，从主内存读取共享变量最新的值到工作内存中。

那大家可能就会想问了，我把上面的代码的i变量用`volatile`修饰一下，是不是就保证线程安全，输出的结果就是1000呢，答案是否定的，volatile保证的是可见性，并不能保证原子性。但是利用volatile可见性这个特点，我们可以利用它完全一些线程中的通信

```java
volatile boolean flag = false; 
// thread a
{
    flag = true;
    // do somethings
}

// thread b
{
    while (flag) {
        // do somethings
    }
}
```

这样就完全一个线程中通信的案例。

## 3、有序性

> 在JMM（java 内存 模型）中，运行编译器和处理器对指令就行重排序，但是重排序过程不会影响到**单线程**程序的执行，却会影响多线程并发执行的正确性。

在java中，可以通过`volatile`关键字来保证一定的有序性。另外也可以通过`synchronized`和`Lock`来保证有序性。很显然，synchronized跟lock保证每个时刻是只有一个线程执行同步代码，相当于让线程属性执行同步代码，自然保证了有序性。

另外java内存模型也具备一些先天的`有序性`,即不需要通过任何手段就能够保证的有序性，这个通常也称为`Happen-Before`原则。如果两个操作的资源无法从`Happen-Before`原则推导出来，那么他们就不能保证它的有序性，虚拟机就可以随机对他们进行重排序。

那么下面就详细介绍`Happen-Before`(先行发生原则)：

1.  线程次序规则： 在一个线程内，按照代码顺序，书写在前的代码先行发生于书写在后的代码操作。
2.  锁定原则：一个unlock操作先行发生于后面的对同一个锁的lock操作。
3.  volatile变量原则，对同一个变量的写操作先行发生于后面对这个变量的读操作。
4.  传递原则：如果操作A先行发生于操作B,而且操作B又先行发生于操作C，则可以得出操作A先行发生于操作C。
5.  线程启动原则：Thread对象的start()方法先行发生于此线程的每一个操作。
6.  线程中断原则：Thread对象的interrupt()方法先行发生于被中断线程检测到中断事件的发生
7.  线程终结原则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束，Thread.isAlive()的返回值手段检测线程是否已经终止。
8.  线程终结原则：一个对象的初始化完成先行发生于他的finalize()方法的开始。
# 4、总结
如果一个操作具有以上的三种特性，那么我们称它为线程安全的。



最后欢迎大家关注一下我的个人公众号 程序咖啡厅 每天一杯逐渐成长。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)