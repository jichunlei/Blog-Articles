# **一、AQS简介**

## **1. 什么是AQS**

**AQS全称为AbstractQueuedSynchronizer**，翻译过来就是**抽象队列同步器**。AQS是**一个用来构建锁和其他同步组件的基础框架**，使用`AQS`可以简单且高效地构造出应用广泛的同步器，例如我们后续会讲到的`ReentrantLock`、`Semaphore`、`ReentrantReadWriteLock`和`FutureTask`等等。

## **2. AQS的核心思想**

> AQS核心思想是，如果被请求的共享资源**空闲**，则将当前请求资源的线程设置为**有效的工作线程**，并且将共享资源设置为**锁定状态**。如果被请求的共享资源**被占用**，那么就需要一套**线程阻塞等待以及被唤醒时锁分配的机制**，这个机制AQS是用**CLH队列锁**实现的，即将暂时获取不到锁的线程加入到队列中。

那么CLH队列又是什么呢？

> CLH(Craig,Landin,and Hagersten)队列是一个**虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)**。**AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配**。

AQS中的共享资源是使用一个**int成员变量来表示同步状态**，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。状态信息通过procted类型的getState，setState，compareAndSetState进行操作。

```java
//同步状态，使用volatile来保证其可见性
private volatile int state;

//获取同步状态
protected final int getState() {
    return state;
}

//设置同步状态
protected final void setState(int newState) {
    state = newState;
}

//原子性地修改同步状态
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

## **3. 资源的共享方式**

AQS定义**两种**资源共享方式

* **Exclusive(独占)**：**只能有一个线程能执行**，如ReentrantLock。又可分为**公平锁**和**非公平锁**： 
  - **公平锁**：**按照线程在队列中的排队顺序，先到者先拿到锁**
  - **非公平锁**：**当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的**
* **Share(共享)**：**多个线程可同时执行**，如`Semaphore`、`CountDownLatch`等。

## **4. AQS的设计模式**

> AQS的设计是使用模板方法设计模式，它将**一些方法开放给子类进行重写，而同步器给同步组件所提供模板方法又会重新调用被子类所重写的方法**。

自定义同步器时需要重写下面几个AQS提供的模板方法

```java
//该线程是否正在独占资源。只有用到condition才需要去实现它。
isHeldExclusively();
//独占方式。尝试获取资源，成功则返回true，失败则返回false
tryAcquire(int);
//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryRelease(int);
//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryAcquireShared(int);
//共享方式。尝试释放资源，成功则返回true，失败则返回false。
tryReleaseShared(int);
```

上面的方法均**使用protect修饰**，且**默认实现都会抛出UnsupportedOperationException异常**。这些方法的实现必须是内部线程安全的。AQS类中的其他方法都是final ，所以无法被其他类覆盖。

# **二、CLH队列**

上面我们说到，**当共享资源被某个线程占有，其他请求该资源的线程会被阻塞，从而进入同步队列**。AQS底层的数据结构采用**CLH队列**，它是一个虚拟的双向队列，即**不存在队列的实例，仅存在节点之间的关联关系**。**AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点(Node)来实现锁的分配**。注：**Sync queue，即同步队列**，是**双向链表**，包括`head`结点和`tail`结点，`head`结点主要用作后续的调度。而**Condition queue**不是必须的，它是一个**单向链表**，只有当使用`Condition`时，才会存在此单向链表，并且可能会有多个`Condition queue`。

![](http://img.xianzilei.cn/CLH%E9%98%9F%E5%88%97.png)

其中**节点的类型是AQS的静态内部类Node**，代码如下

```java
static final class Node {
    //模式，分为共享与独占
    //共享模式
    static final Node SHARED = new Node();
    //独占模式
    static final Node EXCLUSIVE = null;

    //节点状态值：1，-1，-2，-3
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    //节点状态
    volatile int waitStatus;

    //前驱节点
    volatile Node prev;

