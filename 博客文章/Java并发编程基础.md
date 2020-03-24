# **一、进程和线程**

* **进程**

  > 进程（Process） 是计算机中的程序关于某数据集合上的一次运行活动，是**系统进行资源分配和调度的基本单位**，是操作系统结构的基础。 在当代面向线程设计的计算机结构中，进程是线程的容器。程序是指令、数据及其组织形式的描述，进程是程序的实体。是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。程序是指令、数据及其组织形式的描述，进程是程序的实体。

* **线程**

  > 线程（thread） 是**操作系统能够进行运算调度的最小单位**。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

总结起来就是**进程是系统中正在运行的一个应用程序**；**线程则是进程内独立执行的一个单元执行流**。**进程是资源分配的最小单元**；**线程是程序执行的最小单位**。

# **二、并发与并行**

* **并行**

  > 并行(parallel)：指在**同一时刻**，有**多条指令**在多个处理器上**同时执行**。所以无论从**微观还是从宏观来看，二者都是一起执行的**。

* **并发**

  > 并发(concurrency)：指在**同一时刻**只能有**一条指令**执行，但**多个**进程指令被快速的**轮换执行**，使得在**宏观上具有多个进程同时执行的效果，但在微观上并不是同时执行的**，只是把时间分成若干段，使多个进程快速交替的执行。
  >

# **三、Java线程的创建**

**创建Java线程主要有以下三种方式**

## **1. 继承Thread类**

**第一种方式是直接继承Thread类，并重写它的run方法**。这种方式不好的地方是 **Java 不支持多继承**，如果该类继承了Thread类就不能继承其他类了。

```java
public class MyThread extends Thread{

    @Override
    public void run() {
        System.out.println(this.getName()+" do something...");
    }


    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.setName("xianzilei");
        myThread.start();
    }
}
```

**控制台打印**

```tex
xianzilei do something...

Process finished with exit code 0
```

使用继承方式时，`run`方法内获取当前线程直接使用`this`就可以，无须使用` Thread.currentThread()` 方法。

## **2. 实现Runnable接口**

第二种方式是实现Runnable接口，将其作为Thread构造方法的参数，从来创建线程。

我们知道**Thread有如下构造方法**

```java
//参数：Runnable的实现类
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
//参数：Runnable的实现类和自定义的线程名称
public Thread(Runnable target, String name) {
    init(null, target, name, 0);
}
```

下面采用**匿名内部类**方式

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+" do something...");
    }
},"xianzilei").start();
```

如果JDK版本在1.8及以上，上面的代码可以使用**lambda表达式**

```java
new Thread(() -> System.out.println(Thread.currentThread().getName()+" do something..."),"xianzilei").start();
```

从上面的代码可以看到，实现Runnable接口的类的实例对象仅仅作为`Thread`对象的`target`，`Runnable`实现类里包含的`run()`方法仅仅作为**线程执行体**，而**实际的线程对象依然是Thread实例**，这里的`Thread`实例负责执行其`target`的`run()`方法。因此要获取当前线程对象只能通过`Thread.currentThread()`方法，而不能通过`this`关键字获取。使用Runnable让任务类脱离了Thread继承体系，更加灵活。 

## **3. 使用Callable和Future**

上面两个创建线程的run方法均没有返回值，而使用Callable可以获取异步执行的返回值

首先我们来看一下Callable接口的定义

```java
@FunctionalInterface
public interface Callable<V> {
    //类似于run方法，但是call方法可以获取返回值和抛出异常
    V call() throws Exception;
}
```

因此我们可以创建`Callable`接口的实现类，但是`Thread`类的构造函数没有`Callable`类型的参数，无法直接将`Callable`实现类作为参数。因此Java提供了`Future`接口来代表`Callable`接口里`call()`方法的返回值，并为`Future`接口提供了一个`FutureTask`实现类，该类实现了`Future`接口，并实现了`Runnable`接口，所以`FutureTask`可以作为`Thread`类的`target`，同时也解决了`Callable`对象不能作为`Thread`类的`target`这一问题。

```java
public class FutureTask<V> implements RunnableFuture<V> {
}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

**上述创建线程的方式的步骤如下**

* 1）创建Callable接口实现类，并实现call()方法，该方法将作为**线程执行体**，且该方法有返回值，再创建Callable实现类的实例；
* 2）使用`FutureTask`类来包装`Callable`对象，该`FutureTask`对象封装了该`Callable`对象的`call()`方法的返回值；
* 3）使用`FutureTask`对象作为`Thread`对象的`target`创建并启动新线程；
* 4）调用`FutureTask`对象的`get()`方法来获得子线程执行结束后的返回值。

```java
public static void main(String[] args) {
    //创建FutureTask实例，其中callable作为参数
    FutureTask<String> futureTask = new FutureTask<>(() -> "success");
    //FutureTask实例作为Thread的target，并启动线程
    new Thread(futureTask).start();
    try {
        //等待任务执行完毕，并返回结果
        String result = futureTask.get();
        System.out.println("执行结果:"+result);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

**控制台打印**

```tex
执行结果:success

