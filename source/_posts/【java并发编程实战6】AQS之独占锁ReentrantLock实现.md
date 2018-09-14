---
title: 【java并发编程实战6】AQS之独占锁ReentrantLock实现
date: 2018-09-10 18:11:08
tags: [java,多线程, 独占锁, AQS]
categories: java并发编程实战
---

# 前言

自从JDK1.5后，jdk新增一个并发工具包`java.util.concurrent`，提供了一系列的并发工具类。而今天我们需要学习的是`java.util.concurrent.lock`也就是它下面的lock包，其中有一个最为常见类`ReentrantLock`，

我们知道`ReentrantLock`的功能是实现代码段的并发访问控制，也就是通常意义上所说的锁。之前我们也学习过一种锁的实现，也就是`synchronized`关键词，`synchronized`是在字节码层面，通过对象的监视器锁实现的。那么`ReentrantLock`又是怎么实现的呢？

<!-- more -->

如果不看源码，可能会以为它的实现是通过类似于`synchronized`，通过对象的监视器锁实现的。但事实上它仅仅是一个工具类！没有使用更“高级”的机器指令，不是关键字，也不依靠JDK编译时的特殊处理，仅仅作为一个普普通通的类就完成了代码块的并发访问控制，这就更让人疑问它怎么实现的代码块的并发访问控制的了。

我们查看源码发现，它是通过继承抽象类实现的`AbstractQueuedSynchronizer`，为了方便描述，接下来我将用AQS代替`AbstractQueuedSynchronizer`。

# 关于AQS

> AQS，它是用来构建锁或者其他同步组建的基础框架，我们见过许多同步工具类都是基于它构建的。包括`ReentrantLock、CountDownLatch等`。在深入了解AQS了解之前，我们需要知道锁跟AQS的区别。锁，它是面向使用者的，它定义了使用者与锁交互的接口，隐藏了实现的细节；而AQS面像的是锁的实现者，它简化了锁的实现。锁与AQS很好的隔离使用者与实现者所需要关注的领域。那么我们今天就作为一个锁的实现者，一步一步分析锁的实现。

AQS又称同步器，它的内部有一个int成员变量state表示同步状态，还有一个内置的FIFO队列来实现资源获取线程的排队工作。通过它们我们就能实现锁。

在实现锁之前，我们需要考虑做为锁的使用者，锁会有哪几种？

通常来说，锁分为两种，一种是独占锁(排它锁,互斥锁),另一种就是共享锁了。根据这两类，其实AQS也给我们提供了两套API。而我们作为锁的实现者，通常都是要么全部实现它的独占api，要么实现它的共享api，而不会出现一起实现的。即使juc内置的`ReentrantReadWriteLock`也是通过两个子类分别来实现的。

# 锁的实现

## 独占锁

独占锁又名互斥锁，同一时间，只有一个线程能获取到锁，其余的线程都会被阻塞等待。其中我们常用的`ReentrantLock`就是一种独占锁，我们一起来分`ReentrantLock` 析分析`ReentrantLock`的同时看一看AQS的实现，再推理出AQS独特的设计思路和实现方式。最后，再看其共享控制功能的实现。

首先我们来看看获取锁的过程

## 加锁

我们查看`ReentrantLock`的源码。来分析它的lock方法

```java
  public void lock() {
        sync.lock();
   }
```

与我们之前分析的一样，锁的具体实现由内部的代理类完成，lock只是暴露给锁的使用者的一套api。使用过ReentrantLock的同学应该知道，ReentrantLock又分为公平锁和非公平锁，所以，ReentrantLock内部只有两个sync的实现。

```java
    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync{..}
 	/**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync{..}
```

- 公平锁 ：每个线程获取锁的顺序是按照调用lock方法的先后顺序来的。
- 非公平锁：每个线程获取锁的顺序是不会按照调用lock方法的先后顺序来的。完全看运气。

所以我们完全可以猜测到，这个公平与不公平的区别就体现在锁的获取过程。我们以公平锁为例，来分析获取锁过程，最后对比非公平锁的过程，寻找差异。

### lock

查看FairSync的lock方法

```java
	final void lock() {
            acquire(1);
        }
```

这里它调用到了父类AQS的acquire方法，所以我们继续查看acquire方法的代码

### acquire

```java
/**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
            selfInterrupt();
    }
```

查看方法方法的注释我们可以知道这个方法的作用，这里我简单的翻译一下.

Acquires方法是一个独占锁模式的方法，它是不会响应中断的。它至少执行一次tryAcquire去获取锁，如果返回true，则代表获取锁成功，否则它将会被加入等待队列阻塞，直到重新尝试获取锁成功。所以我们需要看看尝试获取锁的方法tryAcquire的实现

### tryAcruire

```java
   protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

抛出一个异常，没有实现。所以我们需要查看它的子类，在我们这里就是FairSync的实现。

> 这里也会大家会有疑惑，没有实现为什么不写成抽象方法呢，前面我们提到过，我们不会同时在一个类中实现独占锁跟共享锁的api，那么tryAcruire是属于独占锁，那么如果我想一个共享锁也要重新独占锁的方法吗？所以大师的设计是绝对没有问题的。

```java

```

目前为止，如果获取锁成功，则返回true，获取锁的过程结束，如果获取失败，则返回false

按照之前的逻辑，如果线程获取锁失败，则会被放入到队列中，但是在放入之前，需要给线程包装一下。

那么这个addWaiter就是包装线程并且放入到队列的过程实现的方法。

### addWaiter

```java
   /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

