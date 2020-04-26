# **一、简介**

## **1. 为什么要有ConcurrentHashMap**

`HashMap`是我们日常编程中最用得最频繁的集合类，但是在多线程并发的情况下，`HashMap`可能会造成死循环或者更新丢失（具体可以参考这篇文章：）。早期JDK中为了解决这个问题，为我们提供了两种方案

* **HashTable**

  `HashTable`是线程安全的Map，它的**所有方法使用synchronized关键字修饰**，保证同一时刻只有一个线程可以操作集合，从而保证了线程安全，但是该方式属于**独占锁方式**，一个线程持有其余线程必须等待，**性能较低**。

* **Collections.synchronizedMap(hashMap)**

  **这种方式同上，在方法内使用synchronized代码块，独占式操作集合，性能较低**，如下

  ```java
  public V put(K key, V value) {
      synchronized (mutex) {return m.put(key, value);}
  }
  ```

因此`Doug Lea`为我们提供了**高性能**的**线程安全**的`HashMap`——`ConcurrentHashMap`

## **2. ConcurrentHashMap简介**

### **2.1 JDK1.7及之前版本**

在JDK1.5~1.7版本中，Java使用了分段锁实现`ConcurrentHashMap`，它的底层是**数组+链表+Segment分段锁+HashEntry**实现。其中`Segment`在实现上继承了`ReentrantLock`，这样就自带了锁的功能。每个`Segment`会锁定一段数组长度

### **2.2 JDK1.8+版本**

# **二、类总览**

![](http://img.xianzilei.cn/ConcurrentHashMap%E7%B1%BB%E5%9B%BE.png)

# **三、核心方法**



# **四、总结**

# **五、参考**

* [【死磕Java并发】—–J.U.C之Java并发容器：ConcurrentHashMap](http://cmsblogs.com/?p=2283)
* https://www.pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html#concurrenthashmap---jdk-17
* https://juejin.im/post/5aeeaba8f265da0b9d781d16#heading-0