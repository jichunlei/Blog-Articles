# **一、什么是ThreadLocal**

我们先看一下**API说明（1.8版本）**

```java
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 */
```

翻译过来就是

> 此类提供**线程局部变量**。 这些变量与普通变量不同，**每个使用该变量（通过其get或set方法）的线程都会初始化一个完全独立的实例副本**。`ThreadLocal` 实例通常是类中的 private static 字段，它们希望将**状态与某一个线程（例如，用户 ID 或事务 ID）相关联**。

总的来说：`ThreadLocal`与线程同步机制不同，**线程同步机制是多个线程共享同一个变量**，而`ThreadLocal`是**为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本**。可以说`ThreadLocal`为多线程环境下变量问题提供了另外一种解决思路，实质就是一种“空间换时间”的思想。

# **二、为什么要使用ThreadLocal**

从上面的的定义我们知道`ThreadLocal`就是定义了线程的本地变量，那么这样做有什么样的好处呢？我们知道如果我们想要在方法内使用到方法外的变量（不包括当前类或父类中的成员属性），我们一般会采用如下的方式

* **1）方法传参**
* **2）将需要使用的变量定义为类的静态变量**

上面两种方式虽然可行，但是有其弊端。如果方法的调用层级很深入，给方法加上参数传递会导致其余方法也需要修改，而且可能很多地方会使用到这个方法，**修改的成本过大且容易出错**；使用类的静态变量会导致别的线程也可以访问到该变量，可能会**导致线程安全**问题。

因此Java提供了`ThreadLocal`，来表示**线程的本地变量**，**线程间互不影响，每个线程在任何地方都可以取到该变量，省去了参数传递的麻烦，同时也保证了线程安全的问题**。

# **三、ThreadLocal用法**

举个简单的小例子（这个例子可能会不能明显地体现ThreadLocal的优势，但是讲解**如何使用**已经足够了）

```java
public static void main(String[] args) {
    ThreadLocal<String> threadLocal = new ThreadLocal<>();
    for (int i = 1; i <= 5; i++) {
        int j = i;
        new Thread(() -> {
            String name = Thread.currentThread().getName();
            System.out.println("设置" + name + "的threadLocal为:" + name);
            threadLocal.set(name);
            System.out.println(">>>取出" + name + "的threadLocal:" + threadLocal.get());
        }, i + "号线程").start();

    }
}
```

执行结果

```tex
设置1号线程的threadLocal为:1号线程
设置5号线程的threadLocal为:5号线程
>>>取出5号线程的threadLocal:5号线程
设置4号线程的threadLocal为:4号线程
设置3号线程的threadLocal为:3号线程
设置2号线程的threadLocal为:2号线程
>>>取出2号线程的threadLocal:2号线程
>>>取出3号线程的threadLocal:3号线程
>>>取出4号线程的threadLocal:4号线程
>>>取出1号线程的threadLocal:1号线程
```

ThreadLocal的用法其实非常简单，只需要实例化一个实例，**调用set方法设置本地变量，get方法获取到本地变量**。

# **四、ThreadLocal原理**

## **1. ThreadLocalMap**

在介绍原理之前，我们先介绍一下**ThreadLocal的一个内部类——ThreadLocalMap**

### **1.1 属性**

```java
//初始容量，必须是2的整数倍
private static final int INITIAL_CAPACITY = 16;

//Entry类型的数组
private Entry[] table;

//数组元素个数
private int size = 0;

//扩容阈值，当 table中存储的元素个数达到该值时就会扩容
private int threshold;
```

**属性类似hashmap，只不过这里固定数组的类型为Entry**，这个Entry是它的内部类，定义如下

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    //需要存储的值
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        //以弱引用的方式保存ThreadLocal
        super(k);
        value = v;
    }
}
```

**Entry类继承WeakReference（表示弱引用），它的构造方法中将ThreadLocal以弱引用的方式保存**。

### **1.2 构造方法**

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //创建数组
    table = new Entry[INITIAL_CAPACITY];
    //计算桶位置
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    //创建Entry实例后放到指定位置
    table[i] = new Entry(firstKey, firstValue);
    //初始化size
    size = 1;
    //设置阈值为初始大小
    setThreshold(INITIAL_CAPACITY);
}
```

### **1.3 核心方法**

* **保存键值**

  ```java
  private void set(ThreadLocal<?> key, Object value) {
  
      //保存table数组
      Entry[] tab = table;
      //数组长度
      int len = tab.length;
      //计算要存储的索引位置，直接使用key的threadLocalHashCode来计算
      int i = key.threadLocalHashCode & (len-1);
  
      //采用开放地址法，hash冲突的时候使用线性探测
      //即如果i位置已经有元素了，就会获取下一个位置，如果已经到达最后位置，则循环到队头继续查找，直到找到空位置（以上是key不存在的情况下，如果存在则直接覆盖value，如果原位置的k失效则覆盖位置）
      for (Entry e = tab[i];
           e != null;
           e = tab[i = nextIndex(i, len)]) {
          //获取当前位置
          ThreadLocal<?> k = e.get();
  		//如果key相同则覆盖
          if (k == key) {
              e.value = value;
              return;
          }
  		//如果k为空，说明该位置已经被回收了，所以直接覆盖该位置
          if (k == null) {
              replaceStaleEntry(key, value, i);
              return;
          }
      }
  
      //能够执行到此处说明该位置有空位，直接新建个Entry保存在此位置
      tab[i] = new Entry(key, value);
      int sz = ++size;
      //清除一些无用的元素再判断是否大小是否达到了阈值，如果达到则扩容
      if (!cleanSomeSlots(i, sz) && sz >= threshold)
          //扩容，该处不是本文的重点，这里不细讲，感兴趣的小伙伴可以扒源码分析
          rehash();
  }
  ```