注释: 把当前线程作为一个节点添加到队列中，并且为这个节点设置模式

> 模式： 也就是独占模式/共享模式,在这里模式是形参，所以我们看看起调方
>
> `acquireQueued(addWaiter(Node.EXCLUSIVE), arg))` Node.EXCLUSIVE 就代表这是独占锁模式。

创建好节点后，将节点加入到队列尾部，此处，在队列不为空的时候，先尝试通过cas方式修改尾节点为最新的节点，如果修改失败，意味着有并发，这个时候才会进入enq中死循环，“自旋”方式修改。

将线程的节点接入到队里中后，当然还需要做一件事:将当前线程挂起！这个事，由acquireQueued来做。

在解释acquireQueued之前，我们需要先看下AQS中队列的内存结构，我们知道，队列由Node类型的节点组成，其中至少有两个变量，一个封装线程，一个封装节点类型。

而实际上，它的内存结构是这样的（第一次节点插入时，第一个节点是一个空节点，代表有一个线程已经获取锁，事实上，队列的第一个节点就是代表持有锁的节点）：
![0730009.png](https://upload-images.jianshu.io/upload_images/5338436-332432b498a8401d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

黄色节点为队列默认的头节点，每次有线程竞争失败，进入队列后其实都是插入到队列的尾节点（tail后面）后面。这个从enq方法可以看出来，上文中有提到enq方法为将节点插入队列的方法:

### enq

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 一个空的节点，通常代表获取锁的线程
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

### acquireQueued

接着我们来看看当节点被放入到队列中，如何将线程挂起，也就是看看acquireQueued方法的实现。

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 获取当前节点前驱结点
                final Node p = node.predecessor();
                // 如果前驱节点是head，那么它就是等待队列中的第一个线程
                // 因为我们知道head就是获取线程的节点，那么它就有机会再次获取锁
                if (p == head && tryAcquire(arg)) {
                    //成功后，将上图中的黄色节点移除，Node1变成头节点。 也证实了head就是获取锁的线程的节点。
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 1、检查前一个节点的状态，判断是否要挂起
                // 2、如果需要挂起，则通过JUC下的LockSopport类的静态方法park挂起当前线程，直到被唤醒。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 如果发生异常
            if (failed)
                // 取消请求，也就是将当前节点重队列中移除。
                cancelAcquire(node);
        }
    }
```

这里我还需要解释的是：

1、Node节点除了存储当前线程之外，节点类型，前驱后驱指针之后，还存储一个叫waitStatus的变量，该变量用于描述节点的状态。共有四种状态。

```java
	   /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
```

分别表示：

- 1 = 取消状态，该节点将会被队列移除。
- -1 = 等待状态，后驱节点处于等待状态。
- -2 = 等待被通知，该节点将会阻塞至被该锁的condition的await方法唤醒。
- -3 = 共享传播状态，代表该节点的状态会向后传播。

到此为止，一个线程对于锁的一次竞争才告于段落，结果有两种，要么成功获取到锁（不用进入到AQS队列中），要么，获取失败，被挂起，等待下次唤醒后继续循环尝试获取锁，值得注意的是，AQS的队列为FIFO队列，所以，每次被CPU假唤醒，且当前线程不是出在头节点的位置，也是会被挂起的。AQS通过这样的方式，实现了竞争的排队策略。



## 释放锁

看完了加锁，再看释放锁。我们先不看代码也可以猜测到释放锁需要的步骤。

- 队列的头节点是当前获取锁的线程，所以我们需要移除头节点
- 释放锁，唤醒头节点后驱节点来竞争锁

接下来我们查看源码来验证我们的猜想是否在正确。

### unlock

```java
public void unlock() {
    sync.release(1);
}
```

unlock方法调用AQS的release方法，因为我们的acquire的时候传入的是1，也就是同步状态量+1，那么对应的解锁就要-1。

### release

```java
  public final boolean release(int arg) {
        // 尝试释放锁
        if (tryRelease(arg)) {
            // 释放锁成功，获取当前队列的头节点
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 唤醒当前节点的下一个节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```



### tryRelease

同样的它是交给子类实现的

```java
	protected final boolean tryRelease(int releases) {
        
            int c = getState() - releases;
        	// 当前线程不是获取锁的线程 抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
        	// 因为是重入的关系，不是每次释放锁c都等于0，直到最后一次释放锁时，才通知AQS不需要再记录哪个线程正在获取锁。
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
    }
```

### unparkSuccessor

释放锁成功之后，就唤醒头节点后驱节点来竞争锁 

```java
 private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

值得注意的是，寻找的顺序是从队列尾部开始往前去找的最前面的一个waitStatus小于0的节点。因为大于0 就是1状态的节点是取消状态。

## 公平锁与非公平锁

到此我们锁获取跟锁的释放已经分析的差不多。那么公平锁跟非公平锁的区别在于加锁的过程。对比代码

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }
}
```

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
}
```

从代码中也可以看出来，非公平在公平锁的加锁的逻辑之前先直接cas修改一次state变量（尝试获取锁），成功就返回，不成功再排队，从而达到不排队直接抢占的目的。





最后欢迎大家关注一下我的个人公众号。一起交流一起学习，有问必答。
![公众号](https://upload-images.jianshu.io/upload_images/5338436-ddb4dd1530787751.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)