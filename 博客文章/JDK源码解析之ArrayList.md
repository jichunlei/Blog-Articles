# **一、简介**

> ArrayList就是动态数组，用MSDN中的说法，就是Array的复杂版本，它提供了动态的增加和减少元素，实现了ICollection和IList接口，灵活的设置数组的大小等好处

它的底层就是**数组队列**，相对于Java中的数组来说，它的容量可以动态增长，因此被称为**动态数组**。正因为**自动扩容机制**，`ArrayList`已经成为平时最常用的集合类（以下的讲解基于jdk1.8版本）。

# **二、类图**

**ArrayList的类图如下所示**

![](http://img.xianzilei.cn/ArrayList%E7%B1%BB%E5%9B%BE.png)

可以看到`ArrayList`实现了**四个接口**

* `java.util.List`：提供数组的**添加、删除、修改、迭代遍历**等操作
* `java.util.RandomAccess` ：表示 `ArrayList` 支持**快速的随机访问**
* `java.io.Serializable` ：表示 `ArrayList` 支持**序列化**的功能
* `java.lang.Cloneable` ：表示 `ArrayList` 支持**克隆**

`ArrayList`同时继承了一个**抽象类**

* `java.util.AbstractList`：**提供了List接口的骨架实现，大幅度的减少了实现迭代遍历相关操作的代码**（实际上`ArrayList`大量重写了`AbstractList`的提供的方法，所以，`AbstractList` 对于 `ArrayList` 意义不大，其内部方法更多用于 `AbstractList` 其它子类）。

# **三、属性和构造方法**

## **1. 属性**

```java
    /**
     * 序列化版本号
     */
	private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组（用于空实例）
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用于默认大小空实例的共享空数组实例
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 保存ArrayList数据的数组
     */
    transient Object[] elementData;

    /**
     * ArrayList 所包含的元素个数
     */
    private int size;

	/**
     * 最大数组容量
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

其中核心的两个属性

* `elementData`：存放元素的数组，可以动态扩容。
* `size`：元素数量，这里指的数组中被使用的元素的个数

## **2. 构造方法**

`ArrayList`一共有**三个构造方法**

* **1）无参构造方法**

  ```java
  /**
       * Constructs an empty list with an initial capacity of ten.
       */
  public ArrayList() {
      this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
  }
  ```

  初始化的时候默认为`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`这个空数组，即初始容量为0，只有**首次添加元素时才真正初始化为容量10**，这可能是考虑**节省内存**。

  注意：为啥不使用直接使用`EMPTY_ELEMENTDATA` ？这是因为`EMPTY_ELEMENTDATA` 按照1.5倍扩容而非从10开始，即起点不同，下面的扩容机制会详细说明。

* **2）带初始容量的有参构造方法**

  ```java
  /**
       * Constructs an empty list with the specified initial capacity.
       *
       * @param  initialCapacity  the initial capacity of the list
       * @throws IllegalArgumentException if the specified initial capacity
       *         is negative
       */
  public ArrayList(int initialCapacity) {
      //初始容量大于0时，创建指定大小的Object数组
      if (initialCapacity > 0) {
          this.elementData = new Object[initialCapacity];
      }
      //初始容量等于0时，直接引用到EMPTY_ELEMENTDATA属性
      else if (initialCapacity == 0) {
          this.elementData = EMPTY_ELEMENTDATA;
      } 
      //初始容量小于0时，抛出IllegalArgumentException异常
      else {
          throw new IllegalArgumentException("Illegal Capacity: "+
                                             initialCapacity);
      }
  }
  ```

  **注意**：初始容量指定为0时，使用`EMPTY_ELEMENTDATA`空数组，而无参构造方法时使用`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`共享空数组，后面的扩容机制会讲解二者的不同。

* **3）带集合的有参构造方法**

  ```java
  /**
       * Constructs a list containing the elements of the specified
       * collection, in the order they are returned by the collection's
       * iterator.
       *
       * @param c the collection whose elements are to be placed into this list
       * @throws NullPointerException if the specified collection is null
       */
  public ArrayList(Collection<? extends E> c) {
      //将传进来的集合c转换成Object数组并赋值给elementData
      elementData = c.toArray();
      //如果数组长度大于0，即数组不为空
      if ((size = elementData.length) != 0) {
          // c.toArray might (incorrectly) not return Object[] (see 6260652)
          // 如果集合元素不是 Object[] 类型，则会创建新的 Object[] 数组，并将 elementData 赋值到其中，最后赋值给 elementData
          if (elementData.getClass() != Object[].class)
              elementData = Arrays.copyOf(elementData, size, Object[].class);
      } 
      //如果数组为空，则引用指向EMPTY_ELEMENTDATA，类似带初始容量为0的有参构造方法
      else {
          // replace with empty array.
          this.elementData = EMPTY_ELEMENTDATA;
      }
  }
  ```

  该方法不常用。

# **四、扩容机制**

`ArrayList`提供两种扩容机制，一种**自动扩容**，用户无法显示调用；另一种**手动扩容**，即提供给用户直接调用的。

## **1. 自动扩容**

自动扩容主要发生在添加元素的方法中，下面以add方法为例一步一步剖析扩容

### **1.1 先看add方法**

```java
/**
     * 添加指定元素到列表末尾
     */
public boolean add(E e) {
    //添加元素前先调用ensureCapacityInternal方法
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //为数组赋值，很简单的操作
    elementData[size++] = e;
    return true;
}
```

可以看到添加的时候预先需要进行扩容判断，扩容结束后直接添加元素到数组末尾即可。可以看到**ensureCapacityInternal()方法即为我们想要的扩容逻辑**。

### **1.2 ensureCapacityInternal()**

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//计算最小扩容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果elementData引用指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA属性，即默认大小空实例的共享空数组实例，也就是无参构造函数初始化的空实例，会取指定大小和默认大小（10）的较大值。也就是无参构造方法那一块所说的，初始时刻数组大小为空，首次添加元素时会初始化为默认大小（10）
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

**这里的计算最小扩容量实质就是判断是否是无参构造函数首次添加元素，如果是则初始化为默认大小10，否则不调整指定的扩容量**。

### **1.3 ensureExplicitCapacity()**

```java
private void ensureExplicitCapacity(int minCapacity) {
    //修改的次数自增
    modCount++;

    //如果需要扩容的大小比当前数组大小大，则进行扩容
    //这里分析一下为啥要加这个判断。无参构造函数首次添加元素的时候，elementData的大小还是0，根据上面的分析，首次新增结束后数组长度会变为10，这样第2次、第3次...到第10次都不用去扩容，所以这里加了个判断。
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

**啰嗦了这么久，这里终于到了我们想要的真正的扩容方法——grow()，接下来重点分析该方法的扩容逻辑**

### **1.4 grow()** 

```java
/**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
private void grow(int minCapacity) {
    // overflow-conscious code
    //数组旧容量
    int oldCapacity = elementData.length;
    //准新容量=旧容量的1.5倍（这里使用移位计算提供计算速度）
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果准新容量比最小需要容量还要小，则最小需要容量当做准新容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //如果准新容量大于最大的数组大小，需要进一步进行计算（后面讲解）
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**扩容的主要逻辑是先计算预计的新容量（原来的1.5倍大小），然后根据最小所需容量和最大数组大小进行调整。**

### **1.5 hugeCapacity()**

```java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
    MAX_ARRAY_SIZE;
}
```

在grow方法中我们知道，如果准新容量大于最大的数组大小（**MAX_ARRAY_SIZE=Integer.MAX_VALUE - 8**），说明该准新容量不满足要求，我们需拿最小所需容量来做为判断标准

* **如果最小所需容量（minCapacity）大于最大的数组大小（MAX_ARRAY_SIZE），则新容量为Integer.MAX_VALUE。**
* **否则新容量为最大的数组大小（MAX_ARRAY_SIZE）**

## **2. 手动扩容**

**手动扩容提供的方法是ensureCapacity，该方法ArrayList内部没有调用过，且被public修饰，所以很显然是提供给用户调用的**

```java
/**
     * Increases the capacity of this <tt>ArrayList</tt> instance, if
     * necessary, to ensure that it can hold at least the number of elements
     * specified by the minimum capacity argument.
     *
     * @param   minCapacity   the desired minimum capacity
     */
public void ensureCapacity(int minCapacity) {
    //如果是无参构造函数初始化状态，则设置minExpand为默认大小10，否则为0
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```

该方法是显示指定用户想要扩容后的数组长度，一般用于需要新增大量元素之前使用，这样可以减少扩容的次数，提供效率。比如预测需要新增10000000条数据，为了减少扩容的性能损耗

```java
ArrayList<Object> list = new ArrayList<Object>();
//预先扩容
list.ensureCapacity(10000000);
```

# **五、添加操作**

**添加操作有四种**

## **1. add(E)**

**添加单个元素到数组队列末尾**

```java
/**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
public boolean add(E e) {
    //先扩容判断
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //添加元素到队列末尾，同时size++
    elementData[size++] = e;
    //返回添加成功
    return true;
}
```

**逻辑较为简单**

## **2. add(int, E)**

**添加单个元素插入到指定位置处**

```java
public void add(int index, E element) {
    //校验位置是否在数组范围内
    rangeCheckForAdd(index);
	//扩容操作
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //将index + 1 位置开始的元素，进行往后挪
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //在指定位置放入指定元素
    elementData[index] = element;
    //队列长度+1
    size++;
}

// 校验位置是否在数组范围内，如果不在则抛数组越界异常
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

这里有个`System.arraycopy()`方法需要说明一下，该方法就是将**源数组某段元素复制到目标数组上去**。

```java
//System类中native方法
//第一个参数src：源数组对象
//第二个参数srcPos：源数组中的起始位置
//第三个参数dest：目标数组对象
//第四个参数destPos：目标数组中的起始位置
//第五个参数length：要复制的数组元素的数量
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

## **3. addAll(Collection<? extends E>)**

**添加多个元素到数组队列的末尾**

```java
/**
     * Appends all of the elements in the specified collection to the end of
     * this list, in the order that they are returned by the
     * specified collection's Iterator.  The behavior of this operation is
     * undefined if the specified collection is modified while the operation
     * is in progress.  (This implies that the behavior of this call is
     * undefined if the specified collection is this list, and this
     * list is nonempty.)
     *
     * @param c collection containing elements to be added to this list
     * @return <tt>true</tt> if this list changed as a result of the call
     * @throws NullPointerException if the specified collection is null
     */
public boolean addAll(Collection<? extends E> c) {
    //将集合转化为数组
    Object[] a = c.toArray();
    //要添加的元素的数量
    int numNew = a.length;
    //扩容
    ensureCapacityInternal(size + numNew);  // Increments modCount
    //将这些元素复制到原数组的末尾
    System.arraycopy(a, 0, elementData, size, numNew);
    //更新队列长度
    size += numNew;
    return numNew != 0;
}
```

## **4. addAll(int, Collection<? extends E>)**

**添加多个元素到数组队列的指定位置**

```java
/**
     * Inserts all of the elements in the specified collection into this
     * list, starting at the specified position.  Shifts the element
     * currently at that position (if any) and any subsequent elements to
     * the right (increases their indices).  The new elements will appear
     * in the list in the order that they are returned by the
     * specified collection's iterator.
     *
     * @param index index at which to insert the first element from the
     *              specified collection
     * @param c collection containing elements to be added to this list
     * @return <tt>true</tt> if this list changed as a result of the call
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * @throws NullPointerException if the specified collection is null
     */
public boolean addAll(int index, Collection<? extends E> c) {
    //校验位置是否在数组范围内
    rangeCheckForAdd(index);
	//将集合转化为数组
    Object[] a = c.toArray();
    //要添加的元素的数量
    int numNew = a.length;
    //扩容
    ensureCapacityInternal(size + numNew);  // Increments modCount
	//由于是指定位置，所以需要先移动位置为要添加的元素腾出位置
    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);
	//将要添加的元素复制到指定位置处
    System.arraycopy(a, 0, elementData, index, numNew);
    //更新队列大小
    size += numNew;
    return numNew != 0;
}
```

# **六、删除操作**

**删除操作有四种**

## **1. remove(int)**

**移除指定位置的元素，并返回该位置的原元素**

```java
/**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
public E remove(int index) {
    //校验index是否合法
    rangeCheck(index);
	//修正次数+1
    modCount++;
    //获取指定位置的元素
    E oldValue = elementData(index);
	
    int numMoved = size - index - 1;
    //如果删除位置不是最末尾的话，需要将删除位置后面元素前移一位
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //将新的末尾置为 null ，帮助GC及时回收
    elementData[--size] = null; // clear to let GC do its work
	//返回原值
    return oldValue;
}

//校验index是否合法位置是否存在，不存在则抛数组越界异常
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
//获取指定位置的元素
E elementData(int index) {
    return (E) elementData[index];
}
```

## **2. remove(Object)**

**移除首个指定值的元素，并返回是否移除到**

```java
/**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns <tt>true</tt> if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return <tt>true</tt> if this list contained the specified element
     */
public boolean remove(Object o) {
    //如果对象为null
    if (o == null) {
        //循环找到第一个为null的对象删除
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } 
    //如果对象不为空
    else {
        //循环找到第一个equals匹配的对象删除
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

//快速删除逻辑，本质就是根据索引位置删除
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

## **3. removeRange(int, int)**

**批量删除某一范围内的元素，其中左边右开区间，即不包括最右边元素。**

```java
/**
     * Removes from this list all of the elements whose index is between
     * {@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
     * Shifts any succeeding elements to the left (reduces their index).
     * This call shortens the list by {@code (toIndex - fromIndex)} elements.
     * (If {@code toIndex==fromIndex}, this operation has no effect.)
     *
     * @throws IndexOutOfBoundsException if {@code fromIndex} or
     *         {@code toIndex} is out of range
     *         ({@code fromIndex < 0 ||
     *          fromIndex >= size() ||
     *          toIndex > size() ||
     *          toIndex < fromIndex})
     */
protected void removeRange(int fromIndex, int toIndex) {
    //数组修改次数+1
    modCount++;
    
    int numMoved = size - toIndex;
    //将toIndex及其后面的元素往前移动，通过覆盖方式删除元素
    System.arraycopy(elementData, toIndex, elementData, fromIndex,
                     numMoved);

    // clear to let GC do its work
    //计算新数组队列元素个数
    int newSize = size - (toIndex-fromIndex);
    //将末尾遗留的元素置为null，便于gc回收
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    //更新数组队列元素个数
    size = newSize;
}
```

## **4. removeAll(Collection<?>)**

**批量移除指定的多个元素，并返回是否移除到**

```java
/**
     * Removes from this list all of its elements that are contained in the
     * specified collection.
     *
     * @param c collection containing elements to be removed from this list
     * @return {@code true} if this list changed as a result of the call
     * @throws ClassCastException if the class of an element of this list
     *         is incompatible with the specified collection
     * (<a href="Collection.html#optional-restrictions">optional</a>)
     * @throws NullPointerException if this list contains a null element and the
     *         specified collection does not permit null elements
     * (<a href="Collection.html#optional-restrictions">optional</a>),
     *         or if the specified collection is null
     * @see Collection#contains(Object)
     */
public boolean removeAll(Collection<?> c) {
    //空判断，如果c为null，抛出空指针异常
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```

可以看到**核心逻辑是batchRemove方法**，如下所示

```java
//该方法有两个地方使用的。
//complement=false：removeAll(Collection<?>)方法使用到，用于批量删除指定集合元素
//complement=true：retainAll(Collection<?>)方法使用到，用于获取当前集合和指定集合的交集
private boolean batchRemove(Collection<?> c, boolean complement) {
    //当前队列数组
    final Object[] elementData = this.elementData;
    //r做循环控制，w表示集合c与当前集合的不重复的元素个数
    int r = 0, w = 0;
    boolean modified = false;
    try {
        //这个循环就是将c和elementData不重复的元素规整到elementData中，w表示结束位置
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        //因为c集合的contains是各个子类实现的，不可控，可能会发生异常，所以在发生异常的时候，将剩余未比对完的元素统一作为不重复元素放到elementData中
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        //如果w不等于size，说明c和elementData存在交集，所以需要将w及后面位置的数据清除，同时modified设置为true
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

# **七、查找操作**

**查找的方式较多，以下列出主要的查找方法**

* **get(int)** 

  **获取指定位置元素**

  ```java
  public E get(int index) {
      //检查索引是否合法（index必须小于size），否则抛出数组越界异常
      rangeCheck(index);
  	//返回指定位置的元素
      return elementData(index);
  }
  
  //查找指定位置的元素
  E elementData(int index) {
      return (E) elementData[index];
  }
  ```

* **indexOf(Object)**

  **查找首个为指定元素的位置，如果不存在返回-1**

  ```java
  //逻辑很简单，就是循环遍历查找，因此效率较低
  public int indexOf(Object o) {
      if (o == null) {
          for (int i = 0; i < size; i++)
              if (elementData[i]==null)
                  return i;
      } else {
          for (int i = 0; i < size; i++)
              if (o.equals(elementData[i]))
                  return i;
      }
      return -1;
  }
  ```

* **lastIndexOf(Object)**

  **查找最后一个为指定元素的位置，不存在则返回-1**

  ```java
  //实现同indexOf方法，只不过从数组队列末尾开始遍历
  public int lastIndexOf(Object o) {
      if (o == null) {
          for (int i = size-1; i >= 0; i--)
              if (elementData[i]==null)
                  return i;
      } else {
          for (int i = size-1; i >= 0; i--)
              if (o.equals(elementData[i]))
                  return i;
      }
      return -1;
  }
  ```

* **contains(Object)**

  **集合是否包含指定元素**

  ```java
  //提供调用indexOf方法来实现
  public boolean contains(Object o) {
      return indexOf(o) >= 0;
  }
  ```

# **八、其余操作**

除了上面几个重点的操作外，下面列举其余一些较为重要的方法。

* **set(int, E)**

  **设置指定位置的元素，返回该位置的原元素**

  ```java
  public E set(int index, E element) {
      //检查索引是否合法（index必须小于size），否则抛出数组越界异常
      rangeCheck(index);
  	//获取当前位置的原元素，作为后续的返回
      E oldValue = elementData(index);
      //修改指定位置元素为指定元素
      elementData[index] = element;
      //返回原元素
      return oldValue;
  }
  ```

* **clear()**

  **清空数组**

  ```java
  public void clear() {
  	//更新数组修改值
      modCount++;
  
      // 遍历数组，设置为 null
      for (int i = 0; i < size; i++)
          elementData[i] = null;
  	//数组队列元素长度置0
      size = 0;
  }
  ```

* **subList(int, int)**

  **创建子数组**

  ```java
  public List<E> subList(int fromIndex, int toIndex) {
      //索引校验
      subListRangeCheck(fromIndex, toIndex, size);
      //创建子数组
      return new SubList(this, 0, fromIndex, toIndex);
  }
  
  //索引校验
  static void subListRangeCheck(int fromIndex, int toIndex, int size) {
      if (fromIndex < 0)
          throw new IndexOutOfBoundsException("fromIndex = " + fromIndex);
      if (toIndex > size)
          throw new IndexOutOfBoundsException("toIndex = " + toIndex);
      if (fromIndex > toIndex)
          throw new IllegalArgumentException("fromIndex(" + fromIndex +
                                             ") > toIndex(" + toIndex + ")");
  }
  ```

  这里的**SubList是ArrayList的一个内部类**，下面是`SubList`部分代码（属性和构造方法）

  ```java
  private class SubList extends AbstractList<E> implements RandomAccess {
      private final AbstractList<E> parent;
      private final int parentOffset;
      private final int offset;
      int size;
  
      SubList(AbstractList<E> parent,
              int offset, int fromIndex, int toIndex) {
          this.parent = parent;
          this.parentOffset = fromIndex;
          this.offset = offset + fromIndex;
          this.size = toIndex - fromIndex;
          this.modCount = ArrayList.this.modCount;
      }
  }
  ```

  **由源码可以看到，SubList 和根数组 root 共享相同的 elementData 数组，只是限制了 fromIndex, toIndex 的范围**