---
title: 【java并发编程实战5】线程与线程通信
date: 2018-09-07 17:36:13
tags: [java,多线程, 线程通信]
categories: java并发编程实战
---

现代操作系统在运行一个程序，会为其创建一个进程。然后它调度的最小单元是线程，线程也叫轻量级进程(Light Weight Process)，在一个进程中可以创建多个线程，这些线程都有各自的计算器，堆，栈和局部变量

<!--more -->

## 线程介绍

### 线程定义

>现代操作系统在运行一个程序，会为其创建一个进程。然后它调度的最小单元是线程，线程也叫轻量级进程(Light Weight Process)，在一个进程中可以创建多个线程，这些线程都有各自的计算器，堆，栈和局部变量，并且都能访问共享的内存变量。处理器在这些线程上高速切换，让用户感觉这些线程在同时在执行。



### 线程优先级

在计算机操作系统，操作系统采用的是时间片轮转法来调度线程的。操作系统会为每个线程分配时间片，当线程的时间片用了，就会发生线程调度，并且等待下次分配，线程分配到的时间片的多与少就决定线程能占用cpu的时间。

线程优先级就是决定线程能分配的时间片的多与少。在java线程中，可以通过`priority`来控制线程优先级，线程优先级的范围从`1~10`。默认值是`5`，优先级大的分配的时间片会大于优先级低，所以频繁阻塞线程可以设置高优先级，而占用cpu比较长的线程（计算线程）可以设置较低的优先级。但是在有的操作系统会无视对线程有限制。



###  线程的状态

|   状态名称   |                             解释                             |
| :----------: | :----------------------------------------------------------: |
|     NEW      |        初始状态，线程被构建，但是还没执行start()方法         |
|   RUNNABLE   |         运行状态，Java中将就绪与运行统称为 ”运行中“          |
|   BLOCKED    |             阻塞状态，表示线程阻塞与获取锁的过程             |
|   WAITING    | 等待状态，表示线程进入等待状态，进入该状态需要等待其他线程做出一些特定的动作（通知或者中断） |
| TIME_WAITING | 超时等待状态，该状态不同于WAITING，它是可以在指定时间能自行返回的 |
|  TERMINATED  |             终止状态，表示当前下才能已经执行完成             |

下面就用代码演示各种方法时线程的状态。

```java
public class ThreadState {
    public static void main(String[] args) {
        new Thread(new TimeWaiting(), "TimeWaitingThread").start();
        new Thread(new Waiting(), "WaitingThread").start();
        new Thread(new Blocked(), "BlockedThread - 1").start();
        new Thread(new Blocked(), "BlockedThread - 2").start();
    }
    
    static class TimeWaitnging implements Runnable {
        @override
        public void run() {
            while(true) {
                try {
                  TimeUnit.SECONDS.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    static class Waitnging implements Runnable {
        @override
        public void run() {
            while(true) {
                synchronized(Waiting.class) {
                    try {
                        Waiting.class.wait()
                    } catch (InterruptedException e) {
                    	e.printStackTrace();
                    }
                }
            }
        }
    }
    
    static class Blocked implements Runnable {
        @override
        public void run() {
            synchronized(Blocked.class) {
                while(true) {
                    try {
                          TimeUnit.SECONDS.sleep(1000L);
                    } catch (InterruptedException e) {
                            e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

java中线程状态的变迁如下图

![1536140160461.png](https://upload-images.jianshu.io/upload_images/5338436-fb8fa816ef0dde42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 线程通信 



### 通知等待机制

首先我们需要了解一下wait()与notify方法

- wait() 调用该方法的线程会进入WAITING状态，只有等待另外线程通知或者被中断才能返回，wait方法会释放对象锁
- wait(long) 调用该方法的线程会进入TIME_WAITING状态， 超过等待一段时间，参数单位是毫秒，意味着等待n毫秒后，如果没有通知就返回
- wait(long, int) 控制跟粒度更细，到纳秒
- notify) 通知一个在此对象上等待的线程，从其从wait()方法返回，返回的前提是该线程获取到了对象的线程锁。
- notifyAll() 通知在此对象上等待的所有线程。

现在我们可以通过 `synchronized+wait+notify`来实现一个简单的`通知\等待模型`

- 等待方（消费者）

1）获取对象锁

2）如果条件不满足，那么调用对象的wait()方法，被通知依旧要检查条件。

3）条件满足则执行对应的逻辑

伪代码：

```java
synchronized(对象) {
    while (条件不满足) {
        对象.wait();
    }
    处理对应逻辑
}
```

- 通知方法 (生产者)

1）获取对象锁

2）改变条件

3）通知所有等在在此对象上的线程

对应伪代码

```java
synchronized(对象) {
    改变条件
    对象.notifyAll();
}
```

根据上面的通知等待机制，我们可以实现一个简单的线程池。



首先我们先定义一下线程池的接口。

```java
/**
 * @author yukong
 * @date 2018/9/5
 * @description 线程池接口，抽象出来，定义规范
 */
