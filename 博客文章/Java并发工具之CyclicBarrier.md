# **一、简介**

摘自《Java并发编程的艺术》一书中

> `CyclicBarrier`的字面意思是**可循环使用（Cyclic）的屏障（Barrier）**。它要做的事情是，**让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行**。

`CyclicBarrier`，一个同步辅助类，在API中是这么介绍的： **它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)**。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 `CyclicBarrier` 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 `barrier`。 通俗点讲就是：**让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活**。

**举个栗子**，想必很多小伙伴都会玩英雄联盟或者农药手游，大家在选完英雄的时候会需要进行等待加载，等到10位玩家加载准备完成之后才能正式开始游戏。**这个10位玩家可以理解为10个线程，在加载过程中，10个线程互相等待，直到最后一位玩家加载完成，即所有线程都达到某一个屏障，此时被等待的线程才能继续执行**，即大家才能开始happy起来。

# **二、类总览**

![](http://img.xianzilei.cn/CyclicBarrier%E7%B1%BB%E5%9B%BE.png)

## 1. 类的继承关系

`CyclicBarrier`没有显示继承哪个父类或者实现哪个父接口

```java
public class CyclicBarrier {}
```

`CyclicBarrier`中有个内部类Generation，定义如下

```java
private static class Generation {
    //表示当前屏障是否被损坏，默认为false
    boolean broken = false;
}
```

这个内部类从字面意思理解是“**代**”的意思，为什么要有这个内部类呢？我们知道`CyclicBarrier`是**可重复使用的，每次重复使用都会新建一个Generation，它的broken属性默认为false**。举个栗子，很多人都玩过过山车吧。假设一个过山车是20个座位，工作人员一般会等到栏杆外排队够20人才会打开栏杆让这20人通过；然后就会将栏杆重新关闭，后面新来的继续等待。这里的前面已经通过的人就是一“代”，后面再继续等待的20人就是另外一“代”。**每次栏杆打开关闭一次，就会产生新的一“代”**。**在`CyclicBarrier`，开启新的一代使用的是`nextGeneration`方法**，定义如下

```java
private void nextGeneration() {
    // 唤醒当前这一代中所有等待在条件队列里的线程
    trip.signalAll();
    // 重置count值
    count = parties;
    //新建Generation，开启新的一代
    generation = new Generation();
}
```

**该方法用于开启新的一代，通常是被最后一个调用await方法的线程调用**。在该方法中，我们的主要工作就是**唤醒当前这一代中所有等待在条件队列里的线程，将count的值恢复为parties，以及开启新的一代**。

## **2. 核心属性**

```java
//可重入独占锁
private final ReentrantLock lock = new ReentrantLock();
//Condition实例，表示条件队列
private final Condition trip = lock.newCondition();
//参与线程的总数
private final int parties;
//代表了一个任务，表示当barrier开启时就会执行这个对象的run方法
//这个参数不是必须的，如果不需要执行，可以为null
private final Runnable barrierCommand;
//generation实例，表示当前代
private Generation generation = new Generation();
//还需要等待的线程数，初始值为parties
private int count;
```

属性包括`ReentrantLock`和`Condition`，即`CyclicBarrier`是**基于独占锁`ReentrantLock`和条件队列实现的**，**所有相互等待的线程都会在同样的条件队列`trip`上挂起，被唤醒后将会被添加到`sync queue`中去争取独占锁lock，获得锁的线程将继续往下执行**。

## **3. 构造方法**

`CyclicBarrier`有**两个构造函数**

* **指定parties参数**

  ```java
  public CyclicBarrier(int parties) {
      this(parties, null);
  }
  ```

  **调用第二个构造方法，传递`barrierAction`参数为null**

* **指定`parties`和`barrierAction`参数**

  ```java
  public CyclicBarrier(int parties, Runnable barrierAction) {
      //parties即参与的线程数必须大于0
      if (parties <= 0) throw new IllegalArgumentException();
      //初始化parties、count和barrierCommand属性
      this.parties = parties;
      this.count = parties;
      this.barrierCommand = barrierAction;
  }
  ```

# **三、使用案例**

假设有一家公司要全体员工进行团建活动，活动内容为翻越三个障碍物，参与活动的一共有五名员工，要求**所有人在翻越当前障碍物之后再开始翻越下一个障碍物**，代码如下（**混个脸熟，先学会使用，原理后面讲解**）

```java
public static void main(String[] args) {
    //参与的线程数
    int threadNum = 5;
    //创建cyclicBarrier实例，定义barrierAction
    CyclicBarrier cyclicBarrier = new CyclicBarrier(threadNum, () -> System.out.println(
        "所有员工通过当前屏障，继续前进！"));
    //创建线程开始执行
    for (int i = 1; i <= threadNum; i++) {
        new Thread(() -> {
            for (int j = 1; j <= 3; j++) {
                try {
                    Random rand = new Random();
                    //产生1000到3000之间的随机整数，模拟跨越障碍的耗时
                    int randomNum = rand.nextInt((3000 - 1000) + 1) + 1000;
                    Thread.sleep(randomNum);
                    System.out.println(Thread.currentThread().getName() + ", 通过了第" + j +
                                       "个障碍物, " +
                                       "使用了 " + ((double) randomNum / 1000) + "s");
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }, i + "号员工").start();
    }
}
```

**执行结果如下**

```tex
1号员工, 通过了第1个障碍物, 使用了 1.046s
3号员工, 通过了第1个障碍物, 使用了 1.276s
2号员工, 通过了第1个障碍物, 使用了 2.298s
4号员工, 通过了第1个障碍物, 使用了 2.394s
5号员工, 通过了第1个障碍物, 使用了 2.818s
所有员工通过当前屏障，继续前进！
4号员工, 通过了第2个障碍物, 使用了 1.021s
2号员工, 通过了第2个障碍物, 使用了 2.014s
5号员工, 通过了第2个障碍物, 使用了 2.335s
3号员工, 通过了第2个障碍物, 使用了 2.557s
1号员工, 通过了第2个障碍物, 使用了 2.573s
所有员工通过当前屏障，继续前进！
4号员工, 通过了第3个障碍物, 使用了 1.46s
5号员工, 通过了第3个障碍物, 使用了 2.098s
3号员工, 通过了第3个障碍物, 使用了 2.66s
2号员工, 通过了第3个障碍物, 使用了 2.796s
1号员工, 通过了第3个障碍物, 使用了 2.896s
所有员工通过当前屏障，继续前进！
```

这里每个员工相当于每个参与的线程，每个线程执行完当前任务时会调用await方法，**该方法的作用是如果存在没有到达Barrier的线程就会自我阻塞；如果不存在则会唤醒所有阻塞的线程，同时执行`barrierAction`的run方法**。同时我们看到`CyclicBarrier`可以**重复使用**，印证了它的循环屏障的含义。

# **四、核心方法**

根据上面的分析我们知道`CyclicBarrier`的使用**只需要它的一个await方法即可完成它强大的功能**，下面我们来分析一下它的await方法。

## **1. 辅助方法**

在介绍await方法前，我们先来了解一下一些**辅助方法**

### **1.1 breakBarrier方法**

方法定义如下

```java
private void breakBarrier() {
    //设置broken状态为true，即表示屏障被打破
    generation.broken = true;
    //重置count值
    count = parties;
    //唤醒当前这一代中所有等待在条件队列里的线程（因为栅栏已经打破了）
    trip.signalAll();
}
```

继续拿上面的过山车的例子来说明，假如当天是工作日，游客较少，有时候很难凑够20人，为了避免当前等待的游客着急，这个时候工作人员也会打开栏杆（**此时人数不够20人**）让当前等待的游客通过。**这个工作人员的行为就相当于调用方法`breakBarrier`，因为此时不是在凑够20人的情况下打开屏障，所以我们把这一代的broken状态设置为true，表示屏障被打破**。

### **1.2 reset方法**

reset方法用于将barrier恢复成初始的状态，**它的内部就是简单地调用了`breakBarrier`方法和`nextGeneration`方法**

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    //需要先获取锁
    lock.lock();
    try {
        //打破当前屏障
        breakBarrier();
        //开启新的一代
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```

注意：**如果在我们执行该方法时有线程正等待在barrier上，则它将立即返回并抛出`BrokenBarrierException`异常**

## **2. await**

**先看方法定义**

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

`await`方法内部调用`dowait`方法

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
TimeoutException {
    final ReentrantLock lock = this.lock;
    //获取重入锁
    //所有执行await方法的线程必须是已经持有了锁，所以这里必须先获取锁
    lock.lock();
    try {
        final Generation g = generation;
        //前面介绍到调用breakBarrier方法会将当前代的broken属性设置为true，表示当前屏障被打破了
		//如果发现当前的barrier已经被打破了，则直接抛出异常
        if (g.broken)
            throw new BrokenBarrierException();
        
		//如果当前线程被中断了，则需要将屏障打破，再抛出中断异常
        //这里这么做的原因是：由于在barrier上的线程是互相等待的，如果其中一个被中断了，那么其他的就不用再等待了
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
		
        //当前线程到达屏障出，将等待的线程数减一
        int index = --count;
        //如果等待的线程数为0，表示所有的线程都到齐了，则可以唤醒所有等待的线程，同时重置屏障
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                //如果有设置barrierCommand属性，则会调用它的run方法
                if (command != null)
                    command.run();
                ranAction = true;
                //唤醒所有线程，开启新的一代
                nextGeneration();
                return 0;
            } finally {
                //这里是防止barrierCommand的run方法执行出了异常，导致无法唤醒其余等待的线程，这里做一下兜底，直接打破屏障
                if (!ranAction)
                    breakBarrier();
            }
        }

        //能执行到这说明此时等待的线程数还不为0，需要将线程挂起
        for (;;) {
            try {
                //如果没有设置超时机制，直接调用Condition的await方法
                if (!timed)
                    trip.await();
                //否则，则等待指定的时间
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } 
            //如果在等待的过程中线程被中断了，执行下面代码
            catch (InterruptedException ie) {
                //如果线程处于当前这一“代”，并且当前这一代还没有被broken,则先打破栅栏
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    //重新抛出异常
                    throw ie;
                } 
                //否则无需处理，直接恢复中断即可
                // 注意来到这里有两种情况
                // 一种是g!=generation，说明新的一代已经产生了，所以我们没有必要处理这个中断，只要再自我中断一下就好，交给后续的人处理
               // 一种是g.broken = true, 说明中断前栅栏已经被打破了，既然中断发生时栅栏已经被打破了，也没有必要再处理这个中断了
                else {
                    //自我中断
                    Thread.currentThread().interrupt();
                }
            }

            //能够执行到此处说明线程被唤醒了
            //这里检测一下broken状态是否为true，如果是抛出异常
            //能使broken状态变为true的，只有调用breakBarrier()方法
            if (g.broken)
                //BrokenBarrierException异常一般表示某个线程在等待某个处于“打破”状态的barrier
                throw new BrokenBarrierException();

            //如果线程被唤醒时，新一代已经被开启了，说明一切正常，直接返回
            if (g != generation)
                return index;
			//如果是超时等待且已经超时，则打破屏障，抛出超时异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

**正常情况下的await的逻辑很简单，就是线程间互相等待，知道所有线程都到达屏障后，屏障打开，各线程继续执行。但是await方法的难点在于屏障被打破的情况下的处理**。我们知道如果在**参与者（线程）在调用await方法的过程中，barrier被破坏，就会抛出`BrokenBarrierException`异常**。在await方法中，抛出该异常的代码只有一种方式

```java
//该判断存在await方法两处
//1）第一处是刚开始获取锁之后的判断，即await方法的最前端处，也可以理解为到达屏障前的位置
//2）第二次是唤醒之后的判断
if (g.broken)
    throw new BrokenBarrierException();
```

即当`generation`实例中的`broken`标识为`true`时，才会抛出异常。但是`generation`实例中的`broken`标识默认为false，只有当调用`breakBarrier`方法才会修改标识为true，因此得出结论：**当前线程如果刚开始执行await方法或者唤醒之后发现自己等待的屏障已经被打破了，会直接抛出`BrokenBarrierException`异常。**

下面我们来分析一下`breakBarrier`方法调用的几种情况。

* **第一种情况：当前线程达到屏障前发现自己被中断了**

  这种情况下，意味着后续的线程以及等待的线程再也**不可能达到屏障开启的条件**了，所以**当前线程会主动打破屏障，唤醒等待的线程，避免等待的线程一直等待**。

* **第二种情况：最后一个到达的线程在执行`barrierCommand`的run方法时发生了错误**

  该情况下，首先所有的线程肯定都已经就位了，只是在执行用户自定义的屏障处的执行方法时报错，为了避免报错导致所有等待的线程没有人去唤醒，会主动去唤醒。

* **第三种情况：线程在调用`Condition.await`方法的时候发现自己被中断了，会抛出中断异常，此时如果当前代没有更新为下一代，且当前代没有被打破**

  会调用`breakBarrier`方法主动打破屏障

* **第四种情况：reset方法被调用**

  reset方法的作用是重置`CyclicBarrier`，类似**清除历史重新来，这个方法JDK内部不会调用，可能是用户代码调用**。

上面前三张种情况我们在`dowait`方法中标注一下

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        //1）第一种情况：当前线程达到屏障前发现自己被中断了
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
		
        //到达屏障的位置
        int index = --count;
        if (index == 0) {
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    //第二种情况：最后一个到达的线程在执行barrierCommand的run方法时发生了错误
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    //第三种情况：挂起的过程中发生了中断
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

**总结一下：`CyclicBarrier`使用了`“all-or-none breakage model”`，所有互相等待的线程，要么一起通过barrier，要么一个都不要通过，如果有一个线程因为中断，失败或者超时而过早的离开了barrier，则该barrier会被broken掉，所有等待在该barrier上的线程都会抛出`BrokenBarrierException`（或者`InterruptedException`）**

# **五、CyclicBarrier与CountdownLatch的区别**

`CyclicBarrier`的功能与`CountDownLatch`类似，它可以使得一组线程之间相互等待，直到所有的线程都到齐了之后再继续往下执行。但是二者还是存在些许区别。

* `CyclicBarrier`基于**条件队列和独占锁**来实现；而`CountDownLatch`基于**共享锁**实现
* `CyclicBarrier`可以**重复使用**，当所有线程就位完成时，会开启下一代；而`CountDownLatch`是**一次性的，不可重复使用**
* `CyclicBarrier`是**加计数**；`CountDownLatch`是**减计数**
* `CyclicBarrier`中**操作计数和阻塞的是同一个线程**，调用方法只有一个await方法；而`CountDownLatch`中**操作计数和阻塞等待是两个线程**，控制计数调用方法`countDown`，且不会被阻塞挂起，阻塞等待调用方法`await`方法，会根据计数值选择是否阻塞等待。