Process finished with exit code 0
```

## 4. Runnable和Callable的区别

* **1）Callable规定（重写）的方法是call()，Runnable规定（重写）的方法是run()。**
* **2）Callable的任务执行后可返回值，而Runnable的任务没有返回值。**
* **3）call方法可以抛出异常，run方法不可以。**
* **4）运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果**。

# 四、Java线程常用方法

**主要方法如下**

| 方法名                           | static | 功能说明                                                     |
| -------------------------------- | ------ | ------------------------------------------------------------ |
| **start()**                      |        | **启动一个新线程，使线程进入就绪状态，等待CPU的调度**。每个线程的start方法只能调用一次，如果多次调用会抛出`IllegalThreadStateException`异常 |
| **run()**                        |        | 线程启动后获取CPU的执行时调用的方法，即**线程执行体**。      |
| **join()**                       |        | `t.join()`方法会使主线程进入等待池并等待t线程执行完毕后才会被唤醒 |
| **join(long millis)**            |        | 主线程等待t线程超过指定时间，则结束等待。                    |
| **getId()**                      |        | **获取线程长整型的id**，id唯一                               |
| **getName()**                    |        | **获取线程名**                                               |
| **setName(String name)**         |        | **修改线程名**                                               |
| **getPriority()**                |        | **获取线程的优先级**。Java中规定线程的优先级是1~10的整数，数字越大优先级越高 |
| **setPriority(int newPriority)** |        | **修改线程的优先级**                                         |
| **getState()**                   |        | **获取线程状态**。Java中的线程状态使用6个枚举值：`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`,`TIMED_WAITING`, `TERMINATED` |
| **isInterrupted()**              |        | **判断是否被打断**，不会清楚打断标记                         |
| **isAlive()**                    |        | **线程是否存活**                                             |
| **interrupt()**                  |        | **打断线程**。如果被打断线程正在 `sleep`，`wait`，`join` 会导致被打断的线程抛出 `InterruptedException`异常，并清除打断标记 ；如果打断的正在运行的线程，则会设置 打断标记 ；park 的线程被打断，也会设置打断标记 |
| **interrupted()**                | static | **判断当前线程是否被打断**，会清除打断标记                   |
| **currentThread()**              | static | **获取当前正在执行的线程**                                   |
| **sleep(long millis)**           | static | **让当前执行的线程休眠指定时间**，休眠期间会让出CPU的时间片给其他线程，但是不释放对象锁。其它线程可以使用 `interrupt` 方法打断正在睡眠的线程，会抛出 `InterruptedException`异常 |
| **yield()**                      | static | **线程让步**。让当前线程由“运行状态”进入到“就绪状态”，从而让其它具有相同优先级的等待线程获取执行权；但是，并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权；也有可能是当前线程又进入到“运行状态”继续运行。 |
| **stop()**                       |        | **停止线程运行**                                             |
| **suspend()**                    |        | **挂起（暂停）线程运行**                                     |
| **resume()**                     |        | **恢复线程运行**                                             |



# **五、Java线程的状态**

Java线程在运行的生命周期中可能会有6种不同的状态，在给定的一个时刻，线程只能处于其中的一个状态。

| 状态名称         | 状态说明                                                     |
| ---------------- | ------------------------------------------------------------ |
| **NEW**          | **初始状态**，线程刚被创建，但是还没有调用其`start()`方法    |
| **RUNNABLE**     | **运行状态**，Java线程将操作系统中的就绪和运行两种状态统称为运行中 |
| **BLOCKED**      | **阻塞状态**，表示线程阻塞于锁                               |
| **WAITING**      | **等待状态**，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定的动作（通知或中断） |
| **TIME_WAITING** | **超时等待状态**，该状态不同于WAITING，它是可以在指定的时间自行返回 |
| **TERMINATED**   | **终止状态**，表示当前线程已经执行完毕                       |

**Java线程状态间的变迁**如下图所示（借鉴《Java并发编程的艺术》中的插图）

![](http://img.xianzilei.cn/Java%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E5%8F%98%E8%BF%81.png)

# **六、线程间通信**

**线程通信的目标是使线程间能够互相发送信号，从而相互协调完成任务**。Java中线程通信的方式有很多，这里介绍几种常见的通信方式

## **1. 共享变量**

最简单的方式就是**在线程的共享对象的变量中设置信号值**，也是最容易想到的。

```java
public class MySharedSignal {
	//共享信号量
    protected boolean sign=false;

    //获取信号量
    public synchronized boolean getSign() {
        return sign;
    }

    //修改信号量
    public synchronized void setSign(boolean sign) {
        this.sign = sign;
    }
}
```

该方式需要保证多个线程获得同一个`MySharedSignal`实例。

## **2. 忙等待**

线程A不断处理数据并改变信号值，线程B不停地通过while语句检测信号值，从而实现线程间的通信。

```java
//线程A
//Thread A do something...
sign=false;

