# **一、简介**

> final在Java中是一个保留的关键字，可以声明成员变量、方法、类以及本地变量。一旦你将引用声明作final，你将**不能改变这个引用**了，编译器会检查代码，如果你试图将变量再次初始化的话，编译器会报编译错误。

# **二、使用场景**

在`Java`中，`final`关键字可以用来修饰**类**、**方法**和**变量（包括成员变量和局部变量）**。

## **1. 修饰类**

 当用`final`修饰一个类时，表明这个类**不能被继承**。最常见是就是String类，任何类都无法继承它。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence 
```

**如果一个类你永远不会让他被继承**（子类继承往往可以重写父类的方法和改变父类属性，会带来一定的安全隐患），就可以用**final进行修饰**。注意，**final类中的成员变量可以根据需要设置为final，但是它的所有成员方法会被隐式地指定为final方法**。在使用`final`修饰类的时候，要注意谨慎选择，除非这个类真的在以后不会用来继承或者出于安全的考虑，**尽量不要将类设计为final类**。

## **2. 修饰方法**

当父类的方法被`final`修饰的时候，**子类不能重写父类的该方法**，比如在`Object`中，`getClass()`方法就是`final`的，我们就不能重写该方法。

```java
public class Object {
    public final native Class<?> getClass();
}
```

如果想**禁止该方法在子类中被重写**的，可以设置该方法为为`final`。注意：**因为重写的前提是子类可以从父类中继承此方法，如果父类中final修饰的方法同时访问控制权限为private，将会导致子类中不能直接继承到此方法，此时子类中就可以定义相同的方法名和参数**。

## **3. 修饰变量**

**final修饰变量表示这个变量一旦赋值就不能修改**。需要注意，当`final`修饰一个基本数据类型时，表示该**基本数据类型的值一旦在初始化后便不能发生变化**；如果`final`修饰一个引用类型时，则在对其初始化之后便不能再让其指向其他对象了，但该**引用所指向的对象的内容是可以发生变化**的，`final`只保证这个**引用类型变量所引用的地址不会发生改变，即一直引用这个对象，但这个对象属性是可以改变的**。

### **3.1 成员变量**

Java中，成员变量分为**类变量（static修饰）**和**实例变量**。针对这两种类型的变量赋初值的时机是不同的，类变量可以在**声明变量的时候直接赋初值**或者**在静态代码块中给类变量赋初值**。而实例变量可以**在声明变量的时候给实例变量赋初值**，**在非静态初始化块中**以及**构造器中赋初值**。因此类变量有**两个时机赋初值**，而实例变量则可以有**三个时机赋初值**。被final修饰的变量必须在上述时机赋初值，否则编译器会报错。总结一下

* **final修饰的类变量**：必须要在**静态初始化块**中指定初始值或者**声明该类变量时**指定初始值，而且只能在这**两个地方之一**进行指定，一旦赋值后不能再修改。
* **final修饰的实例变量**：必要要在**非静态初始化块**，**声明该实例变量**或者在**构造器中**指定初始值，而且只能在这**三个地方之一**进行指定，一旦赋值后不能再修改。

### **3.2 局部变量**

`final`局部变量由程序员进行**显式初始化**，如果`final`局部变量已经进行了初始化则后面就不能再次进行更改，如果`final`变量未进行初始化，可以进行赋值，**当且仅有一次**赋值，一旦赋值之后再次赋值就会出错。

## **4. 宏变量与宏替换**

### **4.1 宏变量**

> 如果一个变量满足一下三个条件时，该变量就会成为一个**宏变量**，**即是一个常量**
>
> * 1）被final修饰符修饰
> * 2）在定义该final变量时就指定了初始值
> * 3）该初始值在编译时就能够唯一指定

例如：

```java
final String a = "hello";
final String b = a;
final String c = getHello();
```

**变量a被final修饰，且初始化的时候就声明了初始值，且该初始值在编译的时候就可以唯一指定，即”hello“，所以a就是一个宏变量。而变量b和c虽然满足一、二条件，但是初始值在编译期间无法唯一确定，所以b和c不是**。

### **4.2 宏替换**

> 如果一个变量是**宏变量**，那么编译器会把**程序所有用到该变量的地方直接替换成该变量的值**，这就是**宏替换**

例如（借助网上一个有意思的面试题）：

```java
public static void main(String[] args) {
    String hw = "hello world";

    String hello = "hello";
    final String finalWorld2 = "hello";//宏变量，值为hello
    final String finalWorld3 = hello;
    final String finalWorld4 = "he" + "llo";//宏变量，值为hello

    String hw1 = hello + " world";
    String hw2 = finalWorld2 + " world";//相当于String hw2 = "hello" + " world";也就相当于String hw2="hello world";
    String hw3 = finalWorld3 + " world";
    String hw4 = finalWorld4 + " world";//相当于String hw4 = "hello" + " world";也就相当于String hw2="hello world";

    System.out.println(hw == hw1); //false
    System.out.println(hw == hw2); //true
    System.out.println(hw == hw3); //false
    System.out.println(hw == hw4); //true
}
```

**根据上面对宏变量的分析，我们知道finalWorld2和finalWorld4是属于宏变量，因此后续程序会直接使用其值“hello”来代替finalWorld2和finalWorld4。因此hw2和hw4等同于"hello world"，所以hw=hw2=hw4**。

# **三、final域重排序规则**

一二章介绍的只是final关键字的**基础用法**。然而在多线程的层面，final也有其自己的**内存语义**。**主要体现在final域的重排序上**，下面我们来介绍final的重排序规则（**下面分析来源于《Java并发编程的艺术》一书中**）。

## **1. final域为基本类型**

```java
public class FinalExample {
    int i; // 普通变量
    final int j; // final变量
    static FinalExample obj;

