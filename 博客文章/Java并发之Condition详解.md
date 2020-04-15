# **一、简介**

## **1. 什么是Condition**

> 任意一个Java对象，都拥有一组监视器方法（定义在`java.lang.Object`上），主要包括`wait()`、`wait(long timeout)`、`notify()`以及`notifyAll()`方法，这些方法与`synchronized`同步关键字配合，可以实现**等待/通知模式**。`Condition`接口也提供了类似`Object`的监视器方法，与`Lock`配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。——摘自《Java并发编程的艺术》

下面是`Condition`与`Object`的监视器方法的对比（摘自《Java并发编程的艺术》）

|                         对比项                         |     Object Monitor Methods      |                          Condition                           |
| :----------------------------------------------------: | :-----------------------------: | :----------------------------------------------------------: |
|                        前置条件                        |          获取对象的锁           | 1.调用`Lock.lock()`获取锁<br>2.调用`Lock.newCondition()`获取Condition对象 |
|                        调用方式                        | 直接调用。<br>如`Object.wait()` |             直接调用。<br/>如`condition.await()`             |
|                     等待队列的个数                     |              一个               |                             多个                             |
|              当前线程释放锁并进入等待状态              |              支持               |                             支持                             |
| 当前线程释放锁并进入等待状态，且在等待状态中不响应中断 |             不支持              |                             支持                             |
|            当前线程释放锁并进入超时等待状态            |              支持               |                             支持                             |
|     当前线程释放锁并进入等待状态直到将来的某个时间     |             不支持              |                             支持                             |
|                唤醒等待队列中的一个线程                |              支持               |                             支持                             |
|                唤醒等待队列中的全部线程                |              支持               |                             支持                             |

## **2. Condition接口**

我们来看一下Condition接口的定义

```java
public interface Condition {

    //等待，当前线程接受到信号前、或中断前一直处于等待状态
    void await() throws InterruptedException;

    //等待，当前线程接受到信号前一直处于等待状态，不响应中断
    void awaitUninterruptibly();

    //等待，当前线程接受到信号前、或中断前、或到达指定等待时间之前一直处于等待状态，返回值 = nanosTimeout - 实际消耗的时间，返回值 <= 0表示超时
    long awaitNanos(long nanosTimeout) throws InterruptedException;

    //等待，当前线程接受到信号前、或中断前、或到达指定等待时间之前一直处于等待状态，返回boolean类型，表示是否在指定时间内获取到接受信号，false表示超时。
    //它与上一个方法不同在于可以自定义超时时间单位，等同于awaitNanos(unit.toNanos(time)) > 0
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    //等待，当前线程在接收到信号前、被中断或到达指定最后唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。期限之前一直处于等待状态
    boolean awaitUntil(Date deadline) throws InterruptedException;

    //唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取与Condition相关的锁。
    void signal();

    //唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取与Condition相关的锁。
    void signalAll();
}
```

`Condition`是一种广义上的**条件队列**。他为线程提供了一种**更为灵活的等待/通知模式**，线程在**调用await方法后执行挂起操作**，直到线程等待的某个条件为真时才会**被唤醒**。`Condition`必须要配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下。**一个Condition的实例必须与一个Lock绑定**，因此Condition一般都是作为Lock的内部实现。

# **二、基本使用**

以基本的**B线程唤醒A线程**为例

## **1. Object等待唤醒写法**

```java
public static void main(String[] args) throws InterruptedException {
    //对象锁
    Object obj=new Object();

    //创建线程A
    Thread threadA = new Thread(()->{
        System.out.println("A尝试获取锁...");
        synchronized (obj){
            System.out.println("A获取锁成功！");
            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println("A开始释放锁，并开始等待...");
                //线程A开始等待
                obj.wait();
                System.out.println("A被通知继续运行，直至结束。");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    });
    //创建线程B
    Thread threadB = new Thread(()->{
        System.out.println("B尝试获取锁...");
        synchronized (obj){
            System.out.println("B获取锁成功！");
            try {
                TimeUnit.SECONDS.sleep(3);
                System.out.println("B开始释放锁...");
                //线程B开始唤醒线程A
                obj.notify();
                System.out.println("B随机通知lock对象的等待队列中某个线程！");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    //启动线程A
    threadA.start();
    //这里为了是线程A先执行，避免B先执行了notify导致A永远无法被唤醒
    TimeUnit.SECONDS.sleep(1);
    //启动线程B
    threadB.start();
}
```