//线程B
while (!sign){
    //do noting,just waiting
}
// Thread B do something...
```

这种方式线程B一直while循环，会**极大地浪费CPU资源**。同时也会存在**轮询条件的可见性**问题。

## **3. 等待/通知机制**

**wait/notify机制**是最常见的线程通信的方法，它使用`Java`中`Object`类的`wait()`、`notify()`或`notifyAll()`等方法，同时配合使用synchronized关键字来实现等待/通知机制。

### **3.1 相关方法**

这些方法都是Object类中的方法

* **notify()**

  **任意通知一个在该对象上等待的线程，使其从wait()方法返回**，而返回的前提是该线程获取到了对象的锁。

* **notifyAll()**

  相对于`notify()`，它是通知所有等待在该对象上的线程。

* **wait()**

  **调用该方法的线程进入WAITING状态，同时释放该对象的锁，进入该对象的等待队列中**。只有等到另外线程的通知或被中断才能返回。

* **wait(long timeout)**

  相对于wait()，这里可以指定等到的最大时长，如果没有通知则超时就返回。单位毫秒。

* **wait(long timeout, int nanos)**

  更细粒度的超时时间控制，可以达到纳秒级别

### **3.2 使用原理**

等待/通知机制，是指一个线程A调用了对象O的`wait()`方法进入等待状态，而另一个线程B
调用了对象O的`notify()`或者`notifyAll()`方法，线程A收到通知后从对象O的`wait()`方法返回，进而
执行后续操作。上述两个线程**通过对象O来完成交互**，而对象上的`wait()`和`notify/notifyAll()`的
关系就如同**开关信号**一样，用来**完成等待方和通知方之间的交互工作**。

### **3.3 案例**

**代码如下**

```java
public static void main(String[] args) throws Exception {
    //对象锁
    Object lock=new Object();
    //创建线程A
    Thread threadA = new Thread(()->{
        System.out.println("A尝试获取锁...");
        synchronized (lock){
            System.out.println("A获取锁成功！");
            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println("A开始释放锁，并开始等待...");
                lock.wait();
                System.out.println("A被通知继续运行，直接结束。");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    });
    //创建线程B
    Thread threadB = new Thread(()->{
        System.out.println("B尝试获取锁...");
        synchronized (lock){
            System.out.println("B获取锁成功！");
            try {
                TimeUnit.SECONDS.sleep(3);
                System.out.println("B开始释放锁...");
                lock.notify();
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
A被通知继续运行，直接结束。

Process finished with exit code 0
```

## **4. 管道通信**

管道通信，就是使用管道输入/输出流来进行线程之间的数据传输，传输的媒介为内存。Java中管道输入/输出流主要包括了如下4种具体实现：`PipedOutputStream`、`PipedInputStream`、`PipedReader`和`PipedWriter`，前两种面向字节，而后两种面向字符。接下来借鉴《Thinking in Java》中的案例来说明管道通信

* 定义两个线程，Sender负责写数据，Receiver负责读数据

  ```java
  //管道写线程
  class Sender implements Runnable {
      //随机数类
      private Random random = new Random(47);
      //out往管道写数据
      private PipedWriter out = new PipedWriter();
  
      public PipedWriter getPipedWriter() {
          return out;
      }
  
      //循环输出A到Z，每个字母间隔休眠一段随机时间
      public void run() {
          try {
              while (true) {
                  for (char c = 'A'; c < 'Z'; c++) {
                      out.write(c);
                      //随机休眠500以内毫秒
                      TimeUnit.MILLISECONDS.sleep(random.nextInt(500));
                  }
              }
          } catch (IOException e) {
              System.out.println(e + " Sender write exception");
          } catch (InterruptedException e) {
              System.out.println(e + " Sender sleep interrupt");
          }
      }
  }
  
  //管道读线程
  class Receiver implements Runnable {
      //in往管道读数据
      private PipedReader in;
  
      public Receiver(Sender sender) throws IOException {
          //管道输入流和输出流的绑定
          in = new PipedReader(sender.getPipedWriter());
      }
  
      //循环读取管道数据
      public void run() {
          try {
              while (true) {
                  //read():如果管道没有数据，则会自动阻塞
                  System.out.print("Read: " + (char) in.read() + ",");
              }
          } catch (IOException e) {
              System.out.println(e + " Receiver read exception");
          }
      }
  }
  ```

* **启动线程**

  ```java
  public static void main(String[] args) throws Exception {
      //创建写线程
      Sender sender = new Sender();
      //创建读线程
      Receiver receiver = new Receiver(sender);
      ExecutorService exec = Executors.newCachedThreadPool();
      exec.execute(sender);
      exec.execute(receiver);
      //休眠4秒钟后中断
      TimeUnit.SECONDS.sleep(4);
      exec.shutdownNow();
  }
  ```

* **查看日志**

  ```tex
  Read: A,Read: B,Read: C,Read: D,Read: E,Read: F,Read: G,Read: H,Read: I,Read: J,Read: K,Read: L,Read: M,java.lang.InterruptedException: sleep interrupted Sender sleep interrupt
  java.io.InterruptedIOException Receiver read exception
  
  Process finished with exit code 0
  ```