public interface ThreadPool<Job extends Runnable> {

    /**
     * 执行任务，这个任务需要继承Runnable接口
     * @param job 任务
     */
    void execute(Job job);

    /**
     * 关闭线程池
     */
    void shutdown();

    /**
     * 添加工作者数目
     * @param num 要添加的数量
     */
    void addWorkers(int num);

    /**
     * 减少工作者数目
     * @param num 要减少的数量
     */
    void removeWorks(int num);

    /**
     * 获取正在等待执行的任务数量
     * @return
     */
    int getJobCount();
}
```



然后编写一个实现类

```java
/**
 * @author yukong
 * @date 2018/9/5
 * @description
 */
public class DefaultThreadPool<Job extends Runnable> implements ThreadPool<Job> {

    /**
     * 线程池最大数
     */
    private static final int MAX_WORKER_NUMBERS = 10;

    /**
     * 线程池默认数
     */
    private static final int DEFAULT_WORKER_NUMBERS = 5;

    /**
     * 线程池最小数
     */
    private static final int MIN_WORKER_NUMBERS = 1;

    /**
     * 任务队列
     */
    private final LinkedList<Job>  jobs = new LinkedList<>();

    /**
     * 工作者列表
     */
    private final List<Worker> workers = Collections.synchronizedList(new ArrayList<>());

    /**
     * 工作者线程数量
     */
    private int workerNum = DEFAULT_WORKER_NUMBERS;

    /**
     * 线程编号生成
     */
    private AtomicInteger threadNum = new AtomicInteger();

    public DefaultThreadPool(int workerNum) {
        this.workerNum = workerNum > MAX_WORKER_NUMBERS ? MAX_WORKER_NUMBERS
                : workerNum < MIN_WORKER_NUMBERS ? MIN_WORKER_NUMBERS
                : workerNum;
        initializeWorkers(this.workerNum);
    }

    private void initializeWorkers(int num) {
        for (int i = 0; i < num; i++) {
            Worker worker = new Worker();
            workers.add(worker);
            Thread thread = new Thread(worker, "ThreadPool-Worker-" + threadNum.incrementAndGet());
            thread.start();
        }
    }

    @Override
    public void execute(Job job) {
        if (job != null) {
            synchronized (jobs) {
                jobs.addLast(job);
                jobs.notifyAll();
            }
        }
    }

    @Override
    public void shutdown() {
        for (Worker worker: workers) {
            worker.shutdown();
        }
    }

    @Override
    public void addWorkers(int num) {
       synchronized (jobs) {
           // 限制新增的数目与已有的数目之和超过最大数
           if (num + this.workerNum> MAX_WORKER_NUMBERS) {
               num = MAX_WORKER_NUMBERS - this.workerNum;
           }
           initializeWorkers(num);
           this.workerNum += num;
       }
    }

    @Override
    public void removeWorks(int num) {
        synchronized (jobs) {
            if (num > this.workerNum) {
                throw new IllegalArgumentException("beyond workNum");
            }
            int count = 0;
            while (count < num) {
                Worker worker = workers.get(count);
                if (workers.remove(worker)) {
                    worker.shutdown();
                    count++;
                }
            }
            this.workerNum -= num;
        }
    }

    @Override
    public int getJobCount() {
        return jobs.size();
    }

    class Worker implements Runnable {

        private volatile Boolean running = true;

        @Override
        public void run() {
            while (running) {
                Job job = null;
                synchronized (jobs) {
                    while (jobs.isEmpty()) {
                        try {
                            jobs.wait();
                        } catch (InterruptedException e) {
                            // 设置中断标记，让外界感知
                            Thread.currentThread().interrupt();
                            return;
                        }
                    }
                    job = jobs.removeFirst();
                }
                if (job != null) {
                    job.run();
                }
            }
        }

        public void shutdown() {
            running = false;
        }
    }
}
```
至此我们就实现了一个简单的线程池了。

最后欢迎大家关注一下我的个人公众号。一起交流一起学习，有问必答。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
