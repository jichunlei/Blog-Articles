# **一、简介**

## **1. Lock接口**

在讲`ReentrantLock`前，我们先熟悉一下`Lock`接口。在`Lock`接口出现之前，`Java`程序主要是靠`synchronized`关键字实现锁功能的，而JDK1.5之后，并发包中增加了`Lock`接口，它提供了与`synchronized`一样的锁功能。虽然它失去了像synchronize关键字**隐式加锁解锁**的便捷性，但是却拥有了**锁获取和释放的可操作性，可中断的获取锁以及超时获取锁**等多种`synchronized`关键字所不具备的同步特性。`Lock`接口的定义如下

```java
public interface Lock {

    //获取锁，若当前锁被其他线程获取，则此线程会阻塞等待lock被释放
    //使用lock方法需要显式地去释放锁，即使发生异常时也不会自动释放锁。
    void lock();

    //作用同lock方法，并且在获取锁的过程中可以响应中断
    void lockInterruptibly() throws InterruptedException;

    //尝试获取锁，获取成功返回true，失败则返回false
    //即该方法不会导致线程阻塞，无论如何都会返回
    boolean tryLock();

    //作用同tryLock方法，如果获取到锁直接返回true
    //新增的特性是如果获取不到锁，会等待一段时间，且在等待的过程可以响应中断，一旦超过等待时间仍获取不到锁，就返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    //释放锁
    void unlock();

    //返回一个绑定该lock的Condition对象，这个后续在Condition专题会详细讲解，本文先不赘述
    Condition newCondition();
}
```

`ReentrantLock`完全**实现了Lock接口**，也是JDK中**唯一**实现Lock接口的类（其余都是一些内部类）。Lock接口的基本模板使用如下所示

```java
//创建lock实例
Lock lock = new ReentrantLock();
try {
    //加锁
    lock.lock();
    // do something...
} finally {
    //解锁
    lock.unlock();
}
```

## **2. 什么是ReentrantLock**

> ReentrantLock是一个**可重入且独占式**的锁，是一种**递归无阻塞**的同步机制。它**支持重复进入锁，即该锁能够支持一个线程对资源的重复加锁**。除此之外，该锁的还支持获取锁时的**公平**和**非公平性**选择。

## 3. 可重入概念

重进入是指**任意线程在获取到锁之后能够再次获取该锁而不会被锁阻塞**，该特性的首先需要解决以下两个问题：

* **线程再次获取锁**：所需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次获取成功；
* **锁的最终释放**：线程重复n次获取了锁，随后在第n次释放该锁后，其它线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前线程被重复获取的次数，而被释放时，计数自减，当计数为0时表示锁已经成功释放。

## **4. ReentrantLock和Synchronized对比**

### **4.1 共同点**

* **1）都可以协调多线程对共享对象、变量的访问**
* **2）都是可重入的，同一个线程可以多次获得同一个锁**
* **3）都是阻塞式地同步，即一个线程获取了锁，其他访问该锁的线程必须阻塞在同步块外等待**
* **4）性能方面：Synchronized在JDK1.6的一波优化后性能与lock差别不大**

### **4.2 区别**

* **1）ReentrantLock需要自己手动地获取和释放锁，而synchronized关键字可以隐式获得和释放，无需用户操心。**
* **2）ReentrantLock具有响应中断，超时获取，公平非公平锁和利用Condition绑定多个条件等特性，而synchronized不具备这些特性**
* **3）ReentrantLock是API级别的，而synchronized是JVM级别的**
* **4）ReentrantLock底层实现采用的是同步非阻塞，乐观并发（CAS）策略。而synchronized是同步阻塞，悲观并发。**

# **二、类总览**

