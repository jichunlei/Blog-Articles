# **一、简介**

> `HashSet`是基于`HashMap`实现的Java集合，它可以保证存储的元素不重复

* `HashSet`的底层就是通过`HashMap`的**key来保存元素**，从而保证存储的元素**不重复**。
* 如果对`HashMap`熟悉的话，`HashSet`基本也就掌握了，它的底层源码相对简单多了。如果对`HashMap`不熟悉，可以参考我之前的文章：[JDK源码解析之HashMap](http://xianzilei.cn/blog/46)
* 基于`JDK1.8`版本

# **二、类图**

![](http://img.xianzilei.cn/HashSet%E7%B1%BB%E5%9B%BE.png)

**实现了三个接口**

* 实现 `java.util.Set`接口
* 实现 `java.io.Serializable` 接口。
* 实现 `java.lang.Cloneable`接口。

**继承了一个抽象类**

* 继承 `java.util.AbstractSet` 抽象类。

# **三、属性和构造方法**

## **1. 属性**

```java
//序列化版本号
static final long serialVersionUID = -5024744406713321676L;

//存放元素（存放在key上）
private transient HashMap<E,Object> map;

//因为HashSet不存放value，所以这里定义一个统一的对象作为value
private static final Object PRESENT = new Object();
```

属性主要是一个`HashMap`，`key`存放元素，`value`为统一的对象（可忽略）。

## **2. 构造方法**

`HashSet`一共有**5个构造方法**

* **HashSet()**

  ```java
  public HashSet() {
      //直接new一个HashMap
      map = new HashMap<>();
  }
  ```

* **HashSet(int initialCapacity)**

  ```java
  public HashSet(int initialCapacity) {
      //指定HashMap的初始容量
      map = new HashMap<>(initialCapacity);
  }
  ```

* **HashSet(int initialCapacity, float loadFactor)**

  ```java
  public HashSet(int initialCapacity, float loadFactor) {
      //指定HashMap的初始容量和加载因子
      map = new HashMap<>(initialCapacity, loadFactor);
  }
  ```

  

* **HashSet(int initialCapacity, float loadFactor, boolean dummy)**

  ```java
  HashSet(int initialCapacity, float loadFactor, boolean dummy) {
      map = new LinkedHashMap<>(initialCapacity, loadFactor);
  }
  ```

  该构造方法的访问权限仅仅为**包权限**，**不对外开放**，即根据指定的初始化容量和负载因子构造一个`LinkedHashMap`。`dummy`只是一个标识。该构造方法主要是为了支持`LinkedHashSet`，本文暂不研究它。

*  **HashSet(Collection<? extends E> c)**

  ```java
  public HashSet(Collection<? extends E> c) {
      //Math.max((int) (c.size()/.75f) + 1, 16)计算最小所需容量，最小为16，避免扩容
      map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
      //批量添加元素
      addAll(c);
  }
  ```

# **四、添加操作**

* **add(E e)**

  **添加单个元素，返回添加是否成功**

  ```java
  public boolean add(E e) {
      //直接调用HashMap的添加方法，默认value为PRESENT
      return map.put(e, PRESENT)==null;
  }
  ```

* **addAll(Collection<? extends E> c)**

  **批量添加元素，返回是否有元素添加成功**

  ```java
  public boolean addAll(Collection<? extends E> c) {
      boolean modified = false;
      for (E e : c)
          if (add(e))
              modified = true;
      return modified;
  }
  ```

  `addAll`方法在`AbstractCollection`中，里面调用`HashSet`重写的`add`方法。

# **五、移除操作**

**移除指定元素，返回是否移除成功**

```java
public boolean remove(Object o) {
    //直接调用HashMap的remove方法
    return map.remove(o)==PRESENT;
}
```

# **六、查找操作**

**判断指定元素是否存在**

```java
public boolean contains(Object o) {
    //直接调用HashMap的containsKey方法
    return map.containsKey(o);
}
```

# **七、其他操作**

* **clear()**

  **清空集合**

  ```java
  public void clear() {
      //因为HashSet底层就是HashMap，所以清空HashSet就是清空HashMap
      map.clear();
  }
  ```