    //后继节点
    volatile Node next;

    //节点对应的线程
    volatile Thread thread;

    //下一个等待者
    Node nextWaiter;

    //判断节点是否在共享模式下等待
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    //获取前驱节点，如果为null，则抛异常
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
	
    //无参构造函数
    Node() {    // Used to establish initial head or SHARED marker
    }

    //有参构造函数1
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    //有参构造函数2
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

**Node节点的状态有如下四种**

* **CANCELLED = 1**

  表示当前节点从同步队列中取消，即当前线程被取消

* **SIGNAL = -1**

  表示后继节点的线程处于等待状态，如果当前节点释放同步状态会通知后继节点，使得后继节点的线程能够运行

* **CONDITION = -2**

  表示当前节点在等待condition，也就是在condition queue中

* **PROPAGATE = -3**

  表示下一次共享式同步状态获取将会无条件传播下去

# **三、AQS类图**

![](http://img.xianzilei.cn/AbstractQueuedSynchronizer%E7%B1%BB%E5%9B%BE.png)

**AbstractQueuedSynchronizer类继承AbstractOwnableSynchronizer抽象类，实现Serializable接口**。其中AbstractOwnableSynchronizer抽象类代码如下

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    //序列化版本号
    private static final long serialVersionUID = 3737899427754241961L;

    //无参构造方法
    protected AbstractOwnableSynchronizer() { }

    //独占模式下的线程
    private transient Thread exclusiveOwnerThread;

    //设置独占线程，final修饰，不可被重写
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    //获取独占线程，final修饰，不可被重写
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

**AbstractQueuedSynchronizer类中有两个内部类，分别为Node和ConditionObject**。`Node`类上面已经介绍过了，`ConditionObject`类实现`Condition`接口，用于实现`Condition`相关功能，这里先不赘述，后续会专门出一篇`Condition`的讲解。

# **四、AQS属性及构造方法**

## 1. 属性

```java
//序列化版本号
private static final long serialVersionUID = 7373984972572414691L;
//头结点
private transient volatile Node head;
//尾结点
private transient volatile Node tail;
//状态
private volatile int state;
//自旋时间
static final long spinForTimeoutThreshold = 1000L;

//Unsafe实例
private static final Unsafe unsafe = Unsafe.getUnsafe();
//state的内存偏移地址
private static final long stateOffset;
//head的内存偏移地址
private static final long headOffset;
//tail的内存偏移地址
private static final long tailOffset;
//waitStatus的内存偏移地址
private static final long waitStatusOffset;
//next的内存偏移地址
private static final long nextOffset;

//调用Unsafe类的objectFieldOffset方法获取各个属性的内存偏移地址，这些偏移地址可以直接操作内存获取和修改值
static {
    try {
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        headOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
        tailOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        waitStatusOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("waitStatus"));
        nextOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("next"));

    } catch (Exception ex) { throw new Error(ex); }
}
```

## 2. 构造方法

```java
protected AbstractQueuedSynchronizer() { }
```

**protected修饰，供子类调用**。

# **五、核心方法详解**

AQS中方法众多，总体可分为两大类，**独占和共享式地获取和释放锁**。下面从这个两方面进行详细讲解。

## **1. 独占式**

**同一时刻仅有一个线程持有同步状态**

### **1.1 独占式同步状态获取（acquire方法）**

AQS提供的acquire方法是**独占式获取同步状态**，但是该方法**对中断不敏感**，也就是说由于线程获取同步状态失败加入到CLH同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移除。代码如下

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

该方法中一共出现了**四个方法**

* **tryAcquire**：**尝试获取锁**，获取成功则设置锁状态并返回true，否则返回false。**该方法由自定义同步组件自己去实现**，该方法必须要**保证线程安全**的获取同步状态。
* **addWaiter**：入队，即将当前线程加入到CLH同步队列尾部。
* **acquireQueued**：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过。
* **selfInterrupt**：自我中断

我们来分析一下acquire方法的执行的流程

* 1）首先调用方法tryAcquire方法（该方法需要自定义同步组件自己实现，这里先不赘述，后续讲解各个组件时再详细描述）**尝试获取锁，如果成功返回true，整个acquire方法全部返回，获取锁成功，否则获取锁失败，进行往下执行**；

* 2）tryAcquire返回false，获取锁失败，则继续执行方法acquireQueued，在此之前会先执行方法addWaiter，**将当前线程加入到CLH同步队列尾部**；

  ```java
  private Node addWaiter(Node mode) {
      //new一个代表当前线程的新节点，默认为独占模式
      Node node = new Node(Thread.currentThread(), mode);
      // Try the fast path of enq; backup to full enq on failure
      //保存尾结点
      Node pred = tail;
      //如果尾结点不为空，说明队列已经被初始化了，直接入队尾
      if (pred != null) {
          //将新节点的前驱连接到原尾结点上
          node.prev = pred;
          //修改尾结点为当前新节点
          if (compareAndSetTail(pred, node)) {
              //原尾结点的后继指向新节点
              pred.next = node;
              //返回新节点
              return node;
          }
      }
      //如果尾结点为空(表示队列未初始化)或者CAS操作失败(有其他线程抢先修改了尾结点信息)，则会调用该方法进行初始化队列后自旋，直到入队成功，或者直接自旋入队操作
      enq(node);
      return node;
  }
  
  //CAS方式修改尾结点
  private final boolean compareAndSetTail(Node expect, Node update) {
      return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
  }
  
  //自旋操作，确保节点成功入队
  private Node enq(final Node node) {
      //无限循环（也叫自旋），目的是确保节点能够成功入队
      for (;;) {
          //保存尾结点，这里每次循环都会重新取一次最新的尾结点
          Node t = tail;
          //如果尾结点为空，则需要先初始化
          if (t == null) { // Must initialize
              //new一个空Node作为头结点和尾结点
              if (compareAndSetHead(new Node()))
                  tail = head;
          } 
          //如果尾结点不为空，说明队列已经初始化了，再次尝试入队，入队成功直接返回原尾结点，否则继续下一次循环
          else {
              node.prev = t;
              if (compareAndSetTail(t, node)) {
                  t.next = node;
                  return t;
              }
          }
      }
  }
  ```

* 3）入队操作结束后执行方法acquireQueued，该方法是一个**自旋**的过程，即**每个线程进入同步队列中，都会自旋地观察自己是否满足条件且获取到同步状态，则就可以从自旋过程中退出，否则继续自旋下去**

  ```java
  final boolean acquireQueued(final Node node, int arg) {
      //表示执行过程中是否发生异常，初始值为true
      boolean failed = true;
      try {
          //表示当前线程是否被中断过，初始值为false
          boolean interrupted = false;
          //自旋过程
          for (;;) {
              //获取当前节点的前驱节点
              final Node p = node.predecessor();
              //如果前驱节点是头结点，表示可以进行获取锁，此时就会调用方法tryAcquire，否则无法参与获取锁，获取锁成功后进入if内
              if (p == head && tryAcquire(arg)) {
                  //此时当前节点获取到了锁，设置头结点为当前节点，同时该节点的Thread的引用设置为空
                  setHead(node);
                  //清除p的引用
                  p.next = null; // help GC
                  //执行过程未发生异常，failed标记为false
                  failed = false;
                  //返回中断标志
                  return interrupted;
              }
              //当获取同步状态失败后，判断是否需要park
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  interrupted = true;
          }
      } finally {
          //如果执行过程发生任何异常则执行
          if (failed)
              cancelAcquire(node);
      }
  }
  
  //设置node为头节点
  private void setHead(Node node) {
      //头结点引用设置为node
      head = node;
      //node的thread引用置为空
      node.thread = null;
      ////node的prev引用置为空
      node.prev = null;
  }
  ```

  自旋获取同步状态，如果获取同步状态失败，会执行方法shouldParkAfterFailedAcquire判断是否需要进行park，代码如下

  ```java
  //判断是否需要对当前线程进行park
  //pred：当前线程节点的前驱节点
  //node：当前线程节点
  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
      //获取当前线程节点的前驱节点的状态
      int ws = pred.waitStatus;
      //如果状态是SIGNAL，表示node可以被安全park
      if (ws == Node.SIGNAL)
          //返回true，即node可以被安全park
          return true;
      //如果状态大于0，即为CANCELLED，表示pred节点的线程被取消，则需要找到从后往前找到第一个态不为CANCELLED的结点作为node的前驱
      if (ws > 0) {
          //从后往前找到第一个态不为CANCELLED的结点
          do {
              node.prev = pred = pred.prev;
          } while (pred.waitStatus > 0);
          pred.next = node;
      } 
      // 表示节点状态为为PROPAGATE（-3）或者是0（表示无状态）或者是CONDITION（-2，表示此节点在condition queue中）
      else {
          //比较并设置前驱结点的状态为SIGNAL
          compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
      }
      return false;
  }
  ```

  如果`shouldParkAfterFailedAcquire`返回`false`，则继续下次自旋，否则执行方法`parkAndCheckInterrupt`来**park当前线程**，代码如下

  ```java
  // 进行park操作并且返回该线程是否被中断
  private final boolean parkAndCheckInterrupt() {
      //阻塞当前线程
      LockSupport.park(this);
      //被唤醒后返回当前线程是否被中断过，会清除中断标记
      return Thread.interrupted();
  }
  ```

  **返回结果表示当前线程是否被中断过**。最后我们来看一下`finally`块中的`cancelAcquire`方法（**只有在自旋过程中发生异常时才执行，因为此时failed为true**），该方法的作用就是**取消当前线程对资源的获取，即设置该结点的状态为CANCELLED**，代码如下

  ```java
  private void cancelAcquire(Node node) {
      // Ignore if node doesn't exist
      //如果节点为空直接返回
      if (node == null)
          return;
  	
      //将当前节点的thread引用置空
      node.thread = null;
  
      // Skip cancelled predecessors
      //保存当前节点的前驱
      Node pred = node.prev;
      //找到node前驱结点中第一个状态不为CANCELLED状态的结点
      while (pred.waitStatus > 0)
          node.prev = pred = pred.prev;
  
      // predNext is the apparent node to unsplice. CASes below will
      // fail if not, in which case, we lost race vs another cancel
      // or signal, so no further action is necessary.
      //获取pred的后继节点
      Node predNext = pred.next;
  
      // Can use unconditional write instead of CAS here.
      // After this atomic step, other Nodes can skip past us.
      // Before, we are free of interference from other threads.
      //设置当前节点node的状态为CANCELLED
      node.waitStatus = Node.CANCELLED;
  
      // If we are the tail, remove ourselves.
      //如果当前节点是尾结点，设置尾结点为前驱节点
      if (node == tail && compareAndSetTail(node, pred)) {
          //比较并设置pred结点的next节点为null
          compareAndSetNext(pred, predNext, null);
      } 
      //如果node结点不为尾结点，或者比较设置不成功
      else {
          // If successor needs signal, try to set pred's next-link
          // so it will get one. Otherwise wake it up to propagate.
          int ws;
          //两种情况满足其一，则if成立
          //情况1：pred结点不为头结点，并且pred结点的状态为SIGNAL 
          //情况2：pred结点状态小于等于0，并且比较并设置等待状态为SIGNAL成功，并且pred结点所封装的线程不为空
          if (pred != head &&
              ((ws = pred.waitStatus) == Node.SIGNAL ||
               (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
              pred.thread != null) {
              //保存node的后继
              Node next = node.next;
              //如果后继不为空并且后继的状态小于等于0
              if (next != null && next.waitStatus <= 0)
                  //比较并设置pred.next = next
                  compareAndSetNext(pred, predNext, next);
          } 
          //如果不满足上面任何一种情况，则释放node的前一个结点
          else {
              //释放node的后继结点
              unparkSuccessor(node);
          }
  
          node.next = node; // help GC
      }
  }
  ```

  **unparkSuccessor方法的作用就是为了释放node节点的后继结点**，细节在下面的释放锁时会细讲。

* 4）最后如果acquireQueued方法返回false，acquire方法直接结束，否则返回true表示当前线程被中断过，需要**恢复它的中断标记**，所以**调用方法selfInterrupt进行自我中断**。

  ```java
  static void selfInterrupt() {
      //中断当前线程
      Thread.currentThread().interrupt();
  }
  ```

**整个acquire方法的主要流程如下图所示**

![](http://img.xianzilei.cn/acquire%E6%96%B9%E6%B3%95%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)

### **1.2 独占式同步状态释放（release方法）**

当线程获取同步状态后，执行完相应逻辑后就需要释放同步状态。**释放锁的逻辑相对简单一些**，代码如下

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

该方法中包含两个方法

* **tryRelease**：**尝试释放锁，成功则返回true，失败则返回false**。同样也是由自定义组件自己实现
* **unparkSuccessor**：**释放当前节点的后继结点**。

分析一下release方法的执行的流程，大致分为以下两步

* **1）调用方法tryRelease，尝试去释放锁，成功则继续执行，否则返回false**

* **2）释放锁成功后，如果存在后继节点，则需要调用方法unparkSuccessor唤醒后继节点的线程**，代码如下

  ```java
  private void unparkSuccessor(Node node) {
  
      //当前节点的状态
      int ws = node.waitStatus;
      //如果状态小于0,设置状态为0
      if (ws < 0)
          compareAndSetWaitStatus(node, ws, 0);
  
      //获取当前节点的后继
      Node s = node.next;
      //如果下一个结点为空或者下一个节点的等待状态为CANCELLED，需要继续往后找到第一个状态小于等于0的节点
      if (s == null || s.waitStatus > 0) {
          s = null;
          //从尾结点开始往前遍历，找到第一个在node节点后的状态小于等于0的节点，赋值给s
          for (Node t = tail; t != null && t != node; t = t.prev)
              if (t.waitStatus <= 0)
                  s = t;
      }
      //如果s不为空，唤醒s对应的线程
      if (s != null)
          LockSupport.unpark(s.thread);
  }
  ```

**整个release方法的主要流程如下图所示**

![](http://img.xianzilei.cn/release%E6%96%B9%E6%B3%95%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### **1.3 独占式获取响应中断（acquireInterruptibly方法）**

前面说到的acquire方法对中断不响应。因此**为了响应中断**，AQS提供了`acquireInterruptibly`方法，该方法**在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断抛出异常InterruptedException**。代码如下

```java
public final void acquireInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        //如果获取锁失败，入队同时响应中断
        doAcquireInterruptibly(arg);
}
```

由代码可以看出，首先会校验该线程是否已经被中断了，如果是则抛出`InterruptedException`，否则执行tryAcquire方法（该方法同样由自定义同步器自己实现）获取同步状态，**如果获取成功，则直接返回，否则执行doAcquireInterruptibly方法**，代码如下

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    //节点入队
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //如果发现线程被中断了，直接抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

该方法与acquireQueued方法非常类似，**唯一不同之处在于如果发现线程被中断了，直接抛异常出去，目的为了响应中断异常**。

### **1.4 独占式超时获取（tryAcquireNanos方法）**

AQS还提供了一个增强版的获取锁的方法，**tryAcquireNanos方法不仅可以响应中断，还有超时控制，即当前线程没有在指定时间内获取同步状态则返回失败**。

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

方法实现与`acquireInterruptibly`类似，不过**当获取锁失败时，会调用方法doAcquireNanos**，代码如下

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    //计算超时时间，即最迟获取锁的时间
    final long deadline = System.nanoTime() + nanosTimeout;
    //入队
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        //自旋获取锁
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //获取同步状态失败，进行中断和超时判断
            //重新计算需要休眠的时间
            nanosTimeout = deadline - System.nanoTime();
            //如果deadline小于等于当前时间，说明超时了，直接返回false
            if (nanosTimeout <= 0L)
                return false;
            //如果未超时，则进行park操作
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            //中断判断
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这里的超时控制需要说明一下，在自旋获取锁之前，先记录一下超时时间dealine，紧接着获取锁，如果获取成功直接返回true，否则会计算`deadline`与当前时间的差值`nanosTimeout` ，如果小于等于0，则表示**已经到了超时时间，直接返回fasle**，否则进行`park`判断，此处除了之前的`shouldParkAfterFailedAcquire`方法的判断，还有**nanosTimeout和spinForTimeoutThreshold常量的比较**，如果大于常量`spinForTimeoutThreshold`，则会进行`park`相应的`nanosTimeout` 时间，**如果小于等于常量spinForTimeoutThreshold，说明nanosTimeout 值很小**（因为常量`spinForTimeoutThreshold`本身的值就很小），在非常短的时间等待无法做到十分精确，如果这时再次进行超时等待，相反会让`nanosTimeout` 的超时从整体上面表现得不是那么精确，所以在超时非常短的场景中，`AQS`会进行**无条件的快速自旋**。

## **2. 共享式**

**同一时刻可以有多个线程获取同步状态**

### **2.1 共享式同步状态获取（acquireShared方法）**

**AQS提供acquireShared方法共享式获取同步状态**

```java
public final void acquireShared(int arg) {
    //尝试获取锁，小于0表示获取锁失败
    if (tryAcquireShared(arg) < 0)
        //自旋获取同步状态
        doAcquireShared(arg);
}
```

**首先调用tryAcquireShared（自定义同步组件自己实现）方法尝试获取同步状态，如果获取成功直接结束，否则调用doAcquireShared方法获取同步状态**，代码如下

```java
private void doAcquireShared(int arg) {
	//入队，此时节点为共享式节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前节点的前驱节点
            final Node p = node.predecessor();
            //如果p是头节点
            if (p == head) {
                //尝试获取锁
                int r = tryAcquireShared(arg);
                //如果r大于等于0，表示获取锁成功
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

代码与之前讲解的非常相似，这里就不再赘述了。

### **2.2 共享式同步状态释放（releaseShared方法）**

**AQS提供releaseShared方法释放共享式同步状态**

```java
public final boolean releaseShared(int arg) {
    //尝试释放锁
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

**调用方法tryReleaseShared尝试释放锁，如果失败直接返回false，否则调用方法doReleaseShared**，代码如下

```java
private void doReleaseShared() {
    /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
    for (;;) {
        Node h = head;
        //如果队列存在排队的节点
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                //CAS设置不成功则不断循环
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //CAS操作成功后释放后继节点
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //队列不存在排队的节点，直接结束自旋
        if (h == head)                   // loop if head changed
            break;
    }
}
```

### **2.3 共享式获取响应中断（acquireSharedInterruptibly方法）**

```java
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

**与独占式类似，不过调用的方法是doAcquireSharedInterruptibly**

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**添加中断的相应，直接抛出异常**。代码类似，这里不再赘述。

### **2.4 共享式超时获取（tryAcquireSharedNanos方法）**

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
```

**调用方法doAcquireSharedNanos**

```java
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

代码类似，不再赘述。

# 总结

**关于AQS的知识点还有很多，一篇博客无法穷尽其知识点，后续的博文会不断完善相应的知识点。博主能力有限，如有不正确的地方请及时与我联系**。