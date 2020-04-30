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

在JDK1.5~1.7版本中，Java使用了分段锁实现`ConcurrentHashMap`，它的底层是**数组+链表+Segment分段锁+HashEntry**实现。其中`Segment`在实现上继承了`ReentrantLock`，这样就自带了锁的功能。**每个Segment会锁定一段数组长度**，结构如下（1.7版本不在讲解范围内，细节的话这里略过，小伙伴可以自行研究）

![](http://img.xianzilei.cn/ConcurrentHashMap1.7%E7%89%88%E6%9C%AC%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

其中**Segment的数量又叫并发级别，并发数。默认是16个，且固定为2的正整数次幂**。即默认情况下理论上可以支持16个线程并发写，只要他们操作分布在不同的Segment上。注意：**并发数在初始化时候可以自己设置，但是一旦初始化以后就不可用改变**。

### **2.2 JDK1.8+版本**

1.7版本的`ConcurrentHashMap`性能完全ConcurrentHashMap。因此在1.8版本中它抛弃了分段锁的设置，直接**采用了synchronized+CAS**来实现。因此在1.6版本中JDK对`synchronized`做了很大的优化，包括**偏向锁，轻量级锁和重量级锁**（具体可以参考之前的文章：[深入理解synchronized底层原理](http://www.xianzilei.cn/blog/54)），使得`synchronized`的性能与lock差不多，甚至在某些情况下由于lock。另外1.8版本的`ConcurrentHashMap`的底层数据结构采用了**数组+链表+红黑树**的形式。下**面将以1.8版本为主进行讲解**。

# **二、类总览**

![](http://img.xianzilei.cn/ConcurrentHashMap%E7%B1%BB%E5%9B%BE.png)

## **1. 数据结构**

数据结构同`HashMap`，都是**数组+链表+红黑树**形式，这里不再赘述

## **2. 核心属性**

```java
//序列化版本号
private static final long serialVersionUID = 7249069246763182397L;

//最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;

//默认初始容量
private static final int DEFAULT_CAPACITY = 16;

//数组最大长度
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//默认的并发等级，这里并未使用，主要是为与该类的以前版本兼容而定义
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

//加载因子
private static final float LOAD_FACTOR = 0.75f;

//链表转红黑树阈值
static final int TREEIFY_THRESHOLD = 8;

//红黑树转链表阈值
static final int UNTREEIFY_THRESHOLD = 6;

//链表转红黑树的最小元素个数
static final int MIN_TREEIFY_CAPACITY = 64;

//桶数组
transient volatile Node<K,V>[] table;

//扩容时使用，平时为 null
private transient volatile Node<K,V>[] nextTable;

//基本计数器值
private transient volatile long baseCount;

//控制标识符，用来控制table初始化和扩容操作的，在不同的地方有不同的用途，其值也不同，所代表的含义也不同
//-1:代表table正在初始化,其他线程应该交出CPU时间片
//-N:表示正有N-1个线程执行扩容操作（高 16 位是 length 生成的标识符，低 16 位是扩容的线程数）
//大于0:如果table已经初始化,代表table容量,默认为table大小的0.75,如果还未初始化,代表需要初始化的大小
private transient volatile int sizeCtl;

/**
     * The next table index (plus one) to split while resizing.
     */
private transient volatile int transferIndex;

/**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
private transient volatile int cellsBusy;

/**
     * Table of counter cells. When non-null, size is a power of 2.
     */
private transient volatile CounterCell[] counterCells;

// views
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
```

## **3. 核心内部类**

### **3.1 Node**

类似于`HashMap`中的`Node`类，存放**键-值对**

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    //value，采用volatile修饰，保证可见性
    volatile V val;
    //下一个节点的指针，采用volatile修饰，保证可见性
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    
    //不允许修改value值，内部直接抛出异常
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

与`HashMap`的`Node`不同的是，他的**value和next属性都使用了volatile修饰，不允许修改value的值（对value的setter方法进行了修改），同时提供find方法供map.get调用**。

### **3.2 TreeNode**

```java

```



### **3.3 TreeBin**

### **3.4 ForwardingNode**

## **4. 构造方法**

# **三、核心方法**



# **四、总结**

# **五、参考**

* [【死磕Java并发】—–J.U.C之Java并发容器：ConcurrentHashMap](http://cmsblogs.com/?p=2283)
* https://www.pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html#concurrenthashmap---jdk-17
* https://juejin.im/post/5aeeaba8f265da0b9d781d16#heading-0