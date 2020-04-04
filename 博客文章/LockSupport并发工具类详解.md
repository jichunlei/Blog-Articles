# **一、简介**

我们知道，`AbstractQueuedSynchronizer`（AQS）的核心包括**同步队列（CLH）**、**CAS操作**和**LockSupport**。其中**AQS利用LockSupport来控制线程的状态，利用park和unpark实现线程的等待与唤醒，从而能够达到管理线程的目的**。下面将为大家详细介绍`LockSupport`类。

# **二、属性及构造方法**

## **1. 属性**

```java
public class LockSupport {
    // Unsafe实例
    private static final sun.misc.Unsafe UNSAFE;
    //以下四个变量表示都是各自的内存偏移量，会在static静态代码块中进行初始化
    private static final long parkBlockerOffset;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    static {
        try {
            //获取Unsafe的实例
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            //线程类类型
            Class<?> tk = Thread.class;
            //获取Thread的parkBlocker字段的内存偏移地址
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
            //获取Thread的threadLocalRandomSeed字段的内存偏移地址
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            //获取Thread的threadLocalRandomProbe字段的内存偏移地址
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            //获取Thread的threadLocalRandomSecondarySeed字段的内存偏移地址
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception ex) { throw new Error(ex); }
    }

}
```

## **2. 构造方法**

```java
private LockSupport() {} // Cannot be instantiated.
```

**构造函数被私有化，即无法被实例化。**

# **三、核心方法分析**

## **1. park方法**

**park方法有两个重载版本**

* **park()**

  **阻塞线程当前线程**

  ```java
  public static void park() {
  	//0L表示一直阻塞，直到被唤醒
      UNSAFE.park(false, 0L);
  }
  ```

  该方法直接调用Unsafe方法的park方法（不了解Unsafe类的可以查看这篇博客：[聊一聊Java中的Unsafe类](http://xianzilei.cn/blog/63)）

* **park(Object blocker)**

  **阻塞线程当前线程**，入参增加一个Object对象，用来**记录导致线程阻塞的阻塞对象**，方便进行**问题排查**。

  ```java
  public static void park(Object blocker) {
      // 获取当前线程
      Thread t = Thread.currentThread();
      // 设置Blocker
      setBlocker(t, blocker);
      // 阻塞程序
      UNSAFE.park(false, 0L);
      // 被唤醒后重新可运行后设置Blocker为空
      setBlocker(t, null);
  }
  
  //设置线程t的parkBlocker字段的值为arg
  private static void setBlocker(Thread t, Object arg) {
      // Even though volatile, hotspot doesn't need a write barrier here.
      //调用Unsafe类的putObject方法直接修改内存信息
      UNSAFE.putObject(t, parkBlockerOffset, arg);
  }
  ```

## **2. parkNanos方法**

**parkNanos方法也有两个重载版本**

* **parkNanos(long nanos)**

  **阻塞当前线程，最长不超过nanos纳秒**，相对于park方法增加了**超时返回**的特性。

  ```java
  public static void parkNanos(long nanos) {
      if (nanos > 0)
          //调用Unsafe的park方法
          UNSAFE.park(false, nanos);
  }
  ```

* **parkNanos(Object blocker, long nanos)**

  **功能同上**，入参增加一个Object对象，用来**记录导致线程阻塞的阻塞对象**，方便进行**问题排查**。

  ```java
  public static void parkNanos(Object blocker, long nanos) {
      if (nanos > 0) {
          Thread t = Thread.currentThread();
          setBlocker(t, blocker);
          UNSAFE.park(false, nanos);
          setBlocker(t, null);
      }
  }
  ```

## **3. parkUntil方法**

**parkUntil方法同样也有两个重载版本**

* **parkUntil(long deadline)**

  **阻塞当前线程，直到绝对时间为deadline**。

  ```java
  public static void parkUntil(long deadline) {
      //第一个参数为true表示第二个参数为绝对时间
      UNSAFE.park(true, deadline);
  }
  ```

* **parkUntil(Object blocker, long deadline)**

  **功能同上**，入参增加一个Object对象，用来**记录导致线程阻塞的阻塞对象**，方便进行**问题排查**。

  ```java
  public static void parkUntil(Object blocker, long deadline) {
      Thread t = Thread.currentThread();
      setBlocker(t, blocker);
      UNSAFE.park(true, deadline);
      setBlocker(t, null);
  }
  ```

## **4. unpark方法**

**唤醒处于阻塞状态的指定线程**

```java
public static void unpark(Thread thread) {
    if (thread != null)
        //调用Unsafe的unpark方法
        UNSAFE.unpark(thread);
}
```

# **四、park与unpark实现原理**

LockSupport在执行park和unpark时，查看的是一个许可，**如果存在许可，线程在调用park的时候，会立马返回，此时许可也会被消费掉，如果没有许可，则会阻塞。调用unpark的时候，如果许可本身不可用，则会使得许可可用**。注：**许可只有一个，不可累加**。

上面我们介绍到park和unpark的实现是通过**调用Unsafe的park和unpark方法**。

```java
//Unsafe类中
//取消阻塞线程，即唤醒指定线程
public native void unpark(Object thread);

//阻塞当前线程，isAbsolute表示是否是绝对时间（true表示会实现ms定时；false则会实现ns定时），time表示可以设置超时自动唤醒时间（0表示一直阻塞，等待唤醒）
public native void park(boolean isAbsolute, long time);
```

可以看出该方法是native方法，底层是C++实现的，这里就不深入扒C++代码了，直接上结论吧。

LockSupport通过**控制变量_counter来对线程阻塞唤醒进行控制**的，原理有点类似于**信号量**机制。

* **当调用park()方法时，会将_counter置为0，同时判断前值，小于1说明前面被unpark过，则直接退出，否则将使该线程阻塞。**
* **当调用unpark()方法时，会将_counter置为1，同时判断前值，小于1会进行线程唤醒，否则直接退出**。

总结一下，线程阻塞需要消耗凭证(permit)，这个凭证最多只有1个。当调用park方法时，如果有凭证，则会直接消耗掉这个凭证然后正常退出；但是如果没有凭证，就必须阻塞等待凭证可用；而unpark则相反，它会增加一个凭证，但凭证最多只能有1个。例如**我们先调用unpark在调用park方法并不会导致线程阻塞，因为调用unpark获得了一个凭证，之后调用park因为有凭证消费，故不会阻塞**。