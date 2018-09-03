---
title: 【java并发编程实战2】无锁编程CAS与atomic包
date: 2018-08-30 14:35:34
tags: [多线程, CAS, java,atomic]
categories: java并发编程实战
---

无锁编程可能大家在面试的时候会被经常问道，那么何为无锁编程CAS，怎么实现无锁编程CAS？

<!-- more -->

# 1、无锁编程CAS

## 1.1、CAS

CAS的全称是Compare And Swap 即比较交换，其算法核心思想如下

> 执行函数：CAS(V,E,N)

其包含3个参数

- V表示要更新的变量
- E表示预期值
- N表示新值

如果V值等于E值，则将V的值设为N。若V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。通俗的理解就是CAS操作需要我们提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，也就是执行CAS操作，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，但可以选择重新读取该变量再尝试再次修改该变量，也可以放弃操作，原理图如下

![1535524330707.png](https://upload-images.jianshu.io/upload_images/5338436-f350e287ad4e724f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于CAS操作属于乐观派，它总认为自己可以成功完成操作，当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作，这点从图中也可以看出来。基于这样的原理，CAS操作即使没有锁，同样知道其他线程对共享资源操作影响，并执行相应的处理措施。同时从这点也可以看出，由于无锁操作中没有锁的存在，因此不可能出现死锁的情况，也就是说无锁操作天生免疫死锁。

## 1.2、CPU指令对CAS的支持

或许我们可能会有这样的疑问，假设存在多个线程执行CAS操作并且CAS的步骤很多，有没有可能在判断V和E相同后，正要赋值时，切换了线程，更改了值。造成了数据不一致呢？答案是否定的，因为CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。

# 2、 atomic族类

在atomic包下共有AtomicBoolean、AtomicInteger、AtomicIntegerArray、AtmoicReference、AtomicReferenceFieldUpdater、LongAdder等类。

在上面线程不安全的例子中，我们用1000个线程调用整型变量的i的自增，然后输出i最后的大小，在多线程情况下，这明显是线程不安全的，因为`i++`不是原子操作，这里我们可以使用AtomicInteger代替Integer。

```java
public class AtomicIntegerExample {

    /**
     * 并发线程数目
     */
    private static int threadNum = 1000;

    /**
     * 闭锁
     */
    private static CountDownLatch countDownLatch  = new CountDownLatch(threadNum);

    private static AtomicInteger i = new AtomicInteger(0);
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int j = 0; j < threadNum; j++) {
            executorService.execute(() -> {
                add();
            });
        }
        // 使用闭锁保证当所有统计线程完成后，主线程输出统计结果。 其实这里也可以使用Thread.sleep() 让主线程阻塞等待一会儿实现
        countDownLatch.await();
        System.out.println(i.get());
    }

    private static void add() {
        countDownLatch.countDown();
        i.getAndIncrement();
    }

}
```

上面的代码就是线程安全的，运行输出 i=1000,这是为什么呢。我们来看看`getAndIncrement()`方法的实现

```java
	/**
     * Atomically increments by one the current value.
     *
     * @return the previous value
     */
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
```



我们看到方法是由一个unsafe对象的getAndAddInt方法实现。我们继续点进去看看getAndAddInt方法的实现。

```java
	public final int getAndAddInt(Object var1, long var2, int var4) {
        //
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
     }
```

上面就是`getAndAddInt`方法的实现,具体流程如下，

1、首先根据当前的传过来的对象指针，获取期望的值  var5,

2、然后while判断调用compareAndSwapInt方法 ，这是一个native本地方法，它有四个参数，

第一个参数，当前的对象，第二个参数实际的值，第三个参数期望的值，第四个参数想要更新的值。

只有实际的值等于期望的值的时候，才会把值更新成第四个参数，也就是想要的更新的值，否则一直循环尝试。

这也就是无锁编程，CAS。

在高并发的场景，这种循环尝试的次数会比较高，成功率会比较低，这样性能会比较差。但是在JDK8中推出了一个新的类名为`LongAdder`

我们看看它的用法。我们也继续上面的求和的例子，只需要把`AtomicInteger`改成`LongAddr`然后更改对应调用的方法即可。具体代码如下。

```java
public class LongAdderExample {

    /**
     * 并发线程数目
     */
    private static int threadNum = 1000;

    /**
     * 闭锁
     */
    private static CountDownLatch countDownLatch  = new CountDownLatch(threadNum);

    private static LongAdder i = new LongAdder();
    
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
        i.increment();
    }

}
```

运行程序，发现结果与预期的一样i=1000，是线程安全的。那么我们看看它又是如何。保证线程安全的呢。

由于篇幅原因我不跟入源码讲解了，大致思想与ConcurrentHashMapy大致一致，采用的是热点分离法，

把value分成base+cells数组，避免所有的写操作都是在value上面，这样就保证提高性能，但是在多线程情况下，统计会有误差。

接着我们讲解CAS中常常会遇到的ABA问题。

比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且two进行了一些操作变成了B，然后two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后one操作成功。尽管线程one的CAS操作成功，但是不代表这个过程就是没有问题的。如果链表的头在变化了两次后恢复了原值，但是不代表链表就没有变化。

这里我们可以借助乐观锁的一个概念，使用version版本号来判断是否一致，每次操作后版本号加1如果两次对比版本号一致才交换，这样就避免了ABA问题，在atomic包下面也提供了对应的类`AtomicStampedReference `。

`AtomicStampedReference`每次操作前判断更新的时间戳与预期的时间戳是否一致，这样就巧妙的避免了ABA问题。

最后大家希望关注一下我的个人公众号。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
