# **一、简介**

## **1. 什么是读写锁**

前面我们介绍了`ReentrantLock`类，知道它的功能类似`synchronized`关键字，属于排它锁，同一时刻仅有一个线程可以进行访问。但是在大部分的场景下，读操作往往远大于写操作，其中**读操作之前不存在数竞争关系，而写操作与其余的读写操作互斥**。因此JDK为我们提供了**读写锁**——`ReentrantReadWriteLock`。

**读写锁**维护着一对锁，一个**读锁**和一个**写锁**。通过分离**读锁**和**写锁**，使得并发性比一般的排他锁有了较大的提升：在同一时间可以**允许多个读线程同时访问**，但是在**写线程访问时，所有读线程和写线程都会被阻塞**。

## **2. 主要特性**

* **公平性**

  支持**非公平（默认）**和**公平**的锁获取方式，其中非公平性能优于公平。

* **重入性**

  **支持重入**。即读线程获取读锁之后能够再次获取读锁；写线程获取写锁之后能够再次获取写锁，同时也可以获取读锁。

* **锁降级**

  遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级为读锁。

## **3. 获取读写锁的前提条件**

* 线程**获取读锁**的前提条件
  * **没有其他线程的写锁**
  * **没有写请求**或者有**写请求，但是调用线程和持有锁的线程是同一个**
* 线程**获取写锁**的前提条件
  * **没有其他线程的读锁**
  * **没有其他线程的写锁**

# **二、类总览**