**执行结果如下**

```tex
A尝试获取锁...
A获取锁成功！
A开始释放锁，并开始等待...
B尝试获取锁...
B获取锁成功！
B开始释放锁...
B随机通知lock对象的等待队列中某个线程！
A被通知继续运行，直至结束。

Process finished with exit code 0
```

## **2. Condition等待唤醒写法**

```java
public static void main(String[] args) throws InterruptedException {
    //创建lock对象
    Lock lock = new ReentrantLock();
    //新建condition
    Condition condition = lock.newCondition();

    //创建线程A
    Thread threadA = new Thread(()->{
        System.out.println("A尝试获取锁...");
        lock.lock();
        try {
            System.out.println("A获取锁成功！");
            TimeUnit.SECONDS.sleep(1);
            System.out.println("A开始释放锁，并开始等待...");
            //线程A开始等待
            condition.await();
            System.out.println("A被通知继续运行...");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
            System.out.println("A线程释放了锁，执行结束！");
        }

    });
    //创建线程B
    Thread threadB = new Thread(()->{
        System.out.println("B尝试获取锁...");
        lock.lock();
        try {
            System.out.println("B获取锁成功！");
            TimeUnit.SECONDS.sleep(3);
            System.out.println("B开始释放锁...");
            //线程B开始唤醒线程A
            condition.signal();
            System.out.println("B随机通知lock对象的等待队列中某个线程！");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
            System.out.println("B线程释放了锁，执行结束！");
        }
    });
    //启动线程A
    threadA.start();
    //这里为了是线程A先执行，避免B先执行了notify导致A永远无法被唤醒
    TimeUnit.SECONDS.sleep(1);
    //启动线程B
    threadB.start();
}
```

执行结果如下

```tex
A尝试获取锁...
A获取锁成功！
A开始释放锁，并开始等待...
B尝试获取锁...
B获取锁成功！
B开始释放锁...
B随机通知lock对象的等待队列中某个线程！
B线程释放了锁，执行结束！
A被通知继续运行...
A线程释放了锁，执行结束！

Process finished with exit code 0
```

# **三、实现原理**

## **1. ConditionObject类**

`ConditionObject`是`Condition`在java并发中的具体的**实现**，由于Condition的操作需要**获取相关的锁**，而**AQS则是同步锁的实现基础**，所以`ConditionObject`则定义为**AQS的内部类**。

### 1.1 类继承关系

`ConditionObject`定义如下

```java
public class ConditionObject implements Condition, java.io.Serializable {
}
```

实现**Condition**和**Serializable接口**。

### 1.2 类的属性

**它主要包含如下属性**

```java
//序列化版本号
private static final long serialVersionUID = 1173984872572414699L;
//条件（等待）队列的第一个节点
private transient Node firstWaiter;
//条件（等待）队列的最后一个节点
private transient Node lastWaiter;
```

## **2. 等待队列**

每个`ConditionObject`都包含一个**FIFO队列**，队列中的节点类型是AQS的内部类——**Node类**，**每个节点包含着一个线程引用，该线程就是在该Condition对象上等待的线程**。与CLH同步队列不同的是，Condition的等待队列是**单向队列**，即每个节点**仅包含指向下一节点的引用**，如下图所示