* **获取Entry对象**

  ```java
  private Entry getEntry(ThreadLocal<?> key) {
      //计算存储的位置
      int i = key.threadLocalHashCode & (table.length - 1);
      //获取该位置的数据
      Entry e = table[i];
      //如果e存在且key一致，直接返回
      if (e != null && e.get() == key)
          return e;
      //否则做进一步查找
      else
          return getEntryAfterMiss(key, i, e);
  }
  
  ```

  当在指定位置找不到对应的Entry时，因为可能存在hash冲突，会采用开放地址法解决冲突，所以需要进一步查找数据，方法定义如下

  ```java
  private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
      Entry[] tab = table;
      int len = tab.length;
  	
      while (e != null) {
          ThreadLocal<?> k = e.get();
          //如果key一致，则返回该 Entry
          if (k == key)
              return e;
          //如果k为null，说明该处无效，需要清除掉
          if (k == null)
              expungeStaleEntry(i);
          //否则继续查找下一个位置
          else
              i = nextIndex(i, len);
          e = tab[i];
      }
      return null;
  }
  ```

* **移除指定Entry**

  ```java
  private void remove(ThreadLocal<?> key) {
      Entry[] tab = table;
      int len = tab.length;
      //计算存储位置
      int i = key.threadLocalHashCode & (len-1);
      //循环查找key对应的Entry
      for (Entry e = tab[i];
           e != null;
           e = tab[i = nextIndex(i, len)]) {
          //如果key一致，则清除
          if (e.get() == key) {
              e.clear();
              expungeStaleEntry(i);
              return;
          }
      }
  }
  ```

## **2. 存值**

**ThreadLocal提供set方法来设置当前线程的线程局部变量的值**

```java
public void set(T value) {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        //设置value
        map.set(this, value);
    //如果map为空则会初始化map
    else
        createMap(t, value);
}

//获取当前线程的ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}


void createMap(Thread t, T firstValue) {
    //创建ThreadLocalMap对象赋值给当前线程的threadLocals属性
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

**我们可以看到，每个Thread类中都存在一个ThreadLocalMap属性**，我们要存在的值就存在该map中的Entry节点的value属性中，我们可以通过ThreadLocal作为key去找到Entry节点

```java
//Thread类中
ThreadLocal.ThreadLocalMap threadLocals = null;
```

## **3. 取值**

**ThreadLocal提供set方法来返回当前线程所对应的线程变量**

```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //如果当前线程不为空
    if (map != null) {
        //获取对应位置的Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        //如果Entry不为空，则返回其value
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //否则返回初始值
    return setInitialValue();
}
```

## **4. 移除值**

**ThreadLocal提供remove方法移除当前 ThreadLocal 对应的值**

```java
public void remove() {
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        //移除ThreadLocal对应的value
        m.remove(this);
}
```

# **五、ThreadLocal的内存泄漏问题**

我们来看一下`ThreadLocal`的对象关系引用图（**其中实线表示强引用**）

![](http://img.xianzilei.cn/ThreadLocal%E5%90%84%E5%BC%95%E7%94%A8%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB2.png)

我们知道弱引用是为了利于GC回收的，如果一个对象只有弱引用，GC就会回收这个对象。当JVM栈中的ThreadLocal对象的引用为null时，ThreadLocal对象就只剩下弱引用了，此时就会回收ThreadLocal对象，同时Entry中的key对象也会置为空，但是Entry中的value不会被回收，因为根据**可达性**分析，value存在这样一条链路：**Thread对象引用->Thread对象->ThreadLocalMap对象->Entry对象->Value对象**，因此value不会被回收，与线程同生命周期，这样就存在了**内存泄漏**，尤其在使用线程池的时候。**因此在ThreadLocalMap中的setEntry()、getEntry()，如果遇到key == null的情况，会对value设置为null。当然我们也可以显示调用ThreadLocal的remove()方法进行处理**。所以实际上从`ThreadLocal`设计角度来说是不会导致**内存泄露**的！

# **六、总结**

* `ThreadLocal`提供线程内部的局部变量，在本线程内随时随地可取，隔离其他线程
* `ThreadLocal`的设计是：每个`Thread`维护一个`ThreadLocalMap`哈希表，这个哈希表的`key`是`ThreadLocal`实例本身，`value`才是真正要存储的值`Object`。
* 对`ThreadLocal`的常用操作实际是对线程`Thread`中的`ThreadLocalMap`进行操作。
* `ThreadLocalMap`中的哈希表`Entry[] table`存储的核心元素是`Entry`，存储的`key`是`ThreadLocal`实例对象，`value`是`ThreadLocal` 对应储存的值`value`。需要注意的是，此`Entry`继承了弱引用 `WeakReference`，所以在使用`ThreadLocalMap`时，发现`key == null`，则意味着此`key  ThreadLocal`不在被引用，需要将其从`ThreadLocalMap`哈希表中移除。
* `ThreadLocalMap`使用`ThreadLocal`的弱引用作为`key`，如果一个`ThreadLocal`没有外部强引用来引用它，那么系统 GC 的时候，这个`ThreadLocal`势必会被回收。所以，在`ThreadLocal`的`get()`,`set()`,`remove()`的时候都会清除线程`ThreadLocalMap`里所有`key`为`null`的`value`。如果我们不主动调用上述操作，则会导致内存泄露。
* 为了安全地使用`ThreadLocal`，必须要像每次使用完锁就解锁一样，在每次使用完`ThreadLocal`后都要调用`remove()`来清理无用的`Entry`。这在操作在使用线程池时尤为重要。