![](http://img.xianzilei.cn/ReentrantLock%E7%B1%BB%E5%9B%BE.png)

## **1. 类的继承关系**

**ReentrantLock实现类Lock和Serializable接口**。

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

## **2. 类的内部类**

`ReentrantLock`总共有**三个内部类**，它们之间的关系如下所示

![](http://img.xianzilei.cn/ReentrantLock%E5%86%85%E9%83%A8%E7%B1%BB%E5%85%B3%E7%B3%BB.png)

`ReentrantLock`类内部总共存在**Sync**、**NonfairSync**、**FairSync**三个类，其中`NonfairSync`与`FairSync`类继承自`Sync`类，`Sync`类继承自`AbstractQueuedSynchronizer`抽象类。

### **2.1 Sync**

**Sync类继承自AQS**，它有两个子类，分别实现**公平锁和非公平锁**

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    //序列化版本号
    private static final long serialVersionUID = -5179523762034025860L;

    //获取锁，需要子类自己实现
    abstract void lock();

    //非公平方式获取锁
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    //释放锁
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    //判断资源释放被当前线程占用
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    //创建一个新条件
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // 返回占有资源的线程
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    //返回状态
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    //判断资源是否被占用
    final boolean isLocked() {
        return getState() != 0;
    }

    //自定义反序列化
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

### **2.2 NonfairSync**

**NonfairSync类继承了Sync类，表示采用非公平策略获取锁，其实现了Sync类中抽象的lock方法**

```java
static final class NonfairSync extends Sync {
    //序列化版本号
    private static final long serialVersionUID = 7316153563782823691L;

    //获取锁
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    //尝试获取锁
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

方法细节后面详细说明

### **2.3 FairSync**

**FairSync类也继承了Sync类，表示采用公平策略获取锁，其也实现了Sync类中的抽象lock方法**

```java
static final class FairSync extends Sync {
    //序列化版本号
    private static final long serialVersionUID = -3000897897090466540L;

    //获取锁
    final void lock() {
        acquire(1);
    }

    //尝试公平获取锁
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

方法细节后面详细说明

## 3. 类的属性

```java
//序列化版本号
private static final long serialVersionUID = 7373984872572414699L;
//同步器，继承自AQS，提供各种核心操作
private final Sync sync;
```

**sync属性非常重要，大部分的锁操作直接转化为该属性的方法调用**。

# **三、锁类型**

`ReentrantLock` 分为**公平锁**和**非公平锁**，分别有它的两个内部类`FairSync`和`NonfairSync`实现。

* **公平锁**

  加锁前检查是否有排队等待的线程，优先排队等待的线程，先来先得，即遵循FIFO原则。

* **非公平锁**

  加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待

**是否公平可以通过构造方法指定**。

```java
//无参构造函数，默认为非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

//有参构造函数，传递boolean类型参数fair
//fair=false：非公平锁
//fair=true：公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

注：公平锁为了保证时间上的绝对顺序，需要**频繁的上下文切换**，而非公平锁会降低一定的上下文切换，降低性能开销。因此，`ReentrantLock`默认选择的是非公平锁，则是为了**减少一部分上下文切换**，**保证了系统更大的吞吐量**。

# **四、获取锁**

在讲解`ReentrantLock`的源码前，需要大家对AQS的源码熟悉，如果不了解AQS的小伙伴，可以先阅读之前的博文：[深入理解AQS实现原理](http://xianzilei.cn/blog/67)

## **1. 公平锁的获取**

我们知道，lock的获取锁一般使用如下方法

```java
lock.lock();
```

那我们直接沿着该方法往下看，先看lock方法的实现

```java
public void lock() {
    sync.lock();
}
```

调用**属性sync的lock方法**，继续深入（此时是公平锁，所以调用的是`FairSync`对象中的`lock`方法）

```java
final void lock() {
    acquire(1);
}
```

继续查看acquire方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

看到这里想必大家已经很熟悉了，这就是AQS的acquire方法，这里就不再赘述，大家可以参考之前的AQS源码分析文章。但是我们需要**分析一下tryAcquire方法，因为该方法有ReentrantLock中FairSync类自己实现的**，源码如下

```java
protected final boolean tryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //如果同步状态为0表示当前资源空闲
    if (c == 0) {
        //在CAS获取锁前需要调用方法hasQueuedPredecessors来判断队列中是否存在其他正在排队的节点，如果存在返回true，不仅如此if块，否则返回false，进行CAS操作
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            //设置当前资源的持有线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果c不等于0，且当前持有锁线程不等于当前线程，直接返回false，加锁失败
    //如果c不等于0，且当前持有锁的线程等于当前线程，表示这是一次重入，就会把状态+1，结束后返回true，即重入锁返回true
    //这块代码侧面说明Reentrantlock是可以重入的
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这里需要详细说明一下`hasQueuedPredecessors`方法（该方法的代码也是整个`Reentrantlock`代码中的精华部分之一），需要注意一下，**该方法如果返回false表示当前线程可以直接获取锁，否则获取锁失败**。代码如下

```java
public final boolean hasQueuedPredecessors() {
	//保存尾结点
    Node t = tail; 
    //保存头结点
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

重点看一下return后面的判断

```java
h != t &&((s = h.next) == null || s.thread != Thread.currentThread())
```

* 先看条件`h != t`
  * 如果`h=t`，大概分为以下两种情况
    * **队列未初始化**，此时`h=t=null`，说明**肯定没有排队等待的线程，可以直接获取锁**，所以返回false
    * **队列已经初始化** ，但是不存在排队等待的线程。我们知道队列在初始化的时候会新建个节点（thread=null）作为头结点，往后需要排队的线程链接到这之后。因此**当队列只有一个节点**时，满足`h=t`，此时返回false，可以直接获取锁
  * 如果`h!=t`，说明队列节点个数大于1，即肯定存在排队等待的线程，此时`h!=t`返回true，需要继续进行下一步判断
* 再看条件`(s = h.next) == null || s.thread != Thread.currentThread()`
  * 先看`(s = h.next) == null`：通过上面的分析我们知道队列的节点个数大于1，所以s（即头结点的下一节点）肯定不为空，返回false；
  * 再看`s.thread != Thread.currentThread()`：能够执行到此处，整个return后面的判断语句的结果直接取决于当前判断。这里判断s节点对应的线程是否是当前线程，如果是说明此时排队的线程就是当前线程，即可以获取锁，返回false。如果不是则表示当前还没有轮到我（当前线程）获取锁，返回true。

总结一下`hasQueuedPredecessors`方法：该方法的作用就是**查看是否有排队的线程，如果没有直接返回false，代表当前线程可以获取锁，否则再查看第一个排队的线程是否是自己，如果是则返回false，可以获取锁，否则就获取锁失败，进入后续的入队操作**。

## **2. 非公平锁的获取**

非公平锁的获取，同样我们来到`NonfairSync`对象中的`lock`方法

```java
final void lock() {
    //直接尝试获取锁
    if (compareAndSetState(0, 1))
        //获取锁成功后
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

非公平锁的lock方法**一开始就尝试获取CAS获取锁，如果CAS失败再调用方法acquire去进行相应的排队操作**。

# **五、释放锁**

**公平锁和非公平锁的释放流程都是一样的**

```java
public void unlock() {
    sync.release(1);
}
```

调用AQS中的release方法

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

释放锁的流程参看之前AQS详解的博文（[深入理解AQS实现原理](http://xianzilei.cn/blog/67)），这里不再赘述。这里我们详细讲解一下`tryRelease`方法，代码如下

```java
protected final boolean tryRelease(int releases) {
    //计算释放后state值
    int c = getState() - releases;
    //如果获取锁的线程不是当前线程，直接抛异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //锁被重入的次数为0，表示锁被释放，清空独占线程
    if (c == 0) {
        free = true;
        //清空独占线程
        setExclusiveOwnerThread(null);
    }
    //设置state值
    setState(c);
    //返回锁是否被释放
    return free;
}
```

方法详解见注释，这里需要注意一下，调用unlock方法前需要保证锁的占有者必须是当前方法的调用者，否则或抛出`IllegalMonitorStateException`异常。