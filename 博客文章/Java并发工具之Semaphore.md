# **一、简介**

摘自《Java并发编程的艺术》一书

> `Semaphore`（信号量）是用来**控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源**。

`Semaphore`一般用于**流量的控制**，特别是**公共资源有限的应用场景**。例如**数据库的连接**，假设数据库的连接数上线为10个，多个线程并发操作数据库可以使用Semaphore来控制并发操作数据库的线程个数最多为10个。

# **二、类总览**

![](http://img.xianzilei.cn/Semaphore%E7%B1%BB%E5%9B%BE.png)

通过上面的类图可以看到，**Semaphore与ReentrantLock的内部类的结构相同，类内部总共存在Sync、NonfairSync、FairSync三个类，NonfairSync与FairSync类继承自Sync类，Sync类继承自AbstractQueuedSynchronizer抽象类**。

## **1. 类继承关系**

```java
public class Semaphore implements java.io.Serializable {}
```

**实现了Serializable接口**。

## **2. 类的内部类**

### **2.1 Sync类**

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    //序列化版本号
    private static final long serialVersionUID = 1192457210091910933L;

    //构造方法
    Sync(int permits) {
        setState(permits);
    }

    //获取许可
    final int getPermits() {
        return getState();
    }

    //共享模式下非公平策略获取
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    //共享模式下进行释放
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

    //根据指定的缩减量减小可用许可的数目
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    //获取并返回立即可用的所有许可
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

方法细节第四章会详细分析

### **2.2 NonfairSync类**

`NonfairSync`类继承了`Sync`类，表示**采用非公平策略获取资源**，其只有一个`tryAcquireShared`方法，重写了AQS的该方法

```java
static final class NonfairSync extends Sync {
    //序列化版本号
    private static final long serialVersionUID = -2694183684443567898L;

    //构造方法
    NonfairSync(int permits) {
        super(permits);
    }

    //共享模式下获取
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

### **2.3 FairSync类**

`FairSync`类继承了`Sync`类，表示**采用公平策略获取资源**，其只有一个`tryAcquireShared`方法，重写了AQS的该方法

```java
static final class FairSync extends Sync {
    //序列化版本号
    private static final long serialVersionUID = 2014338818796000944L;

    //构造方法
    FairSync(int permits) {
        super(permits);
    }

    //共享模式下获取
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

## **3. 类的属性**

与`CountDownLatch`类似，`Semaphore`主要是通过**AQS的共享锁机制实现的**，因此它的核心属性只有一个sync

```java
//序列化版本号
private static final long serialVersionUID = -3222578661600680210L;
//同步队列
private final Sync sync;
```

## **4. 构造方法**

**Semaphore构造方法有两种**

```java
//指定许可数，默认为非公平策略
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

//指定许可数和是否公平策略
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

构造方法类似于`ReentrantLock`的构造方法，只是多了一个**许可数**的参数。调用构造方法的结果就是**初始化了同步队列实例，设置state值为permits值**。

# **三、使用案例**

这里以经典的**停车**作为案例。假设停车场有3个停车位，此时有5辆汽车需要进入停车场停车。

```java
public static void main(String[] args) {
    //定义semaphore实例，设置许可数为3，即停车位为3个
    Semaphore semaphore = new Semaphore(3);
    //创建五个线程，即有5辆汽车准备进入停车场停车
    for (int i = 1; i <= 5; i++) {
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "尝试进入停车场...");
                //尝试获取许可
                semaphore.acquire();
                //模拟停车
                long time = (long) (Math.random() * 10 + 1);
                System.out.println(Thread.currentThread().getName() + "进入了停车场，停车" + time +
                                   "秒...");
                Thread.sleep(time);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println(Thread.currentThread().getName() + "开始驶离停车场...");
                //释放许可
                semaphore.release();
                System.out.println(Thread.currentThread().getName() + "离开了停车场！");
            }
        }, i + "号汽车").start();
    }
}
```

执行结果如下

```tex
1号汽车尝试进入停车场...
5号汽车尝试进入停车场...
4号汽车尝试进入停车场...
3号汽车尝试进入停车场...
2号汽车尝试进入停车场...
5号汽车进入了停车场，停车5秒...
1号汽车进入了停车场，停车8秒...
4号汽车进入了停车场，停车9秒...
5号汽车开始驶离停车场...
5号汽车离开了停车场！
3号汽车进入了停车场，停车10秒...
1号汽车开始驶离停车场...
1号汽车离开了停车场！
2号汽车进入了停车场，停车2秒...
4号汽车开始驶离停车场...
4号汽车离开了停车场！
2号汽车开始驶离停车场...
2号汽车离开了停车场！
3号汽车开始驶离停车场...
3号汽车离开了停车场！
```

总结，每个线程调用acquire方法尝试获取许可，如果成功就会继续执行，否则就会被阻塞，当获取许可成功后，调用release方法释放许可供其他线程使用，之前被阻塞的线程会被唤醒继续执行。

# **四、核心方法**

通过上面的案例我们知道Semaphore的核心方法是acquire获取信号量和release释放信号量，下面我们来一一分析

## **1. acquire**

**Semaphore提供了acquire方法来获取一个许可**。老规矩，我们跟着上面的案例进入源码分析，acquire方法定义如下

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

内部调用的是AQS的`acquireSharedInterruptibly`方法， 即**共享式获取响应中断**，定义如下

```java
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

