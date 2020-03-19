# **一、概述**

> `LinkedList`是一种可以在任何位置进行高效地插入和移除操作的有序序列，它是基于双向链表实现的，是**线程不安全**的，允许元素为null的**双向链表**。

* `LinkedList`本质就是**双向链表**，每个节点都有指向前驱和指向后继的指针，从而构成链表。
* 相对于之前讲的`ArrayList`，`LinkedList` 使用较少，它的源码也相对简单许多。

# **二、类图**

![](http://img.xianzilei.cn/LinkedList%E7%B1%BB%E5%9B%BE.png)

可以看到`LinkedList`实现了**四个接口**

- `java.util.List`：提供数组的**添加、删除、修改、迭代遍历**等操作
- `java.util.Deque` ：提供**双端**队列的功能，`LinkedList` 支持快速的在头尾添加元素和读取元素，所以很容易实现该特性。
- `java.io.Serializable` ：表示 `LinkedList` 支持**序列化**的功能
- `java.lang.Cloneable` ：表示 `LinkedList` 支持**克隆**

`LinkedList`同时继承了一个**抽象类**

- `java.util.AbstractSequentialList`：它是`AbstractList`的子类，不过同`ArrayList`一样，`LinkedList`大量重写了其基继承的抽象类的方法。

# **三、属性和构造方法**

## **1. 属性**

**一共有三个属性**

```java
/**
 * 链表大小
 */
transient int size = 0;

/**
 * 头节点
 */
transient Node<E> first;

/**
 * 尾节点
 */
transient Node<E> last;
```

**其中Node类就是链表中的一个节点**，它是`LinkedList`的内部类

```java
private static class Node<E> {
    //节点元素
    E item;
    //后继节点
    Node<E> next;
    //前驱节点
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## **2. 构造方法**

一共有两个构造方法

* **1）无参构造方法**

  ```java
  /**
   * Constructs an empty list.
   */
  public LinkedList() {
  }
  ```

* **2）有参构造方法**

  ```java
  /**
       * Constructs a list containing the elements of the specified
       * collection, in the order they are returned by the collection's
       * iterator.
       *
       * @param  c the collection whose elements are to be placed into this list
       * @throws NullPointerException if the specified collection is null
       */
  public LinkedList(Collection<? extends E> c) {
      this();
      //添加所有集合元素到链表中，addAll方法后面讲解
      addAll(c);
  }
  ```

# **四、添加元素**

**添加元素，分成三个情况**

* **1）添加元素到队头**
* **2）添加元素到队尾**
* **3）添加元素到中间**

## **1. 添加单个元素**

* **add(E)**

  **插入单个元素，默认为尾插法。**

  ```java
  public boolean add(E e) {
      //尾插法添加元素
      linkLast(e);
      return true;
  }
  ```

  **内部调用linkLast方法，新增的元素在链表末尾**

  ```java
  /**
       * 尾插法
       */
  void linkLast(E e) {
      //记录原尾节点
      final Node<E> l = last;
      //创建新节点，它的前驱节点为原尾节点，后继节点默认为null
      final Node<E> newNode = new Node<>(l, e, null);
      //新插入的节点作为新的尾节点
      last = newNode;
      //如果原尾节点为null，说明链表是空的，因此头节点为新插入的节点
      if (l == null)
          first = newNode;
      //否则链表不为空，将原尾结点的后继指向新插入的节点
      else
          l.next = newNode;
      //更新链表大小
      size++;
      //更新链表修改次数
      modCount++;
  }
  ```

  **默认为尾插法，代码逻辑就是简答的双向链表尾插法逻辑，较为简单。**

* **add(int, E)**

  **指定位置插入元素**

  ```java
  public void add(int index, E element) {
      //检查位置是否合法，即index必须满足0<=index<=size，否则抛出数组越界异常
      checkPositionIndex(index);
  	//如果插入位置就是末尾或者链表一开始就是空的，直接调用尾插法函数
      if (index == size)
          linkLast(element);
      //否则调用linkBefore函数
      else
          //这里的node(index)就是找到index位置的元素
          linkBefore(element, node(index));
  }
  
  //这里就是遍历取出index位置的节点
  Node<E> node(int index) {
      // assert isElementIndex(index);
  	//这里判断index与size/2的大小，减少循环查找次数
      if (index < (size >> 1)) {
          Node<E> x = first;
          for (int i = 0; i < index; i++)
              x = x.next;
          return x;
      } else {
          Node<E> x = last;
          for (int i = size - 1; i > index; i--)
              x = x.prev;
          return x;
      }
  }
  ```

  接下来分析`linkBefore`函数，入参为**要插入的元素**和**index位置处的节点**

  ```java
  /**
       * Inserts element e before non-null Node succ.
       */
  void linkBefore(E e, Node<E> succ) {
      // assert succ != null;
      //保存原index位置的前驱节点
      final Node<E> pred = succ.prev;
      //创建新节点，前驱节点为index位置的前驱节点，后继节点为index位置的节点
      final Node<E> newNode = new Node<>(pred, e, succ);
      //原index位置的节点的前驱指向新节点
      succ.prev = newNode;
      //如果原index位置的节点前驱节点为空，说明index=0，即要插入位置为第一个位置，所以插入后新节点为头结点
      if (pred == null)
          first = newNode;
      //否则原index位置的后继节点指向新节点
      else
          pred.next = newNode;
      //更新链表大小
      size++;
      //更新链表修改次数
      modCount++;
  }
  ```

  指定位置的插入操作大家可以**画图分析**，仅看代码可能比较抽象。

* **addFirst(E)**

  **添加元素到链表的头部，即头插法**

  ```java
  /**
       * Inserts the specified element at the beginning of this list.
       *
       * @param e the element to add
       */
  public void addFirst(E e) {
      linkFirst(e);
  }
  ```

  **内部调用了linkFirst方法，新增的元素在链表头部**

  ```java
  private void linkFirst(E e) {
      //保存原头结点
      final Node<E> f = first;
      //创建新节点，前驱指向为空，后继指向为原头结点
      final Node<E> newNode = new Node<>(null, e, f);
      //新节点为头结点
      first = newNode;
      //原头结点为空表示链表原来为空，新插入的节点也是尾结点
      if (f == null)
          last = newNode;
      //否则原头结点的前驱指向新节点
      else
          f.prev = newNode;
      //更新链表大小
      size++;
      //更新链表修改次数
      modCount++;
  }
  ```

* **addLast(E)**

  **添加元素到链表的尾部，即尾插法**

  ```java
  public void addLast(E e) {
      //尾插法
      linkLast(e);
  }
  ```

  调用方法`linkLast`方法，**尾插法**，上面已经分析过了，这么不再赘述。

* **push(E)/offer(E)**

  因为`LinkedList`实现了`Queue`接口，所以它实现了**push(E)/offer(E)**方法，**添加元素到链表的头尾**

  ```java
  //头插法
  public void push(E e) {
      addFirst(e);
  }
  //尾插法
  public boolean offer(E e) {
      return add(e);
  }
  ```

  **代码逻辑上面已经分析过了，这么不再说明**。

## **2. 添加多个元素**

* **addAll(Collection<? extends E>)**

  **在尾部批量添加元素**

  ```java
  public boolean addAll(Collection<? extends E> c) {
      //在尾部批量插入元素
      return addAll(size, c);
  }
  ```

  可以看到它调用方法addAll(size, c)，正好是另一个添加多个元素的方法，我们下面分析。

* **addAll(int, Collection<? extends E>)**

  **指定位置添加元素**

  ```java
  public boolean addAll(int index, Collection<? extends E> c) {
      //检查index是否合法
      checkPositionIndex(index);
  	//将入参集合c转换成数组
      Object[] a = c.toArray();
      //待插入的数组长度
      int numNew = a.length;
      //如果数组为0，直接返回false代表当前链表未更改
      if (numNew == 0)
          return false;
  	//这里是获得第index位置的节点succ，和其前一个节点 pred
      Node<E> pred, succ;
      //如果index位置就是链表大小，表示插入队尾，所以pred为原尾节点，succ为null
      if (index == size) {
          succ = null;
          pred = last;
      } 
      //否则，pred为原index位置的节点的前驱，succ为原index位置的节点
      else {
          succ = node(index);
          pred = succ.prev;
      }
  	//遍历数组a，添加到pred的后面
      for (Object o : a) {
          @SuppressWarnings("unchecked") E e = (E) o;
          Node<E> newNode = new Node<>(pred, e, null);
          //如果pred为null ，说明当前位置为链表第一个位置，first为 null，则直接将first指向新节点
          if (pred == null)
              first = newNode;
          //否则将pred的后继指向新节点
          else
              pred.next = newNode;
          //修改pred指向，进行插入数据
          pred = newNode;
      }
  	//上述元素插入结束后，需要修改succ和pred 的指向
      //如果succ为空，根据上面的分析，说明插入位置就是队尾，所以pred就是尾节点
      if (succ == null) {
          last = pred;
      } 
      //否则，说明所有元素插入到原index位置的节点（succ）的前面，因此需要将pred的后继指向succ，succ的前驱指向pred
      else {
          pred.next = succ;
          succ.prev = pred;
      }
  	//更新链表大小
      size += numNew;
      //更新数组修改次数
      modCount++;
      //返回true，表示链接有更新
      return true;
  }
  ```

  如果不好理解可以自己**画图分析**。

# **五、删除元素**

* **remove(int)**

  **删除指定位置的元素，并返回原位置的元素**

  ```java
  public E remove(int index) {
      //校验index是否合法
      checkElementIndex(index);
      return unlink(node(index));
  }
  ```

  **再看unlink方法**

  ```java
  /**
   * Unlinks non-null node x.
   */
  E unlink(Node<E> x) {
      // assert x != null;
      // 保存x的元素，前驱和后继
      final E element = x.item;
      final Node<E> next = x.next;
      final Node<E> prev = x.prev;
      
      //更新前驱指向
  	//如果前驱为空，说明x是原头结点，删除后需要更新头结点指向
      if (prev == null) {
          first = next;
      } 
     	//否则就将x前驱节点的后继指向x的后继，x的前驱置空
      else {
          prev.next = next;
          x.prev = null;
      }
  
      //更新后继指向
      //如果x的后继为空，说明x是原尾节点，删除后需要更新尾结点指向
      if (next == null) {
          last = prev;
      } 
      //否则就将x的后继节点的前驱指向x的前驱，x的后继置空
      else {
          next.prev = prev;
          x.next = null;
      }
  	
      //删除x的元素值
      x.item = null;
      //更新链表大小
      size--;
      //更新链表更新次数
      modCount++;
      //返回原位置的元素值
      return element;
  }
  ```

  该方法的逻辑就是**双向链表的删除操作**。

* **remove(Object)**

  **移除首个对象，从头到尾的顺序执行**

  ```java
  public boolean remove(Object o) {
      //如果对象为空
      if (o == null) {
          for (Node<E> x = first; x != null; x = x.next) {
              if (x.item == null) {
                  unlink(x);
                  return true;
              }
          }
      } 
      //如果对象不为空
      else {
          for (Node<E> x = first; x != null; x = x.next) {
              if (o.equals(x.item)) {
                  unlink(x);
                  return true;
              }
          }
      }
      return false;
  }
  ```

  代码较为简单，**就是从头结点遍历到尾结点，删除第一个遇到的节点，并返回true，如果没有找到返回false**。

* **removeFirst()/removeLast()**

  **删除队首或队尾元素，并返回删除的元素。如果不存在，则抛出NoSuchElementException异常**

  ```java
  public E removeFirst() {
      //获取头结点
      final Node<E> f = first;
      //如果头结点为null，抛异常
      if (f == null)
          throw new NoSuchElementException();
      return unlinkFirst(f);
  }
  
  public E removeLast() {
      //获取尾结点
      final Node<E> l = last;
      //如果尾结点为null，抛异常
      if (l == null)
          throw new NoSuchElementException();
      return unlinkLast(l);
  }
  ```

  调用方法分别为`unlinkFirst`和`unlinkLast`，分别是**删除头结点和尾节点**的操作。

  ```java
  private E unlinkFirst(Node<E> f) {
      // assert f == first && f != null;
      //保存头结点元素值
      final E element = f.item;
      //保存头结点的后继节点
      final Node<E> next = f.next;
      //清除头结点信息
      f.item = null;
      f.next = null; // help GC
      //更新头结点
      first = next;
      //如果原头结点后无元素，则也要更新一下尾结点
      if (next == null)
          last = null;
      //否则正常更新新头结点的前驱为null
      else
          next.prev = null;
      //更新链表长度
      size--;
      //更新链表修改次数
      modCount++;
      //返回原头结点值
      return element;
  }
  
  private E unlinkLast(Node<E> l) {
      // assert l == last && l != null;
      //保存尾结点元素值
      final E element = l.item;
      //保存尾结点的前驱节点
      final Node<E> prev = l.prev;
      //清除尾结点信息
      l.item = null;
      l.prev = null; // help GC
      //更新尾结点
      last = prev;
      //如果原尾结点前无元素，则也要更新一下头结点
      if (prev == null)
          first = null;
      //否则正常更新新尾结点的后继为null
      else
          prev.next = null;
      //更新链表长度
      size--;
      //更新链表修改次数
      modCount++;
      //返回原尾结点值
      return element;
  }
  ```

* **removeFirstOccurrence(Object)/removeLastOccurrence(Object)**

  **删除链表首个或最后匹配的节点**

  ```java
  public boolean removeFirstOccurrence(Object o) {
      //调用remove(Object)方法，这里不再赘述
      return remove(o);
  }
  
  //原理同remove(Object)方法，只不过倒序遍历链表
  public boolean removeLastOccurrence(Object o) {
      if (o == null) {
          for (Node<E> x = last; x != null; x = x.prev) {
              if (x.item == null) {
                  unlink(x);
                  return true;
              }
          }
      } else {
          for (Node<E> x = last; x != null; x = x.prev) {
              if (o.equals(x.item)) {
                  unlink(x);
                  return true;
              }
          }
      }
      return false;
  }
  ```

* **remove()**

  **删除头结点**

  ```java
  public E remove() {
      //直接调用方法removeFirst()，这里不再赘述
      return removeFirst();
  }
  ```

# **六、查找元素**

* **indexOf(Object)**

  **获取首个指定元素的位置，如果不存则返回-1**

  ```java
  public int indexOf(Object o) {
      int index = 0;
      if (o == null) {
          for (Node<E> x = first; x != null; x = x.next) {
              if (x.item == null)
                  return index;
              index++;
          }
      } else {
          for (Node<E> x = first; x != null; x = x.next) {
              if (o.equals(x.item))
                  return index;
              index++;
          }
      }
      return -1;
  }
  ```

  逻辑较为简单，即**从队头遍历到队尾查找**。

* **lastIndexOf(Object)**

  **获取最后一个指定元素的位置，如果不存则返回-1**

  ```java
  public int lastIndexOf(Object o) {
      int index = size;
      if (o == null) {
          for (Node<E> x = last; x != null; x = x.prev) {
              index--;
              if (x.item == null)
                  return index;
          }
      } else {
          for (Node<E> x = last; x != null; x = x.prev) {
              index--;
              if (o.equals(x.item))
                  return index;
          }
      }
      return -1;
  }
  ```

  逻辑同`indexOf(Object)`方法，只不过是**从队尾遍历到队头**。

* **contains(Object)** 

  **查询是否包含指定元素**

  ```java
  public boolean contains(Object o) {
      return indexOf(o) != -1;
  }
  ```

  **本质就是调用indexOf方法查找**

* **get(int)**

  **查找指定位置的元素**

  ```java
  public E get(int index) {
      //校验index是否合法
      checkElementIndex(index);
      //调用node(index)方法查找指定位置的节点
      return node(index).item;
  }
  ```

* **peekFirst()/peekLast()**

  **因为LinkedList实现了Deque接口，所以它实现了peekFirst()和peekLast()方法，分别获得元素到链表的头和尾**

  ```java
  public E peekFirst() {
      final Node<E> f = first;
      return (f == null) ? null : f.item;
  }
  
  public E peekLast() {
      final Node<E> l = last;
      return (l == null) ? null : l.item;
  }
  ```

  **注意：这里如果头和尾节点不存在不好抛出异常，只是返回null**

* **peek()**

  **同peekFirst()方法**

  ```java
  public E peek() {
      final Node<E> f = first;
      return (f == null) ? null : f.item;
  }
  ```

* **element()**

  **获取头结点元素，不存在则抛出NoSuchElementException异常**

  ```java
  public E element() {
      return getFirst();
  }
  ```

# **七、其余操作**

* **set(int, E)**

  **修改指定位置的元素，并返回原值**

  ```java
  public E set(int index, E element) {
      //检查index是否合法
      checkElementIndex(index);
      //找到指定位置的节点
      Node<E> x = node(index);
      //保存原值
      E oldVal = x.item;
      //更新值
      x.item = element;
      //返回原值
      return oldVal;
  }
  ```

* **clear()**

  **清空链表**

  ```java
  public void clear() {
      //循环清除链表中每个节点
      for (Node<E> x = first; x != null; ) {
          Node<E> next = x.next;
          x.item = null;
          x.next = null;
          x.prev = null;
          x = next;
      }
      //清除链表头节点和尾结点
      first = last = null;
      //重置链表长度为0
      size = 0;
      //更新链表修改次数
      modCount++;
  }
  ```