![](http://img.xianzilei.cn/CLH%E9%98%9F%E5%88%97.png)

## **3. await方法解析**

调用Condition的await()方法会使当前线程进入等待状态，同时会加入到Condition等待队列同时释放锁。await方法代码如下

```java
public final void await() throws InterruptedException {
    //响应中断
    if (Thread.interrupted())
        throw new InterruptedException();
    //将当前线程加入到等待队列中
    Node node = addConditionWaiter();
    //释放锁
    int savedState = fullyRelease(node);
    //中断模式
    int interruptMode = 0;
    //检测此节点是否在同步队列中，如果不在，说明该线程还不具备竞争锁的资格，继续等待，直到检测到此节点在同步队列中
    while (!isOnSyncQueue(node)) {
        //挂起线程
        LockSupport.park(this);
        //如果发现线程已经中断了，则直接退出
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

我们来一步一步分析。**首先进行响应中断的判断**。如果没有中断异常抛出，则调用`addConditionWaiter`方法**将当前线程加入到等待队列中**，方法如下

```java
private Node addConditionWaiter() {
    //获取尾结点
    Node t = lastWaiter;
    //如果t不为空且其等待状态不为CONDITION，表示该节点不处于等待状态，需要清除掉
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        //重新获取新的尾结点
        t = lastWaiter;
    }
    //创建包含当前线程的结点，状态为CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //如果t为空，则表示队列为空，因此新节点为第一个节点
    if (t == null)
        //设置头结点
        firstWaiter = node;
    //否则队列不为空，将节点插入到最后一个位置
    else
        //节点插入到最后一个位置
        t.nextWaiter = node;
    //设置尾结点
    lastWaiter = node;
    //返回节点
    return node;
}

//清除条件队列中所有状态不为Condition的节点
private void unlinkCancelledWaiters() {
    //获取头结点
    Node t = firstWaiter;
    Node trail = null;
    //从头往尾清除掉等待状态不为CONDITION的节点
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

当前节点添加到等待队列成功后，会进行调用`fullyRelease`方法完全释放锁

```java
final int fullyRelease(Node node) {
    //是否失败标志
    boolean failed = true;
    try {
        //获取同步状态
        int savedState = getState();
        //如果释放锁成功
        if (release(savedState)) {
            //失败标志为否
            failed = false;
            //返回原同步状态
            return savedState;
        }  
        //如果释放锁失败，直接抛出异常
        else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        //如果操作失败，则将节点状态置为无效
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

释放锁成功后，会进行**自旋**，自旋中会**不断判断节点是否在同步队列中，如果不在，如果不在，说明该线程还不具备竞争锁的资格，继续等待，直到检测到此节点在同步队列中**，代码如下

```java
while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    //线程被唤醒后，需要判断在挂起的时候是否发生过中断，如果发生过中断，直接退出循环
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
}

//---------------------------以下是上述代码调用的方法的详解-----------------------------------
//返回是否发生过中断以及中断的模式
//THROW_IE：表示退出等待时重新中断
//REINTERRUPT：表示退出等待时抛出InterruptedException异常
private int checkInterruptWhileWaiting(Node node) {
    //如果发生线程被中断，执行方法transferAfterCancelledWait，否则返回0
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
    0;
}

//???该方法具体做啥的一直不理解，自己也不敢乱说，如果知道的小伙伴可以私信我
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

我们先来看一下`isOnSyncQueue`方法的实现

```java
final boolean isOnSyncQueue(Node node) {
    //能够在CLH队列的节点，节点的状态肯不为CONDITION（自旋修改成SIGNAL），并且前驱不可能为空（CLH队列非head节点肯定存在前驱），如果有一个不满足，说明节点肯定不在CLH队列中，返回false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //如果节点的后继存在，说明一定入队了
    //这是为什么呢？因为调用该方法之前node节点插入的位置是等待队列的末尾，next域肯定为null，如果不为空，说明肯定不在等待队列中，即在CLH队列中
    if (node.next != null) // If has successor, it must be on queue
        return true;
    //从尾往头查找CLH队列，看是否存在node节点
    return findNodeFromTail(node);
}

//判断CLH队列否存在node节点
private boolean findNodeFromTail(Node node) {
    //获取节点
    Node t = tail;
    //从后往前遍历查找是否存在node节点
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

这里为啥要判断是否在CLH队列中呢？**因为在执行通知操作时会将等待队列的节点转移到CLH队列中，表示这个节点可以被唤醒，任何在等待队列中需要被唤醒的节点都会在进入CLH队列中进行排队等待获取锁**。至此是当前线程被挂起的所有流程，当其他线程通知该线程唤醒的时候，就会继续执行下去，剩下的代码是三个if判断，代码如下所示

```java
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