该方法在[AQS详解](http://xianzilei.cn/blog/67)中有讲解过，这里不赘述了，我们来分析一下子类实现的`tryAcquireShared`方法，这里就要分公平和非公平策略两种情况了

### **1.1 非公平策略下**

非公平策略下的`tryAcquireShared`方法定义如下

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
```

内部调用Sync类中的`nonfairTryAcquireShared`方法

```java
final int nonfairTryAcquireShared(int acquires) {
    //自旋
    for (;;) {
        //获取可用许可值
        int available = getState();
        //计算剩余的许可值
        int remaining = available - acquires;
        //如果剩余许可值小于0，说明许可不够用了，直接返回，否则CAS更新同步状态，更新成功返回，否则继续自旋
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

该方法本质就是一个**自旋方法**，通过**自旋+CAS来保证修改许可值的线程安全性**。该方法返回的情况有如下两种情况

* **信号量不够，直接返回，返回值为负数，表示获取失败**；
* **信号量足够，且CAS操作成功，返回值为剩余许可值，获取成功**。

### **1.2 公平策略下**

公平策略下的`tryAcquireShared`方法定义如下

```java
protected int tryAcquireShared(int acquires) {
    //自旋
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

**我们看到它与非公平策略的唯一区别就是多了下面这个if代码**

```java
if (hasQueuedPredecessors())
    return -1;
```

即在获取共享锁之前，先调用`hasQueuedPredecessors`方法来**判断队列中是否存在其他正在排队的节点**，如果是返回true，否则为false。因此当存在其他正在排队的节点，当前节点就无法获取许可，只能排队等待，这也是公平策略的体现。

## **2. release**

Semaphore提供release来释放许可。我们继续分析release方法，定义如下

```java
public void release() {
    sync.releaseShared(1);
}
```

调用AQS的`releaseShared`方法，即**释放共享式同步状态**

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        //如果释放锁成功，唤醒正在排队的节点
        doReleaseShared();
        return true;
    }
    return false;
}
```

同样该方法在[AQS详解](http://xianzilei.cn/blog/67)中有讲解过，这里也不再赘述了，我们来分析一下子类实现的`tryReleaseShared`方法，定义如下

```java
protected final boolean tryReleaseShared(int releases) {
    //自旋
    for (;;) {
        //获取许可值
        int current = getState();
        //计算释放后的许可值
        int next = current + releases;
        //如果释放后比释放前的许可值还小，直接报Error
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        //CAS修改许可值，成功则返回，失败则继续自旋
        if (compareAndSetState(current, next))
            return true;
    }
}
```

该方法也是一个**自旋**方法，通过**自旋+CAS原子性地修改同步状态**，逻辑很简单。

## **3. 其余方法**

**获取信号量的方法总共有四个**

| 方法                                  | 说明                                           |                                            |
| ------------------------------------- | ---------------------------------------------- | ------------------------------------------ |
| `acquire()`                           | **获取信号量，默认获取1个许可，响应中断**      | `sync.acquireSharedInterruptibly(1)`       |
| `acquire(int permits)`                | **获取信号量，指定获取许可的个数，响应中断**   | `sync.acquireSharedInterruptibly(permits)` |
| `acquireUninterruptibly()`            | **获取信号量，默认获取1个许可，不响应中断**    | `sync.acquireShared(1)`                    |
| `acquireUninterruptibly(int permits)` | **获取信号量，指定获取许可的个数，不响应中断** | `sync.acquireShared(permits)`              |

**释放信号量的方法有两个**

| 方法                   | 说明                                                 |
| ---------------------- | ---------------------------------------------------- |
| `release()`            | **释放信号量，默认释放一个许可**，等同于`release(1)` |
| `release(int permits)` | **释放信号量，指定释放许可的个数**                   |

获取信号量四个方法中后面三个方法原理同`acquire()`，小伙伴可以自己去源码中探究，这里就不再赘述了。我们下面看一下其余的工具方法

### 3.1 tryAcquire

`tryAcquire`方法一共有**四种重载形式**

* **tryAcquire()**

  ```java
  public boolean tryAcquire() {
      return sync.nonfairTryAcquireShared(1) >= 0;
  }
  ```

  `tryAcquire`与`acquire`方法类似，只是`tryAcquire`的意思是**尝试获取许可，如果获取成功返回true，否则返回false，不会阻塞线程，而且不响应中断**。

* **tryAcquire(int permits)**

  ```java
  public boolean tryAcquire(int permits) {
      if (permits < 0) throw new IllegalArgumentException();
      return sync.nonfairTryAcquireShared(permits) >= 0;
  }
  ```

  **同上，可以指定获取许可的个数**。

* **tryAcquire(long timeout, TimeUnit unit)**

  ```java
  public boolean tryAcquire(long timeout, TimeUnit unit)
      throws InterruptedException {
      return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
  }
  ```

  `tryAcquire`可以**指定超时时间**，它调用AQS的`tryAcquireSharedNanos`方法，即**共享式超时获取**。同样该方法在[AQS详解](http://xianzilei.cn/blog/67)中有讲解过，不再赘述。

* **tryAcquire(int permits, long timeout, TimeUnit unit)**

  ```java
  public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
      throws InterruptedException {
      if (permits < 0) throw new IllegalArgumentException();
      return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
  }
  ```

  **同上，可以指定获取许可的个数**。

### 3.2 availablePermits

**获取可用许可数**

```java
public int availablePermits() {
    //获取可用许可数
    return sync.getPermits();
}

//获取可用许可数
final int getPermits() {
    return getState();
}
```

### 3.3 drainPermits

**将剩下的信号量一次性消耗光，并且返回所消耗的信号量**

```java
public int drainPermits() {
    return sync.drainPermits();
}
```

调用Sync类的`drainPermits`方法，定义如下

```java
final int drainPermits() {
    //自旋操作
    for (;;) {
        //获取信号量值
        int current = getState();
        //如果信号量为0，直接返回
        //否则CAS修改为0，成功则返回，否则继续自旋
        if (current == 0 || compareAndSetState(current, 0))
            return current;
    }
}
```

### 3.4 reducePermits

**减少信号量的总数**。

```java
protected void reducePermits(int reduction) {
    if (reduction < 0) throw new IllegalArgumentException();
    sync.reducePermits(reduction);
}
```

调用Sync类的`reducePermits`方法，定义如下

```java
final void reducePermits(int reductions) {
    //自旋
    for (;;) {
        //获取当前信号量值
        int current = getState();
        //计算剩余许可值
        int next = current - reductions;
        if (next > current) // underflow
            throw new Error("Permit count underflow");
        //CAS修改同步状态，成功则返回，失败则继续自旋
        if (compareAndSetState(current, next))
            return;
    }
}
```

`reducePermits`方法本质就是**减少指定的信号量的值**，它和acquire方法相比都是减少信号量的值，但是`reducePermits`是**不会导致任何线程阻塞**，即只要传递的参数`reductions`（减少的信号量的数量）大于0，操作就会成功。**所以调用该方法可能会导致信号量最终为负数**。

# **五、总结**

`Semaphore`是一个**有效的流量控制工具**，它**基于AQS共享锁实现**。我们常常用它来控制**对有限资源的访问**。

* **每次使用资源前，先申请一个信号量，如果资源数不够，就会阻塞等待；**
* **每次释放资源后，就释放一个信号量**。