![](http://img.xianzilei.cn/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.png)

## **1. 类的继承关系**

```java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable
```

`ReentrantReadWriteLock`实现了`ReadWriteLock`接口，`ReadWriteLock`接口定义如下

```java
public interface ReadWriteLock {
    //返回读锁
    Lock readLock();

    //返回写锁
    Lock writeLock();
}
```

## **2. 类的内部类**

ReentrantReadWriteLock有五个内部类，五个内部类之间也是相互关联的。内部类的关系如下图所示

![](http://img.xianzilei.cn/ReentrantReadWriteLock%E4%BA%94%E4%B8%AA%E5%86%85%E9%83%A8%E7%B1%BB.png)

如上图所示，**Sync继承自AQS、NonfairSync和FairSync继承Sync类、FairSync继承自Sync类；ReadLock和WriteLock实现Lock接口**。

### **2.1 Sync类**

```java
abstract static class Sync extends AbstractQueuedSynchronizer{}
```

**Sync类继承AQS**，它的**内部也存在两个内部类**，分别为`HoldCounter`和`ThreadLocalHoldCounter`

```java
//计数器，主要与读锁配套使用
static final class HoldCounter {
    //计数
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    //获取当前线程的TID属性，该字段可以用来唯一标识一个线程
    final long tid = getThreadId(Thread.currentThread());
}

//本地线程计数器
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    // 重写初始化方法，在没有进行set的情况下，获取的都是该HoldCounter值
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

**Sync类的属性如下所示**

```java
//序列化版本号
private static final long serialVersionUID = 6317671515068378041L;
//表示高16位为读锁状态，低16位为写锁状态
static final int SHARED_SHIFT   = 16;
//读锁单位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//读锁的最大数量
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
//写锁最大数量
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
//本地线程计数器
private transient ThreadLocalHoldCounter readHolds;
//缓存的计数器
private transient HoldCounter cachedHoldCounter;
//第一个读线程
private transient Thread firstReader = null;
//第一个读线程的计数
private transient int firstReaderHoldCount;
```

这里需要说明一下，在读写锁中，读锁和写锁的同步状态公用一个整型变量，通过按位切割使用，即将该整型变量切割成两部分，高16位表示读，低16位表示写，如下图所示

![](http://img.xianzilei.cn/%E8%AF%BB%E5%86%99%E9%94%81%E7%8A%B6%E6%80%81%E5%8F%98%E9%87%8F%E6%A0%87%E8%AF%86.png)

例如当前同步状态值为S，则读写状态的获取与操作如下所示

* **获取写状态**

  **S&0x0000FFFF**，即将高16位全部抹去

* **获取读状态**

  **S>>>6**：无符号补0，右移16位

* **写状态加1**

  **S+1**

* **读状态加1**

  **S+(1<<16)**：即S+0x00010000

### **2.2 NonfairSync类**

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        //
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

### **2.3 FairSync类**

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

### **2.4 ReadLock类**

```java
public static class ReadLock implements Lock, java.io.Serializable {
    //序列化版本号
    private static final long serialVersionUID = -5992448646407690164L;
    //Sync实例
    private final Sync sync;

    //构造方法
    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    //获取锁
    public void lock() {
        sync.acquireShared(1);
    }

    //可中断式获取锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //尝试获取锁
    public boolean tryLock() {
        return sync.tryReadLock();
    }

    //超时方式获取锁
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public void unlock() {
        sync.releaseShared(1);
    }

    //创建Condition
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }

    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
```

**ReadLock实现Lock接口，定义了一系列读锁的获取、超时获取、可中断获取和释放锁等操作**。

### **2.5 WriteLock类**

```java
public static class WriteLock implements Lock, java.io.Serializable {
    //序列化版本号
    private static final long serialVersionUID = -4992448646407690164L;
    //Sync实例
    private final Sync sync;

    //构造方法
    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    //获取锁
    public void lock() {
        sync.acquire(1);
    }

    //可中断式获取锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    //尝试获取锁
    public boolean tryLock( ) {
        return sync.tryWriteLock();
    }

    //超时方式获取锁
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public void unlock() {
        sync.release(1);
    }

    //新建Condition
    public Condition newCondition() {
        return sync.newCondition();
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

    //判断当前线程是否是当前锁的持有者
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    //获取当前线程对当前写锁的持有数
    public int getHoldCount() {
        return sync.getWriteHoldCount();
    }
}
```

**WriteLock实现Lock接口，定义了一系列写锁的获取、超时获取、可中断获取和释放锁等操作**。

## **3. 类的属性**

```java
//序列化版本号
private static final long serialVersionUID = -6992448646407690164L;
//读锁
private final ReentrantReadWriteLock.ReadLock readerLock;
//写锁
private final ReentrantReadWriteLock.WriteLock writerLock;
//同步队列
final Sync sync;

// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
//线程ID的偏移地址
private static final long TID_OFFSET;
static {
    try {
        //获取Unsafe类实例
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        //获取线程tid字段的内存地址
        TID_OFFSET = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("tid"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

可以看到`ReentrantReadWriteLock`属性包括了一个`ReentrantReadWriteLock`.`ReadLock`对象，表示**读锁**；一个`ReentrantReadWriteLock`.`WriteLock`对象，表示**写锁**；一个Sync对象，表示**同步队列**。

## **4. 构造方法**

* **无参构造方法**

  ```java
  public ReentrantReadWriteLock() {
      this(false);
  }
  ```

  **默认调用有参构造方法，传递参数false，表示默认使用非公平策略**。

* **有参构造方法**

  ```java
  public ReentrantReadWriteLock(boolean fair) {
      //fair表示使用公平或者非公平策略，并实例化sync
      sync = fair ? new FairSync() : new NonfairSync();
      //实例化读锁，并把sync传递给readerLock中的sync属性
      readerLock = new ReadLock(this);
      //实例化写锁，并把sync传递给writerLock中的sync属性
      writerLock = new WriteLock(this);
  }
  ```

`ReentrantReadWriteLock`实例化就是**根据参数设置公平或非公平的策略，同时实例化同步队列以及读写锁**。接下来我们来分析读写锁实际获取和释放的详细流程。

# **三、写锁**

写锁就是一个支持**可重入的排他锁**。

## **1. 写锁的获取**

写锁的获取调用`WriteLock`内部类中的lock方法

```java
public void lock() {
    sync.acquire(1);
}
```

调用方法acquire，代码如下

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

很明显这是AQS中**获取独占锁**的代码，我们知道AQS会调用子类的`tryAcquire`方法，代码如下

```java
protected final boolean tryAcquire(int acquires) {
    //获取当前线程
    Thread current = Thread.currentThread();
    //获取当前同步状态
    int c = getState();
    //计算写线程数量（即获取独占锁的重入数）
    int w = exclusiveCount(c);
    //如果同步状态不为0，则表示已经有其他线程获取了读锁或者写锁
    if (c != 0) {
        //如果此时w为0，表示读锁被占用，直接返回false，无法获取锁
        //如果此时w不为0，说明存在写锁，因此再判断占用写锁的线程是否是当前线程，如果不是，则无法获取写锁，直接返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //执行到这一步说明写锁占用且是当前线程持有，因此可以获取写锁（重入），但是需要判断是否超过最高写线程数量
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 写锁的重入
        setState(c + acquires);
        return true;
    }
    //能够执行到这说明同步状态为0，表示当前没有任何读锁和写锁
    //writerShouldBlock表示当前线程是否可以获取锁，false表示可以，true无法获取锁
    //如果是非公平锁，则writerShouldBlock始终返回false，因为不需要排队，非公平策略下可直接参与抢锁
    //如果是公平锁，则需要查看当前队列中是否存在其他排队的线程，如果存在则返回true，当前线程不能获取写锁，否则返回false可直接CAS获取锁
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    //能够执行到这说明当前线程已经获取到了写锁，设置锁的持有线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}

//获取写锁状态，即截取c的低16位
static int exclusiveCount(int c) { 
    return c & EXCLUSIVE_MASK; 
}

//NonfairSync中writerShouldBlock方法
final boolean writerShouldBlock() {
    //非公平策略下无须排队，可直接参与锁的争夺
    return false; // writers can always barge
}

//FairSync中writerShouldBlock方法
final boolean writerShouldBlock() {
    //公平策略下需要判断是否有其他正在排队的线程，如果有则需要排队等待，否则可以参与获取锁
    return hasQueuedPredecessors();
}
```

**写锁的tryAcquire方法流程如下**

![](http://img.xianzilei.cn/%E5%86%99%E9%94%81%E8%8E%B7%E5%8F%96%E6%B5%81%E7%A8%8B.png)

## **2. 写锁的释放**

写锁的释放调用`WriteLock`内部类中的`unlock`方法

```java
public void unlock() {
    sync.release(1);
}
```

**调用方法release**，代码如下

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

同样这也是AQS中的**释放独占锁**的代码，它会**调用子类的tryRelease方法**，代码如下

```java
protected final boolean tryRelease(int releases) {
    //如果写锁的持有者不是当前线程，直接抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //计算释放后的写锁数
    int nextc = getState() - releases;
    //free表示写锁是否被完全释放
    boolean free = exclusiveCount(nextc) == 0;
    //如果写锁状态为0，则表示写锁被完全释放，即独占模式被释放
    if (free)
        //设置独占线程（锁的持有者）为null
        setExclusiveOwnerThread(null);
    //设置新的写锁数
    setState(nextc);
    //返回独占模式是否被释放
    return free;
}
```

**写锁的tryRelease方法流程如下**

![](http://img.xianzilei.cn/%E5%86%99%E9%94%81%E9%87%8A%E6%94%BE%E6%B5%81%E7%A8%8B.png)

# **四、读锁**

读锁为一个可重入的共享锁，它能够被多个线程同时持有，在没有其他写线程访问时，读锁总是或获取成功。

## **1. 读锁的获取**

读锁的释放调用`ReadLock`内部类中的`lock`方法

```java
public void lock() {
    sync.acquireShared(1);
}
```

**调用方法acquireShared**，代码如下

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

这是AQS中**获取共享锁**的代码，它会调用子类的`tryAcquireShared`方法，代码如下

```java
protected final int tryAcquireShared(int unused) {
    //获取当前线程
    Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //exclusiveCount获取写线程数量
    //如果存在写锁，且持有锁的线程不是当前线程，则获取失败，返回-1
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    //计算原读锁数量
    int r = sharedCount(c);
    //如果同时满足下面三个条件，进入if块内
    //1.读线程不需要等待
    //2.读锁数量小于最大数量
    //3.CAS获取同步状态成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        //能够执行到此处表示获取读锁成功
        //如果原读锁数量为0，设置第一个读线程和该线程占用的资源数
        if (r == 0) {
            //设置第一个读线程为当前线程
            firstReader = current;
            //设置第一个读线程占用的资源数为1
            firstReaderHoldCount = 1;
        } 
        //如果原读锁数量不为0，且第一个读线程即为当前线程，更新该线程占用的资源数
        else if (firstReader == current) {
            //占用的资源数加1
            firstReaderHoldCount++;
        } 
        //如果原读锁数量不为0，且第一个读线程不为当前线程
        else {
            //获取计数器
            HoldCounter rh = cachedHoldCounter;
            //如果计数器为空，或者计数器的tid不为当前线程的tid
            if (rh == null || rh.tid != getThreadId(current))
                // 获取当前线程对应的计数器
                cachedHoldCounter = rh = readHolds.get();
            //如果当前线程的计数为0
            else if (rh.count == 0)
                ////设置readHolds中值为当前线程的计数器
                readHolds.set(rh);
            //count加1
            rh.count++;
        }
        return 1;
    }
    //如果上面三个条件有一个不满足
    return fullTryAcquireShared(current);
}

//NonfairSync中readerShouldBlock方法
//判断读线程是否需要等待
final boolean readerShouldBlock() {
    //
    return apparentlyFirstQueuedIsExclusive();
}
//
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}

//FairSync中readerShouldBlock方法
//判断读锁是否需要等待
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```

我们来分析一下该方法最后一行的fullTryAcquireShared方法，代码如下

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    //自旋
    for (;;) {
        //获取同步状态
        int c = getState();
        //如果写锁数量不为0
        if (exclusiveCount(c) != 0) {
            //如果持有锁线程不为当前线程，直接返回-1
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } 
        //如果写锁数量为0，且读线程需要被阻塞
        else if (readerShouldBlock()) {
            // 如果当前线程是第一个读线程，不操作
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } 
            //如果当前线程不是第一个读线程
            else {
				//如果计数器为空
                if (rh == null) {
                    rh = cachedHoldCounter;
                    //计数器还是为空或者计数器的tid不为当前正在运行的线程的tid
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        //如果读锁数量大于最大值，直接抛出异常
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //如果CAS设置状态成功
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            //如果读线程数量为0，逻辑同上
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

在`tryAcquireShared`函数中，如果下列三个条件不满足(读线程是否应该被阻塞、小于最大值、比较设置成功)则会进行`fullTryAcquireShared`函数中，它用来保证相关操作可以成功。

## **2. 读锁的释放**

读锁的释放调用`ReadLock`内部类中的`unlock`方法

```java
public void unlock() {
    sync.releaseShared(1);
}
```

**调用方法releaseShared**，代码如下

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

这是AQS中**释放共享锁**的代码，它会调用子类的`tryReleaseShared`方法，代码如下

```java
protected final boolean tryReleaseShared(int unused) {
    //获取当前线程
    Thread current = Thread.currentThread();
    //如果当前线程是第一个读线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        //如果当前线程的计数为1，释放后需要清除
        if (firstReaderHoldCount == 1)
            firstReader = null;
        //否则直接-1
        else
            firstReaderHoldCount--;
    } 
    //如果当前线程不是第一个读线程
    else {
        //获取缓存的计数器
        HoldCounter rh = cachedHoldCounter;
        //如果计数器为空或者计数器的tid不为当前正在运行的线程的tid
        if (rh == null || rh.tid != getThreadId(current))
            //获取当前线程的计数器
            rh = readHolds.get();
        //获取计数
        int count = rh.count;
        //如果计数小于等于1，则需要移除该线程对应的计数器
        if (count <= 1) {
            readHolds.remove();
            //如果计数小于等于0，直接抛出异常
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        //计数减一
        --rh.count;
    }
    //自旋操作
    for (;;) {
        //获取状态
        int c = getState();
        //计算释放锁后的状态
        int nextc = c - SHARED_UNIT;
        //CAS更新状态
        if (compareAndSetState(c, nextc))
            //CAS更成功后返回状态是否是0，即读锁是否完全被释放
            return nextc == 0;
    }
}
```

## **3. HoldCounter的理解**

在读锁的获取和释放的时候，我们可以看到一系列对`HoldCounter`的操作。直接看代码可能会比较抽象，不易理解。这一节我们跳出代码，从**宏观的角度**理解一下`HoldCounter`的作用。

在读锁的获取、释放过程中，基本操作可以总结为线程获取读锁时`HoldCounter`+1，释放读锁时`HoldCounter`-1。我们知道**读锁的内在实现机制就是共享锁**，但是这个共享锁本身并不像一个锁，它更像一个**计数器**的概念。**一次共享锁操作就相当于一次计数器的操作，获取共享锁计数器+1，释放共享锁计数器-1**。只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以`HoldCounter`的作用就是**当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常，因此就使用到了ThreadLocal来与各个线程绑定在一起**。

我们再来看一下Sync的两个内部类——`HoldCounter`和`ThreadLocalHoldCounter`

```java
static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    final long tid = getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

`HoldCounter`中仅有`count`和`tid`两个变量，其中`count`代表着**计数**，`tid`是**线程的id**，但是仅仅有这一个对象无法和线程绑定，因此ThreadLocal就起到了作用。`ThreadLocalHoldCounter`继承`ThreadLocal`，里面保存各个线程的`HoldCounter`对象。

其中有一点需要注意，`HoldCounter`绑定**线程id**而不绑定线程对象，这是为什么呢？原因就是避免`HoldCounter`和`ThreadLocal`互相绑定而GC难以释放它们（尽管GC能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了**帮助GC快速回收对象而已**。

还有一点需要说明，我们在获取锁和释放锁的过程中发现有对应**第一个读线程和该线程的计数**的操作，他们是Sync内部类中的两个属性

```java
//第一个读线程
private transient Thread firstReader = null;
//第一个读线程的计数
private transient int firstReaderHoldCount;
```

那么为什么要加这两个属性呢，每次去`ThreadLocal`中取计数信息不就行了？这是为了一个效率问题，`firstReader`是不会放入到`readHolds`中的，如果读锁仅有一个的情况下就会避免查找`readHolds`。

# **五、锁降级**

在开篇我们介绍到读写锁有一大特性就是**锁降级**。

> 锁降级指的是**写锁降级成为读锁**。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。**锁降级是指把持住(当前拥有的)写锁，再获取到读锁，随后释放(先前拥有的)写锁，从而使写锁降级为读锁的过程**。

以下是oracle官网的对于锁降级的示例代码

```java
class CachedData {
    //共享数据
    Object data;
    //数据是否可用，使用volatile修饰，保证其可见性
    volatile boolean cacheValid;
    //读写锁
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    void processCachedData() {
        //首先获取读锁
        rwl.readLock().lock();
        //如果数据不可用，尝试修改数据使其可用
        if (!cacheValid) {
            // 则释放读锁
            rwl.readLock().unlock();
            //获取写锁
            rwl.writeLock().lock();
            try {
                //此时获取到了写锁，尝试改数据前先判断数据是否可用，如果不可用则修改数据
                if (!cacheValid) {
                    //修改数据操作
                    data = ...;
                    //设置数据可用
                    cacheValid = true;
                }
                //释放写锁的前提先获取读锁
                rwl.readLock().lock();
            } finally {
                //释放写锁，此时锁降级，由写锁降级为读锁
                rwl.writeLock().unlock(); // Unlock write, still hold read
            }
        }

        try {
            //使用数据
            use(data);
        } finally {
            //使用完数据后释放读锁
            rwl.readLock().unlock();
        }
    }
}
```

多个线程并发调用方法`processCachedData`，流程如下

* 1）**首先获取读锁**，获取读锁成功后查看data是否可用，
  * 如果可用则直接读取使用，结束后释放读锁，流程结束；
  * 否则数据失效，直接**释放读锁**。
* 2）**获取写锁**，获取成功需要继续判断data是否可用（多线程情况下可能别的线程修改了data使其可用）
  * 如果data可用，则自己无需修改data，直接**加上读锁**，接着**释放写锁**（此时**写锁降级为读锁**），使用完数据后再**释放读锁**，流程结束。
  * 如果data还是不可用，则**修改data**，将`cacheValid`置为true，接着**加上读锁**，接着**释放写锁**（此时**写锁降级为读锁**），使用完数据后再释放读锁，流程结束。

这里需要注意的是获取写锁之后，修改完数据不能直接释放写锁，而是需要**先加读锁后再释放写锁**。这是因为如果先释放写锁。则当前数据处于未加锁状态，**可能另外的线程此时操作数据，修改了数据而当前线程无法感知，从而造成可能的未知错误**。如果遵循上述代码的操作顺序，先获取读锁，其余线程无法获取写锁，数据不会被别的线程修改，从而保证数据的安全性。