* 第一个if判断：它调用`acquireQueued`方法，该方法是一个**自旋**的过程，即**每个线程进入同步队列中，都会自旋地观察自己是否满足条件且获取到同步状态，则就可以从自旋过程中退出，否则继续自旋下去**（代码细节已经在之前AQS文章中详解过，这里不再赘述，详情参考文章：[深入理解AQS实现原理](http://xianzilei.cn/blog/67)），返回结果表示当前是否中断过，同时再判断`interruptMode`是否不是`THROW_IE`，如果不是则会将`interruptMode`设置为`REINTERRUPT`，即等待结束后自我中断。

* 第二个if判断：**如果node的下一个等待者不为空，会清除所有状态为cancelled的节点**。

* 第三个if判断：根据中断模式进行中断操作

  ```java
  private void reportInterruptAfterWait(int interruptMode)
      throws InterruptedException {
      //如果中断模式为THROW_IE，直接抛出异常
      if (interruptMode == THROW_IE)
          throw new InterruptedException();
      //如果中断模式为REINTERRUPT，自我中断
      else if (interruptMode == REINTERRUPT)
          selfInterrupt();
  }
  ```

至此await的整个流程就走完了，我们来总结一下**await的执行流程**，前提获取到锁（这里忽略中断的细节操作）

* **1）将当前线程包装成节点插入到Condition的等待队列的队尾**
* **2）释放锁**
* **3）自旋挂起，当其他线程唤醒当前线程的时候，退出条件为节点已经被移到CLH队列中或当前节点对应的线程发生了中断，否则继续挂起**
* **4）获取锁，进行后续的操作**

## **4. signal方法解析**

调用`Condition`的`signal()`方法，将会**唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到CLH同步队列中**。代码如下

```java
public final void signal() {
    //判断当前线程是否为持有锁的线程，如果不是抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //获取头结点
    Node first = firstWaiter;
    //如果头结点不为空，唤醒头结点
    if (first != null)
        doSignal(first);
}
```

调用该方法需要**判断当前线程是否获取了锁，否则直接抛出异常**，紧接着调用方法`doSignal`唤醒队列中的头结点，代码如下

```java
private void doSignal(Node first) {
    //因为在外层已经判断过first不为空了，所以使用do..while..
    do {
        //移除first，修改头节点，同时如果此时队列为空，则需要修改尾结点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

//将节点移动到CLH同步队列中，返回是否移动成功0
final boolean transferForSignal(Node node) {
    //将节点状态由CONDITION修改为初始状态0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))		//如果CAS修改失败，返回false
        return false;

    //将节点加入到CLH队列中，返回的是CLH队列中node节点前面的一个节点
    Node p = enq(node);
    //获取p的状态
    int ws = p.waitStatus;
    //如果节点p的状态大于0（cancel状态）直接唤醒node节点对应的线程，否则CAS修改节点p的状态为SIGNAL，修改失败进行唤醒，否则不唤醒
    //这块猜测是如果设置失败避免没有其他节点唤醒这个线程，自己先唤醒一下
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        //唤醒线程
        LockSupport.unpark(node.thread);
    return true;
}
```

整个`signal`的流程较为简单，我们来总结一下

* **判断当前线程是否是持有锁的线程，如果不是则抛出异常，否则继续执行**
* **将Condition对应的等待队列的头结点移到到CLH队列中去**

## **5. 总结**

**总的来说，Condition的实现就是在其中新增一个等待队列。当调用 await方法，就会将当前线程放入到这个等待队列的队尾，同时当前线程挂起，等待其他线程唤醒。当其他线程调用 signal 方法时，就会从 等待队列中取出一个线程并插入到CLH队列中，之后就是常规的AQS获取锁流程。**

其余方法如

```java
public final void awaitUninterruptibly(){...}
public final long awaitNanos(long nanosTimeout){...}
public final boolean await(long time, TimeUnit unit){...}
public final boolean awaitUntil(Date deadline){...}
public final void signalAll(){...}
```

**这些方法的逻辑与await和signal基本差不多，只是增加了一些中断的忽略和超时判断**，篇幅有限，这里就不赘述了，感兴趣的小伙伴可以自行研究。