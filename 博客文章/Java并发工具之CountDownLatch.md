# **一、简介**

> `CountDownLatch`允许一个或多个线程等待其他线程完成操作。

`CountDownLatch`，可以理解为**倒计时工具**，类似“三二一，芝麻开门”。该工具**用于一组操作执行完成才能执行后续操作的场景**。例如项目小组的晨会，需要等到所有人齐了才能正式开会。

`CountDownLatch`是通过一个**计数器**来实现的，当我们在`new`一个`CountDownLatch`对象的时候**初始化计数器的值，该值也代表了需要等待任务的完成数。每当一个线程完成自己的任务后，计数器的值就会减1。当计数器的值变为0时，就表示所有的线程均已经完成了任务，然后就可以恢复等待的线程继续执行了**。

# **二、类总览**

![](http://img.xianzilei.cn/CountDownLatch%E7%B1%BB%E5%9B%BE.png)

## **1. 核心属性**

`CountDownLatch`主要是**通过AQS的共享锁机制**实现的，因此它的**核心属性只有一个sync**。

```java
private final Sync sync;
```

我们来看一下**Sync内部类**的源码

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    //序列化版本号
    private static final long serialVersionUID = 4982264981922014374L;

    //构造函数
    Sync(int count) {
        setState(count);
    }

    //获取计数值
    int getCount() {
        return getState();
    }

    //重写AQS的方法，实现共享锁的获取
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    //重写AQS的方法，实现共享锁的释放
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

它继承自AQS，同时重写了`tryAcquireShared`和`tryReleaseShared`，以**完成具体的实现共享锁的获取与释放的逻辑**。方法的细节下面介绍。

## **2. 构造函数**

```java
public CountDownLatch(int count) {
    //传递的计数值不能小于0
    if (count < 0) throw new IllegalArgumentException("count < 0");
    //初始化sync，即初始化state值
    this.sync = new Sync(count);
}
```

# **三、使用案例**

以简介中说到的**开会**为例，如果参会人员一共10人，需要等到10人入场会议才开始，代码如下（**先混个脸熟，熟悉一下基本用法**）

```java
public static void main(String[] args) throws Exception {
    //参会人数
    int threadNum = 10;
    //创建countDownLatch，传递计数值为参会人数
    CountDownLatch countDownLatch = new CountDownLatch(threadNum);
    System.out.println("组织开会，一共有" + threadNum + "名员工参加会议。");
    //员工入场
    for (int i = 1; i <= threadNum; i++) {
        new Thread(() ->
            {
                System.out.println(Thread.currentThread().getName() + "号员工到达会场。");
                //计数减一
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
    }
    //老板等待员工入场
    countDownLatch.await();
    System.out.println("人到齐了，开会吧。");
}
```

**执行结果如下**

```tex
组织开会，一共有10名员工参加会议。
1号员工到达会场。
2号员工到达会场。
3号员工到达会场。
4号员工到达会场。
6号员工到达会场。
5号员工到达会场。
7号员工到达会场。
9号员工到达会场。
10号员工到达会场。
8号员工到达会场。
人到齐了，开会吧。
```

**主线程调用await方法等待子线程的执行结束，每个子线程执行结束后调用方法countDown，当计数减为0时，主线程就会从await方法返回，继续执行。**

# **四、核心方法分析**

## **1. countDown**

`CountDownLatch`提供`countDown`方法**递减锁存器的计数，如果计数到达零，则释放所有等待的线程**

```java
public void countDown() {
    sync.releaseShared(1);
}
```

调用`releaseShared`方法

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

这个AQS中的**释放共享锁**的方法，它会调用子类重写的`tryReleaseShared`方法，代码如下

```java
protected boolean tryReleaseShared(int releases) {
    //自旋
    for (;;) {
        //获取同步状态
        int c = getState();
        //如果同步状态为0，说明计数已经为0了，释放失败，直接返回false
        if (c == 0)
            return false;
        //否则计算释放后的同步状态
        int nextc = c-1;
        //CAS修改同步状态，修改成功则退出自旋，否则继续自旋
        if (compareAndSetState(c, nextc))
            //返回更新后的同步状态是否为0
            return nextc == 0;
    }
}
```

根据分析可得，该方法**只有在count值原来大于0，经过调用后变为0，才会返回true，否则返回false**。**返回true代表前面所有的任务都完成了**，此时会满足if条件，会调用方法`doReleaseShared`尝试**唤醒被阻塞的线程**。`doReleaseShared`方法在前面AQS的详解（[深入理解AQS实现原理](http://xianzilei.cn/blog/67)）中已经说过，这么不再赘述。

值得一提的是，我们其实并不关心`releaseShared`的返回值，而只关心`tryReleaseShared`的返回值，这里更像是**借了共享锁的“壳”，来完成我们的目的**。

## **2. await**

`CountDownLatch`提供await()方法来使**当前线程在计数至零之前一直等待**，除非线程被中断。**它与Condition的await方法的语义相同，该方法是阻塞式地等待，并且是响应中断的，只不过它不是在等待signal操作，而是在等待count值为0**。代码如下

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

调用AQS中的`acquireSharedInterruptibly`方法

```java
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //响应中断
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

首先响应中断，然后调用子类的`tryAcquireShared`方法

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

该方法的逻辑相当简单，**没有任何抢锁的行为，没有任何CAS操作**，只是**简单判断同步状态是否为0，是返回1，否则返回-1**。

* **如果状态等于0**，说明前面的**任务都已经完成了**，这里没必要阻塞挂起了，因此返回1，不进入方法`doAcquireSharedInterruptibly`，直接结束。
* **如果状态不等于0**，即大于0，说明**任务未完成**，当前线程需要入队阻塞挂起，并自旋判断state是否等于0。

# **五、总结**

* `CountDownLatch`内部通过**共享锁**实现。
* 在创建`CountDownLatch`实例时，需要传递一个`int`型的参数，该参数为**计数器的初始值**，也可以理解为该共享锁可以获取的总次数。
* 当某个线程调用`await`方法，程序**首先判断count的值是否为0**，**如果count不是0的话则会入CLH队后挂起，然后等待唤醒后自旋直到count为0为止**。
* 当其他线程调用`countDown`方法时，则会使**count值减一（如果是0则不减）**。**当count值原来不为0，操作后等于0的情况下，就会主动去唤醒CLH队列的第一个被阻塞的线程**。