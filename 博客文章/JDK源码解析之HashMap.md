# **一、简介**

> 是一种**散列表**，用于存储 **key-value 键值对**的数据结构

* `HashMap`最早出现在 **JDK 1.2**中，底层基于**散列算法**实现。
* `HashMap`允许 **null 键**和 **null 值**，在计算键的哈希值时，**null 键哈希值为 0**。
* `HashMap`并**不保证键值对的顺序**，这意味着在进行某些操作后，键值对的顺序可能会**发生变化**。
* 下面的讲解以**1.8版本**为准，适当的时候会与1.8之前的版本作比较

# **二、底层原理**

## **1. JDK1.8版本之前**

底层采用**数组+链表**的数据结构来实现，如下图所示

![](http://img.xianzilei.cn/HashMap%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-1.png)

添加元素时，获取key的hashCode，经过扰动函数处理后得到的hash值，然后通过`(n-1) & hash`判断当前元素存放的位置（这里的n是数组的长度），如果当前位置存在元素（链表）的话，则判断是否存在相同key的节点，如果存在则覆盖值，否则使用**拉链法**解决冲突。

这里的扰动函数指的就是HashMap的hash方法，使用扰动函数是为了**防止一些实现比较差的 hashCode() 方法，减少冲突概率**。这里的哈希函数后后第五章会详解讲解。

## **2. JDK1.8版本及之后**

到了1.8版本为了**优化查询效率**，底层采用了**数组+链表+红黑树**的数据结构实现，如下图所示

![](http://img.xianzilei.cn/HashMap%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-2.png)

jdk1.8版本之前，如果链表长度过长的时候，就会导致查询速度大大降低，为了提高查找效率，jdk1.8引入了新的数据结构——**红黑树**（如果不了解红黑树的小伙伴可以浏览之前的文章：[一文彻底弄懂红黑树](http://xianzilei.cn/blog/41)）

* **当链表长度大于等于8时，原来的线性结构会升级为红黑树结构**
* **当链表长度小于等于6时，为了减少不必要的空间损耗，会退化为线性链表结构**

# **三、类图**

![](http://img.xianzilei.cn/HashMap%E7%B1%BB%E5%9B%BE.png)

可以看到`HashMap`实现了**三个接口**

* 实现`java.util.Map`接口
* 实现`java.io.Serializable`接口
* 实现`java.lang.Cloneable`接口

同时继承了**一个抽象类**

* 继承`java.util.AbstractMap`抽象类

# **四、属性和构造方法**

## **1. 属性**

```java
//序列化版本号
private static final long serialVersionUID = 362498820763181265L;

//默认初始容量16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

//最大容量2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认的填充因子0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//当桶(bucket)上的结点数大于这个值8时会转成红黑树
static final int TREEIFY_THRESHOLD = 8;

//当桶(bucket)上的结点数小于这个值6时树转链表
static final int UNTREEIFY_THRESHOLD = 6;

//桶中结构转化为红黑树对应的table的最小大小64
static final int MIN_TREEIFY_CAPACITY = 64;

//存储元素的数组，总是2的幂次倍
transient Node<K,V>[] table;

//存放具体元素的集合
transient Set<Map.Entry<K,V>> entrySet;

//存放元素的个数，注意这个不等于数组的长度
transient int size;

//每次扩容和更改map结构的计数器
transient int modCount;

//阀值 当实际大小(容量*填充因子)超过临界值时，会进行扩容，它的值总是2的整数次幂
int threshold;

//加载因子
final float loadFactor;
```

下面讲解一下几个重点属性

* **loadFactor**

  * 加载因子，用来**控制数组存放数据的疏密程度**。当`loadFactor`越趋近于1，则数组中存放的数据越多，即数据越密集。反之loadFactor越趋近于0，数组存放的数据越少，数据越稀疏。

  * loadFactor太大导致查找元素效率低，太小导致数组的利用率低，存放的数据会很分散。**loadFactor的默认值为0.75f是官方给出的一个比较好的临界值**。例如给定的默认容量为16，默认的负载因子为0.75，则当数据个数达到16*0.75=12时，就需要进行扩容处理。

* **threshold**

  **临界值，值等于容量乘以填充因子，当size大于等于该值时就需要考虑对数组进行扩容**。

* **table**

  存储元素的数组，类型为**Node节点类**，**它是HashMap的一个内部类**

  ```java
  static class Node<K,V> implements Map.Entry<K,V> {
      //哈希值
      final int hash;
      //键
      final K key;
      //值
      V value;
      //后继节点指向
      Node<K,V> next;
  
      Node(int hash, K key, V value, Node<K,V> next) {
          this.hash = hash;
          this.key = key;
          this.value = value;
          this.next = next;
      }
  
      public final K getKey()        { return key; }
      public final V getValue()      { return value; }
      public final String toString() { return key + "=" + value; }
  
      public final int hashCode() {
          return Objects.hashCode(key) ^ Objects.hashCode(value);
      }
  
      public final V setValue(V newValue) {
          V oldValue = value;
          value = newValue;
          return oldValue;
      }
  
      public final boolean equals(Object o) {
          if (o == this)
              return true;
          if (o instanceof Map.Entry) {
              Map.Entry<?,?> e = (Map.Entry<?,?>)o;
              if (Objects.equals(key, e.getKey()) &&
                  Objects.equals(value, e.getValue()))
                  return true;
          }
          return false;
      }
  }
  ```

* **entrySet**

  **存放所有的键值对的set集合**

* 补充：**TreeNode节点类**

  **TreeNode继承于Node，作为红黑树的节点**。（红黑树的代码逻辑较复杂，这么不作为重点讲解，大家感兴趣的可以研究一下）

  ```java
  //红黑树节点类
  static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
      //父节点
      TreeNode<K,V> parent;  // red-black tree links
      //左子树节点
      TreeNode<K,V> left;
      //右子树节点
      TreeNode<K,V> right;
      //直接前驱节点
      TreeNode<K,V> prev;    // needed to unlink next upon deletion
      //颜色，true表示红色，false表示黑色
      boolean red;
      TreeNode(int hash, K key, V val, Node<K,V> next) {
          super(hash, key, val, next);
      }
  
      /**
           * Returns root of tree containing this node.
           */
      final TreeNode<K,V> root() {
          for (TreeNode<K,V> r = this, p;;) {
              if ((p = r.parent) == null)
                  return r;
              r = p;
          }
      }
      
    //以下代码省略...  
  ```

## **2. 构造方法**

HashMap一共**四个构造方法**

* **1）HashMap()**

  **无参构造方法**

  ```java
  public HashMap() {
      //加载因子设置为默认值0.75
      this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
  }
  ```

  无参构造方法初始化loadFactor为默认的0.75，存放元素的table采用**延迟初始化**的方式。即**首次添加元素后真正初始化**。

* **2）HashMap(int)**

  **指定容量的有参构造函数**

  ```java
  public HashMap(int initialCapacity) {
      //调用第三个构造方法，同时设置加载因子为默认的0.75
      this(initialCapacity, DEFAULT_LOAD_FACTOR);
  }
  ```

* **3）HashMap(int, float)**

  **指定容量和负载因子的有参构造函数**

  ```java
  public HashMap(int initialCapacity, float loadFactor) {
      //默认初始化大小必须大于等于0
      if (initialCapacity < 0)
          throw new IllegalArgumentException("Illegal initial capacity: " +
                                             initialCapacity);
      //限制初始化的最大容量为默认的最大容量，即2的30次方
      if (initialCapacity > MAXIMUM_CAPACITY)
          initialCapacity = MAXIMUM_CAPACITY;
      //加载因子不能小于等于0或无限小
      if (loadFactor <= 0 || Float.isNaN(loadFactor))
          throw new IllegalArgumentException("Illegal load factor: " +
                                             loadFactor);
      //设置loadFactor为指定值
      this.loadFactor = loadFactor;
      //计算阀值
      this.threshold = tableSizeFor(initialCapacity);
  }
  ```

  这里有个计算阀值的方法`tableSizeFor`需要说一下

  ```java
  //该方法是返回大于或等于输入参数且最近的2的整数次幂的数
  static final int tableSizeFor(int cap) {
      //这里需要减1，是为了另找到的目标值大于或等于原值
      int n = cap - 1;
      n |= n >>> 1;
      n |= n >>> 2;
      n |= n >>> 4;
      n |= n >>> 8;
      n |= n >>> 16;
      return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
  }
  ```

  这里一连串的位运算暂不研究其原理，毕竟不是本文的重点。但是需要说一下第一行的减1操作，这里是**为了使找到的目标值大于或等于原值**，因为该方法是**返回大于或等于输入参数且最小的2的整数次幂的数**。这样说会比较抽象，我们来举一个例子，如果入参cap=8，如果不减1直接计算，得到的结果就是16，然而我们知道大于等于8的最小的2的整数次幂就是8，不符合方法的意思，所以才**预先减1操作**。

  **扩展**：**为什么MAXIMUM_CAPACITY是2的30次方？**

  这是因为：**我们知道int的最大值是2的31次方-1，而我们要求阈值必须为2的整数次幂，所以最大值为2的30次方**。

* **4）HashMap(Map<? extends K, ? extends V>)**

  **指定集合的有参构造函数**

  ```java
  public HashMap(Map<? extends K, ? extends V> m) {
      //设置加载因子为默认的0.75
      this.loadFactor = DEFAULT_LOAD_FACTOR;
      //将集合m批量添加到map中
      putMapEntries(m, false);
  }
  ```

  内部调用`putMapEntries`方法

  ```java
  //上述构造函数调用该方法传递的第二个参数默认为false
  final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
      //获取入参集合的长度
      int s = m.size();
      //集合长度不为空才进行添加
      if (s > 0) {
          //如果table没有初始化
          if (table == null) { // pre-size
              //计算需要最小的tables大小，这里加1是为了防止下面的取值导致大小不够
              float ft = ((float)s / loadFactor) + 1.0F;
              //限制tables大小最大为默认的最大值2的30次方
              int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                       (int)ft : MAXIMUM_CAPACITY);
              //如果计算所需最小table大小大于阈值，则需要初始化阈值
              if (t > threshold)
                  threshold = tableSizeFor(t);
          }
          //如果table已经初始化，判断入参集合长度是否超过阈值，如果超过则需要进行扩容
          else if (s > threshold)
              //扩容操作，后面扩容一节会详细讲解
              resize();
          
          //循环集合添加元素到初始化后的map中
          for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
              //获取键
              K key = e.getKey();
              //获取值
              V value = e.getValue();
              //添加元素，后面添加操作一节会详细讲解
              putVal(hash(key), key, value, false, evict);
          }
      }
  }
  ```

  该方法主要包含**两大步骤**：

  * **第一步：保证所需的最小容量**
    * 如果table未初始化，则我们只需要保证阈值threshold大于或等于入参集合的大小即可，后续首次添加键值对的时候再去做真正的table初始化
    * 如果table已经初始化，则我们需要保证容量足够添加这些元素，否则进行扩容（扩容逻辑第六章会详细讲解）
  * **第二步：遍历添加元素**（添加元素第七章会详细讲解）

# **五、哈希函数**

在前面我们说过，添加元素时，我们需要获取key的hash值，通过hash值找到对应存放的桶位置。在HashMap中，该方法调用的频率非常高，因此该方法的**执行性能必须足够高**，同时计算出来的**哈希值必须足够离散，从而减少冲突概率**。下面我列举出jdk1.7和1.8版本中hash方法源码，对比一下二者的区别

* **JDK1.7**

  ```java
  static int hash(int h) {
      h ^= (h >>> 20) ^ (h >>> 12);
      return h ^ (h >>> 7) ^ (h >>> 4);
  }
  ```

* **JDK1.8**

  ```java
  static final int hash(Object key) {
      int h;
      //key为null时直接返回0
      //key.hashCode()指的是key自带的hashcode方法
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
  ```

上面的方法又叫**扰动函数**，可以看到1.7版本需要对h扰动四次，性能较差，而**1.8只需要扰动一次，性能上得到了提升**，同时**采用16位右位移异或运算，保证了获得的哈希值足够离散**（至于为什么博主也不是很清楚，大家可以查看这里的讲解：[JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617/answer/111577937) ）。

# **六、扩容操作**

提供上面的分析，**HashMap扩容采用的是resize方法，同时它还可以初始化数组**

```java
final Node<K,V>[] resize() {
    //保存原数组
    Node<K,V>[] oldTab = table;
    //原数组长度，如果数组为空则为0
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //原阈值
    int oldThr = threshold;
    //定义新数组长度newCap和新阈值newThr
    int newCap, newThr = 0;
    // 如果原数组长度大于0，表明table已经初始化过了
    if (oldCap > 0) {
        //如果原数组长度已经大于等于容量的最大值，则不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            //阈值调整为Integer最大值，即2的31次方-1
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //如果数组长度没有超过容量最大值，则更新newCap, newThr为原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //原数组长度等于0，表示table未初始化，如果此时oldThr大于0，newCap赋值为oldThr
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //原数组长度等于0，且oldThr等于0
    else {               // zero initial threshold signifies using defaults
        //新数组容量为默认值16
        newCap = DEFAULT_INITIAL_CAPACITY;
        //新阈值为默认的负载因子0.75*默认数组容量16=12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    //如果上面没有计算出新阈值，则使用 newCap * loadFactor 作为新的阀值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //将 newThr 赋值给 threshold 属性
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //创建新的 Node 数组，赋值给 table 属性
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        //循环将旧数组中的元素搬到新数组中
        for (int j = 0; j < oldCap; ++j) {
            //保存旧数组第j位置的Node节点
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                //清除j位置节点（置空）
                oldTab[j] = null;
                //如果e的后继为空，即该位置只有一个节点
                if (e.next == null)
                    //将e装到新数组中
                    newTab[e.hash & (newCap - 1)] = e;
                //如果e节点是红黑树类型，需要对红黑树进行分裂处理，并将分裂后的数据存放到新数组中
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果e节点是链表类型，则进行链表的拆分操作
                //因为HashMap是成倍扩容的，而且扩容的大小始终是2的整数倍，所以原来位置的链表节点会分散新数组的两个位置，分别是j和j+oldCap的位置
                //这里使用了一点技巧，避免一个一个遍历影响性能，分析如下
                // 通过 e.hash & oldCap 计算，根据结果分到高位、和低位的位置中。
                // 情况1. 如果结果为 0 时，则放置到低位
                // 情况2. 如果结果非 1 时，则放置到高位
                else { // preserve order
                    //这里定义两组变量分别存储低位和高位的链表头尾节点
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    //之所以使用do while是因为e已经是非空，无需再判断，直接指向即可
                    do {
                        //保存e的下一个节点
                        next = e.next;
                        //满足情况1，需要放置到低位
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //否则属于情况2，放置到高位
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    //循环结束后将拆分的两个链表组装到新数组中
                    //设置低位到新数组的j位置上
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //设置高位到新数组的j+oldCap位置上
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

上述代码看起来很复杂，其实就做了**三件事情**

* 1）计算新数组的**新容量newCap**和**新阈值newThr**；
* 2）根据计算出的newCap**创建新数组**；
* 3）**将键值对重新映射到新数组中**。

**接下来一步一步分析内部逻辑**

## **1. 计算新容量和新阈值**

其中第一步的逻辑可能有些绕，我们来简化分析一下，将第一步的代码简化一下，这里一共有三个条件分支

```java
//条件分支1
if (oldCap > 0) {
    //条件分支2
    if (oldCap >= MAXIMUM_CAPACITY) {
        ......
    }
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
        ......
}
else if (oldThr > 0) 
    ......
else {               
    ......
}
//条件分支3
if (newThr == 0) {
    ......
}
```

* **条件分支1**

  | 条件                     | 说明                                                         |
  | ------------------------ | ------------------------------------------------------------ |
  | oldCap > 0               | **原数组 table 已经被初始化**，无需操作                      |
  | oldCap = 0且oldThr > 0   | **原数组 table 未被初始化，但是阈值threshold已经初始化了**。这种是调用构造函数HashMap(int) 和 HashMap(int, float)且尚未添加元素造成的。这种情况预先newCap = oldThr，如果newThr=0， 在条件分支3中会重新计算。 |
  | oldCap == 0且oldThr == 0 | **原数组未被初始化，且阈值threshold等于0**。这种是调用无参构造函数造成的。这里会全部设置为**默认值**，newCap为16，newThr为16*0.75=12 |

* **条件分支2**

  | 条件                                                         | 说明                                                         |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | oldCap >= MAXIMUM_CAPACITY                                   | **原数组容量已经大于或等于最大容量了**。这种情况无法进行扩容，直接结束返回。 |
  | (newCap = oldCap << 1) < MAXIMUM_CAPACITY且oldCap >= DEFAULT_INITIAL_CAPACITY | **新数组容量设置为旧数组的两倍后小于最大值且旧数组容量大于或等于16**。这种情况新阈值扩大为原来的两倍，但是可能会在移位的过程中导致溢出（负载因子大于1的时候），因此条件分支3会做相应的补救处理。 |

* **条件分支3**

  | 条件        | 说明                                                         |
  | ----------- | ------------------------------------------------------------ |
  | newThr == 0 | **发生在条件分支1未计算 newThr 或条件分支2在计算过程中导致 newThr 溢出归零**。newThr=newCap * loadFactor，如果大于最大容量则更新为最大容量 |

## **2. 创建新数组**

这里没什么好说的，**就是根据新容量newCap直接new一个新数组出来，并赋值给table**

```java
@SuppressWarnings({"rawtypes","unchecked"})
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
```

## **3. 重新映射键值对**

因为节点类型有两种，一种是**红黑树节点类型**，另一种是**普通节点类型**

- 如果节点是红黑树类型，则需要将**拆分红黑树**

  ```java
  ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
  ```

  该方法涉及逻辑较为复杂，且红黑树不是本文重点，所以简单略过。**实质就将红黑树拆分成两条由 TreeNode 组成的链表**。如果链表长度小于 `UNTREEIFY_THRESHOLD`，则将链表转换成普通链表。否则根据条件重新将 `TreeNode` 链表树化。

- 如果节点类型是普通节点，则需要**按照原顺序分组**

  ```java
  //如果e节点是链表类型，则进行链表的拆分操作
  //因为HashMap是成倍扩容的，而且扩容的大小始终是2的整数倍，所以原来位置的链表节点会分散新数组的两个位置，分别是j和j+oldCap的位置
  //这里使用了一点技巧，避免一个一个遍历影响性能，分析如下
  // 通过 e.hash & oldCap 计算，根据结果分到高位、和低位的位置中。
  // 情况1. 如果结果为 0 时，则放置到低位
  // 情况2. 如果结果非 1 时，则放置到高位
  //这里定义两组变量分别存储低位和高位的链表头尾节点
  Node<K,V> loHead = null, loTail = null;
  Node<K,V> hiHead = null, hiTail = null;
  Node<K,V> next;
  //之所以使用do while是因为e已经是非空，无需再判断，直接指向即可
  do {
      //保存e的下一个节点
      next = e.next;
      //满足情况1，需要放置到低位
      if ((e.hash & oldCap) == 0) {
          if (loTail == null)
              loHead = e;
          else
              loTail.next = e;
          loTail = e;
      }
      //否则属于情况2，放置到高位
      else {
          if (hiTail == null)
              hiHead = e;
          else
              hiTail.next = e;
          hiTail = e;
      }
  } while ((e = next) != null);
  
  //循环结束后将拆分的两个链表组装到新数组中
  //设置低位到新数组的j位置上
  if (loTail != null) {
      loTail.next = null;
      newTab[j] = loHead;
  }
  //设置高位到新数组的j+oldCap位置上
  if (hiTail != null) {
      hiTail.next = null;
      newTab[j + oldCap] = hiHead;
  }
  ```

  实现逻辑与红黑树的拆分类似。因为HashMap是成倍扩容的，而且扩容的大小始终是2的整数倍，所以原来位置的链表节点会分散新数组的两个位置，分别是`j`和`j+oldCap`的位置。这里通过 `e.hash & oldCap` 计算，根据结果分到**高位**、和**低位**的位置中

  * **如果结果为 0 时，则放置到低位**
  * **如果结果非 1 时，则放置到高位**

# **七、树化与链化**

在说添加操作的时候，我们先来了解一下**树化**和**链化**操作。

* **当链表长度大于8时，需要执行树化操作，即将链表结构升级为红黑树结构**
* **当链表长度小于6时，需要执行链化操作，即将红黑树结构退化为链表结构**

## **1. 树化**

```java
//树化操作
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果table没有初始化或者容量小于最小树化容量64，优先进行扩容而不是树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 如果hash计算出来的位置存在节点，则需要进行树化
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        //定义头结点hd和尾结点tl
        TreeNode<K,V> hd = null, tl = null;
        //顺序遍历链表，逐个转换成红黑树节点
        do {
            //将普通节点替换成树形节点，实质就是new了一个TreeNode节点，保存hash、key、value的后继节点，不过初始默认为null
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //初始化头结点hd
            if (tl == null)
                hd = p;
            //执行链接操作，即虽然修改为树节点，但是还是保存了原来链接链接的形式
            else {
                p.prev = tl;
                tl.next = p;
            }
            //每次更新尾结点hd
            tl = p;
        } while ((e = e.next) != null);
        //真正的树化操作
        if ((tab[index] = hd) != null)
            //红黑树树化操作，内部逻辑较为复杂，这里就不深究了
            hd.treeify(tab);
    }
}

// 将普通节点替换成树形节点
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

在上面的代码中有两点我们需要注意：

* 如果数组的长度小于MIN_TREEIFY_CAPACITY的时候，这里优先选择扩容，而不是树化。其实也比较容易理解，当数组容量比较小的时候，键值得冲突会比较高，此时应该先扩容，而不是立马树化，因为容量过小是主要因素。而且数组容量过小很容易导致扩容，而扩容又会拆分红黑树重新映射。因此在数组容量较小的时候，优先扩容而不是树化。
* 在链表节点转换成红黑树节点的时候，红黑树节点依然保存了原链表的节点顺序，因此在下面的红黑树退化为链表（即链化操作），就显得尤为简单。

## **2. 链化**

红黑树中**保留了原链表节点顺序**，因此链化操作相对简单一些。当红黑树的内部节点数小于等于`UNTREEIFY_THRESHOLD`时，此时会触发链化操作。

```java
//链化
final Node<K,V> untreeify(HashMap<K,V> map) {
    //头结点hd和尾结点tl
    Node<K,V> hd = null, tl = null;
    //遍历树节点，并替换成普通节点
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}

//树节点替换成普通节点
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    return new Node<>(p.hash, p.key, p.value, next);
}
```

# **八、添加操作**

## **1. 添加单个元素**

* **put(K, V)**

  ```java
  public V put(K key, V value) {
      //参数onlyIfAbsent为false
      return putVal(hash(key), key, value, false, true);
  }
  ```

  直接调用`putVal`方法

  ```java
  //参数onlyIfAbsent：true-表示在key不存在插入，key存在value为null时更新，false-不存在插入，存在则覆盖
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                 boolean evict) {
      //table数组
      Node<K,V>[] tab;
      //对应位置的Node节点
      Node<K,V> p; 
      //n-数组大小,i-对应table的位置
      int n, i;
      //如果table未初始化或者容量为0，则进行扩容
      if ((tab = table) == null || (n = tab.length) == 0)
          n = (tab = resize()).length;
      //如果要存放的位置为空，则直接创建Node节点即可
      if ((p = tab[i = (n - 1) & hash]) == null)
          tab[i] = newNode(hash, key, value, null);
      //否则，说明存在哈希冲突，采用拉链法解决
      else {
          //key在HashMap中对应的老节点，定义这个是为了返回旧value值
          Node<K,V> e; K k;
          //如果该位置的节点的hash一致且key一致，则代表就是我们要找的节点，直接赋值给e，后面会进行value的覆盖
          if (p.hash == hash &&
              ((k = p.key) == key || (key != null && key.equals(k))))
              e = p;
          //如果该位置是红黑树节点，则直接添加到树中
          else if (p instanceof TreeNode)
              //红黑树的插入操作，这里不深入研究
              e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
          //如果该位置是普通链表节点，则需要遍历查找比对
          else {
              for (int binCount = 0; ; ++binCount) {
                  //如p的后继为null，说明我们已经遍历到尾部了，说明不存在key，所以直接添加到末尾即可
                  if ((e = p.next) == null) {
                      p.next = newNode(hash, key, value, null);
                      //添加完后如果链表节点总数大于TREEIFY_THRESHOLD时，需要进行树化操作
                      if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                          //树化操作
                          treeifyBin(tab, hash);
                      //结束循环
                      break;
                  }
                  //如果key在map中存在，同样需要覆盖操作，这里先赋值给e
                  if (e.hash == hash &&
                      ((k = e.key) == key || (key != null && key.equals(k))))
                      break;
                  p = e;
              }
          }
          //如果e不为空，则代表key存在，则进行value的覆盖操作
          if (e != null) { // existing mapping for key
              V oldValue = e.value;
              //这里onlyIfAbsent控制是否覆盖旧的value，同时如果旧value为null时，也会覆盖
              if (!onlyIfAbsent || oldValue == null)
                  e.value = value;
              // 节点被访问的回调
              afterNodeAccess(e);
              //返回旧value
              return oldValue;
          }
      }
      //此时属于key不存在的情况
      //更新修改次数
      ++modCount;
      //如果容量超过阈值，扩容
      if (++size > threshold)
          resize();
      // 添加节点后的回调
      afterNodeInsertion(evict);
      //因为key不存在，返回空
      return null;
  }
  ```

  插入操作总体逻辑并不复杂，总结起来包括以下**三个步骤**

  * **1）当table未初始化或容量为0，进行初始化操作**
  * **2）查找要插入的key是否存在**
    * **如果存在**：根据条件判断是否需要覆盖旧值
    * **如果不存在**：则将键值对插入链表或红黑树中，如果插入是链表中，则根据链表长度决定是否需要树化操作
  * **3）判断键值对数量是否超过阈值，如果超过则进行扩容**

* **putIfAbsent(K, V)**

  **当 key 不存在的时候，添加 key-value 键值对；当key存在但是value为空时，更新旧值。**

  ```java
  public V putIfAbsent(K key, V value) {
      //参数onlyIfAbsent为true
      return putVal(hash(key), key, value, true, true);
  }
  ```

  逻辑同上，这里不再赘述。

## **2. 添加多个元素**

**putAll方法添加多个元素**

```java
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
```

调用`putMapEntries`方法，带集合的有参构造函数就是调用该方法，这里不再赘述。

# **九、删除操作**

* **remove(Object)**

  **移除指定key，返回对应的value，如果key不存在则返回null**

  ```java
  public V remove(Object key) {
      Node<K,V> e;
      //参数matchValue为false
      return (e = removeNode(hash(key), key, null, false, true)) == null ?
          null : e.value;
  }
  ```

  调用`removeNode`方法

  ```java
  //参数matchValue：true-键和值均匹配才删除；false-键匹配就删除
  final Node<K,V> removeNode(int hash, Object key, Object value,
                             boolean matchValue, boolean movable) {
      //table数组
      Node<K,V>[] tab; 
      //hash 对应 table 位置的 p 节点
      Node<K,V> p; 
      int n, index;
      //查找hash对应的table位置的节点
      if ((tab = table) != null && (n = tab.length) > 0 &&
          (p = tab[index = (n - 1) & hash]) != null) {
          //如果查找key对应的节点，会暂存给node
          Node<K,V> node = null, e; K k; V v;
          //判断key是否相同
          if (p.hash == hash &&
              ((k = p.key) == key || (key != null && key.equals(k))))
              node = p;
          //否则查找p的后继节点
          else if ((e = p.next) != null) {
              //如果e是红黑树节点，则直接去红黑树找
              if (p instanceof TreeNode)
                  //红黑树查找，这里不深入研究
                  node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
              //如果e是普通链表节点，则需要遍历查找
              else {
                  do {
                      if (e.hash == hash &&
                          ((k = e.key) == key ||
                           (key != null && key.equals(k)))) {
                          node = e;
                          break;
                      }
                      //注意这里p会保存找到节点的前驱节点
                      p = e;
                  } while ((e = e.next) != null);
              }
          }
          
          //如果node不为null，则表示找到了指定key的元素
          //matchValue控制是否需要匹配值
          if (node != null && (!matchValue || (v = node.value) == value ||
                               (value != null && value.equals(v)))) {
              //如果是红黑树类型，则直接在红黑树中删除
              if (node instanceof TreeNode)
                  //红黑树删除操作，这么不深入研究
                  ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
              //如果是链表的头结点，直接将该位置指向头结点的后继节点即可
              else if (node == p)
                  tab[index] = node.next;
              //如果是链表的中间或末尾位置，将该节点的前驱节点p的后继指该节点的后继即可
              else
                  p.next = node.next;
              //修改次数+1
              ++modCount;
              //键值对个数-1
              --size;
              //移除 Node 后的回调
              afterNodeRemoval(node);
              //返回被删除的节点
              return node;
          }
      }
      //查找不到直接返回 null
      return null;
  }
  ```

  删除逻辑相比较添加要简单一些，总结起来**分为三步**

  * **1）利用hash定位桶数组的位置**
  * **2）遍历链表或红黑树找到键相等的节点**
  * **3）根据条件决定是否删除该节点**

* **remove(Object, Object)**

  **根据键和值删除，返回是否删除到**

  ```java
  public boolean remove(Object key, Object value) {
      ////参数matchValue为true
      return removeNode(hash(key), key, value, true, true) != null;
  }
  ```

  逻辑同上，不再赘述。

# **十、查找操作**

* **get(Object)**

  **查找指定key，返回对应的value，如果key不存在则返回null**

  ```java
  public V get(Object key) {
      Node<K,V> e;
      return (e = getNode(hash(key), key)) == null ? null : e.value;
  }
  ```

  调用`getNode`方法

  ```java
  final Node<K,V> getNode(int hash, Object key) {
      Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
      //查找hash对应的table位置的节点
      if ((tab = table) != null && (n = tab.length) > 0 &&
          (first = tab[(n - 1) & hash]) != null) {
          //判断是否key匹配
          if (first.hash == hash && // always check first node
              ((k = first.key) == key || (key != null && key.equals(k))))
              return first;
          //如果不匹配，则查找first的后继节点
          if ((e = first.next) != null) {
              //如果是红黑树节点类型，则到红黑树中查找
              if (first instanceof TreeNode)
                  //红黑树查找操作，这里不深入研究
                  return ((TreeNode<K,V>)first).getTreeNode(hash, key);
              //如果是普通链表节点类型，遍历链表查找
              do {
                  if (e.hash == hash &&
                      ((k = e.key) == key || (key != null && key.equals(k))))
                      return e;
              } while ((e = e.next) != null);
          }
      }
      //如果找不到返回null
      return null;
  }
  ```

* **containsKey(Object)**

  **查询是否包含指定key**

  ```java
  public boolean containsKey(Object key) {
      return getNode(hash(key), key) != null;
  }
  ```

  实质就是调用`getNode`方法

* **containsValue(Object)**

  **查询是否包含指定value**

  ```java
  public boolean containsValue(Object value) {
      Node<K,V>[] tab; V v;
      if ((tab = table) != null && size > 0) {
          //遍历 table 数组
          for (int i = 0; i < tab.length; ++i) {
              //遍历链表或者红黑树节点
              for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                  if ((v = e.value) == value ||
                      (value != null && value.equals(v)))
                      return true;
              }
          }
      }
      return false;
  }
  ```

  **两次for循环，第一层遍历数组，第二层遍历链表或红黑树。实质就是遍历每一个节点判断**。

# **十一、其他操作**

## 1. 获取键、值、键值对

* **keySet()**

  **获取所有的key**

  ```java
  public Set<K> keySet() {
      Set<K> ks = keySet;
      if (ks == null) {
          ks = new KeySet();
          keySet = ks;
      }
      return ks;
  }
  ```

  **这里的keySet是HashMap继承的AbstractMap抽象类中的属性，保存所有的key**。如果`keySet`为空，则会new一个新的`KeySet`对象

  **values()**

  ```java
  public Collection<V> values() {
      Collection<V> vs = values;
      if (vs == null) {
          vs = new Values();
          values = vs;
      }
      return vs;
  }
  ```

  **values也是AbstractMap抽象类中的属性，保存所有的value**。如果`values`为空，则会new一个新的`Values`对象

* **entrySet()**

  **获取所有的键值对**

  ```java
  public Set<Map.Entry<K,V>> entrySet() {
      Set<Map.Entry<K,V>> es;
      return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
  }
  ```

  **entrySet是HashMap中的属性，保存所有的键值对**。如果entrySet为空，则会new一个EntrySet对象返回。

## 2. 清空

```java
public void clear() {
    Node<K,V>[] tab;
    //修改次数+1
    modCount++;
    if ((tab = table) != null && size > 0) {
        //重置size为0
        size = 0;
        //遍历table，将每一位置空
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

## 3. 遍历

* **1）键遍历**

  ```java
  for(Object key : map.keySet()) {
      // do something
  }
  ```

* **2）键值对遍历**

  ```java
  for(HashMap.Entry entry : map.entrySet()) {
      // do something
  }
  ```

* **3）迭代器遍历**

  ```java
  Iterator ite = map.keySet().iterator();
  while (ite.hasNext()) {
      Object key = ite.next();
      // do something
  }
  
  //或
  Iterator<Map.Entry> entries = map.entrySet().iterator();
  while (entries.hasNext()) {
      Map.Entry entry = (Map.Entry) entries.next();
      // do something
  }
  ```

* **4）Java8的Lambda表达式遍历**

  ```java
  map.forEach((k, v) -> {
      // do something
  });
  ```