    public FinalExample() { // 构造函数
        i = 1; // 写普通域
        j = 2; // 写final域
    }

    public static void writer() { // 写线程A执行
        obj = new FinalExample();
    }

    public static void reader() { // 读线程B执行
        FinalExample object = obj; // 读对象引用
        int a = object.i; // 读普通域
        int b = object.j; // 读final域
    }
}
```

这里假设一个线程A执行`writer()`方法，随后另一个线程B执行`reader()`方法。

### **1.1 写final域的重排序规则**

> 写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面
>
> 1）JMM**禁止编译器把final域的写重排序到构造函数之外**。
>
> 2）编译器会在`final`域的写之后，构造函数`return`之前，插入一个`StoreStore`屏障。这个屏障
> **禁止处理器把final域的写重排序到构造函数之外**。

上面代码的`writer()`方法只有一行，它的执行包含两步

* **构造一个FinalExample类型的对象**
* **将该对象赋值给引用变量obj**

根据写final域的重排序规则，假设线程A先执行，线程B后执行，下面是可能的一种执行时序图

![](http://img.xianzilei.cn/final%E7%94%A8%E4%BE%8B%E5%9B%BE-01.png)

`i`和`j`变量没有数据依赖性，写普通域`i=1`可能会重排序到构造函数外，此时线程B错误地读取到未赋值的`i`。而写`final`域`j=2`的操作，被写`final`域的重排序规则限制在了构造函数之内，因此线程B读取的`j`为正确的值。

**因此，写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障**。 

### **1.2 读final域的重排序规则**

> 读`final`域的重排序规则是，**在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）**。编译器会在读`final`域操作的前面插入一个`LoadLoad`屏障。

初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也会遵守间接依赖，也不会重排序这两个操作。但有少数处理器允许对存在间接依赖关系的操作做重排序（比如alpha处理器），这个规则就是专门用来针对这种处理器的 。

上面代码的`reader()`方法包含**三步**

* **初次读引用变量obj**
* **初次读引用变量obj指向对象的普通域j**
* **初次读引用变量obj指向对象的final域i**

假设线程A没有重排序，同时程序在不遵守间接依赖的处理器上执行，下面是一种可能的执行时序

![](http://img.xianzilei.cn/final%E7%94%A8%E4%BE%8B%E5%9B%BE-02.png)

如上图所示，读对象的普通域的操作被处理器重排序到读对象引用之前。读普通域时，该域还没有被写线程A写入，这是一个错误的读取操作。而读final域的重排序规则会把读对象final域的操作限定在读对象引用之后，此时该final域已经被A线程初始化过了，这是一个正确的读取操作。

**因此，读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用**。

## **2. final域为引用类型**

```java
public class FinalReferenceExample {
    final int[] intArray; // final是引用类型
    static FinalReferenceExample obj;
    public FinalReferenceExample () { // 构造函数
        intArray = new int[1]; // 1
        intArray[0] = 1; // 2
    }
    public static void writerOne () { // 写线程A执行
        obj = new FinalReferenceExample (); // 3
    }
    public static void writerTwo () { // 写线程B执行
        obj.intArray[0] = 2; // 4
    }
    public static void reader () { // 读线程C执行
        if (obj != null) { // 5
            int temp1 = obj.intArray[0]; // 6
        }
    }
}
```

本例`final`域为一个引用类型，它引用一个`int`型的数组对象。

### **2.1 写final域的重排序规则**

> 对于引用类型，写final域的重排序规则对编译器和处理器**增加了**如下约束：**在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序**。

对于上面的代码，假设首先线程A执行`writerOne()`方法，执行完后线程B执行`writerTwo()`方法，执行完后线程C执行`reader()`方法 ，下面为一种可能的执行时序

![](http://img.xianzilei.cn/final%E7%94%A8%E4%BE%8B%E5%9B%BE-03.png)

如上图所示，1是对final域的写入，2是对这个final域引用的对象的成员域的写入，3是把被构造的对象的引用赋值给某个引用变量。根据前面的分析我们知道1不能和3重排序外，而对于引用类型**2和3也不能重排序**。

### **2.2 读final域的重排序规则**

JMM可以**确保读线程C至少能看到写线程A在构造函数中对final引用对象的成员域的写入**。即C至少能看到数组下标0的值为1。而写线程B对数组元素的写入，读线程C可能看得到，也可能看不到。JMM不保证线程B的写入对读线程C可见，因为写线程B和读线程C之间存在数据竞争，此时的执行结果不可预知。

因此，对于引用类型，在原来规则的基本上，额外增加约束：禁止在构造函数对**一个final修饰的对象的成员域的写入**与随后将**这个被构造的对象的引用赋值给引用变量**重排序。

## 3. 补充说明

前面我们提到过，写final域的重排序规则可以确保：在引用变量为任意线程可见之前，该引用变量指向的对象的final域已经在构造函数中被正确初始化过了。其实，要得到这个效果，还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程所见，也就是对象引用不能在构造函数中“逸出”。例如

```java
public class FinalReferenceEscapeExample {
    final int i;
    static FinalReferenceEscapeExample obj;
    public FinalReferenceEscapeExample () {
        i = 1; // 1写final域
        obj = this; // 2 this引用在此"逸出"
    }
    public static void writer() {
        new FinalReferenceEscapeExample ();
    }
    public static void reader() {
        if (obj != null) { // 3
            int temp = obj.i; // 4
        }
    }
}
```

线程A执行`writer()`方法，线程B执行`reader()`方法。下面是可能的执行时序图

![](http://img.xianzilei.cn/final%E7%94%A8%E4%BE%8B%E5%9B%BE-04.png)

因为构造函数中操作1和2之间没有数据依赖性，1和2可以重排序，先执行了2，这个时候this引用还是个没有完全初始化的对象，而当线程B去读取该对象时就会出错。尽管依然满足了final域写重排序规则：在引用对象对所有线程可见时，其final域已经完全初始化成功。但是，引用对象“this”逸出，该代码依然存在线程安全的问题。

# **四、实现原理**

final语义的实现原理就是上面所说到的使用**内存屏障**（如果不清楚内存屏障的小伙伴可以查看之前的文章：[深入理解volatile底层原理](http://xianzilei.cn/blog/60)）。

以x86处理器为例（下面内容来源于《Java并发编程的艺术》）

* 上面我们提到，写final域的重排序规则会要求译编器在final域的写之后，构造函数return之前，插入一个`StoreStore`障屏。读final域的重排序规则要求编译器在读final域的操作前面插入一个`LoadLoad`屏障。

* 由于x86处理器不会对写-写操作做重排序，所以在x86处理器中，写final域需要的`StoreStore`障屏会被省略掉。同样，由于x86处理器不会对存在间接依赖关系的操作做重排序，所以在x86处理器中，读final域需要的`LoadLoad`屏障也会被省略掉。也就是说**在x86处理器中，final域的读/写不会插入任何内存屏障**！