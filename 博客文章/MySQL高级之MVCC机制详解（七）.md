# **一、前言**

在文章[MySQL高级之锁机制详解（五）](http://xianzilei.cn/blog/38)中，我们详解介绍了MySQL行级锁的概念及使用。但是锁的操作毕竟会影响性能，为了提升性能，可否采用不加锁的方式进行，下面举一个案例：

* 表t包含两列id和col，其中id为主键

  |  id  | col  |
  | :--: | :--: |
  |  1   |  10  |
  |  3   |  15  |

* 现在事务A更新id=1的数据

  ```sql
  update t1 set col=11 where id=1;
  ```

* 事务A还未提交，事务B查询id=1的数据

  ```sql
  select * from t where id=1;
  ```

* 按照之前行级锁的介绍，事务A会获取id=1这一行的排它锁，其余事务无法对该数据进行读或写操作，但是事务B的查询操作并没有被阻塞，反而直接可以执行，并且查询出来的数据是原来的col=10。这是为什么？

* **此处正是利用了MVCC多数据做了多版本处理，读取的数据来源于快照**

# **二、MVCC概述**

## **1. 什么是当前读和快照读**

在学习MVCC的时候，我们先了解一下InnoDB下的**当前读**和**快照读**。

* **当前读**

  指的是**读取当前记录的最新版本**。读取的时候需要保证其他的并发事务不能修改当前记录，会**对读取的记录进行加锁**。如**加锁的读操作**，**增删改操作**都是当前读。

* **快照读**

  即**读取记录数据的可见版本（可能是过期的数据），不加锁**。比如不加锁的select操作就是快照读。快照读的实现是**基于MVCC**的。快照读的**前提是隔离级别不是串行化级别**，否则会退化成当前读。

## **2. 什么是MVCC**

> 全称`Multi-Version Concurrency Control`，即多版本并发控制。MVCC是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，在编程语言中实现事务内存。

* `MVCC`最大的特点是**读不加锁，读写不冲突**。与MVCC相对的，是基于锁的并发控制。但是基于锁的控制下，数据库性能得不到提升。在读多写少的OLTP应用中，读写不冲突是非常重要的，MVCC极大的增加了系统的并发性能。

* `MVCC`在**MySQL InnoDB**中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读。

## **3. 当前读，快照读和MVCC的关系**

* 准确的说，MVCC多版本并发控制指的是 “维持一个数据的多个版本，使得读写操作没有冲突” 这么一个概念。仅仅是一个理想概念
* 而在MySQL中，实现这么一个MVCC理想概念，我们就需要MySQL提供具体的功能去实现它，而快照读就是MySQL为我们实现MVCC理想模型的其中一个具体非阻塞读功能。而相对而言，当前读就是悲观锁的具体功能实现
* 要说的再细致一些，快照读本身也是一个抽象概念，再深入研究。MVCC模型在MySQL中的具体实现则是由 3个隐式字段，undo日志 ，Read View 等去完成的，具体可以看下面的MVCC实现原理

## **4. MVCC的优势**

* **数据库并发场景及可能产生的问题**

  * **1）读-读**：不存在任何问题，也不需要并发控制
  * **2）读-写**：可能会造成事务隔离性问题，可能遇到**脏读**，**幻读**，**不可重复读**
  * **3）写-写**：可能会存在更新丢失问题，比如**第一类更新丢失**，**第二类更新丢失**

* **MVCC带来的好处**

  多版本并发控制（MVCC）是一种用来解决**读-写冲突**的**无锁并发控制**，即**为事务分配单向增长的时间戳，为每个修改保存一个版本，版本与事务时间戳关联，读操作只读该事务开始前的数据库的快照**。 

  * 在并发读写数据库时，可以做到在读操作时不用阻塞写操作，写操作也不用阻塞读操作，提高了数据库并发读写的性能
  * 同时还可以解决脏读，幻读，不可重复读等事务隔离问题，但不能解决更新丢失问题（需要配合锁来解决）。

# **三、MVCC实现原理**

MVCC的目的就是多版本并发控制，在数据库中的实现，就是为了解决`读写冲突`，它的实现原理主要是依赖记录中的 **3个隐式字段**，**undo日志** ，**Read View** 来实现的。所以我们先来看看这个三个point的概念

## **1. 隐式字段**

数据库中每行记录除了我们自定义的字段外，还有数据库隐式定义的`DB_TRX_ID`,`DB_ROLL_PTR`,`DB_ROW_ID`等字段。

* **DB_TRX_ID**
  6byte，**最近修改(修改/插入)事务ID**：记录创建这条记录/最后一次修改该记录的事务ID
* **DB_ROLL_PTR**
  7byte，**回滚指针，指向这条记录的上一个版本**（存储于rollback segment里）
* **DB_ROW_ID**
  6byte，**隐含的自增ID（隐藏主键）**，如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引
* 实际还有一个**删除flag隐藏字段**, 既记录被更新或删除并不代表真的删除，而是删除flag变了

| name | age  | DB_ROW_ID(隐式主键) | DB_TRX_ID(事务ID) | DB_ROLL_PTR(回滚指针) |
| ---- | ---- | ------------------- | ----------------- | --------------------- |
| Jack | 23   | 1                   | 1                 | 0x12345678            |

如上图所示，`DB_ROW_ID`是数据库默认为该行记录生成的**唯一隐式主键**，`DB_TRX_ID`是当前操作该记录的**事务ID**,而`DB_ROLL_PTR`是一个回滚指针，用于配合undo日志，指向**上一个旧版本**。

## **2. undo日志**

**undo log分为两种**

* **insert undo log**
  **事务在insert新记录时产生的undo log**。只在事务回滚时需要，并且在事务提交后可以被立即丢弃。
* **update undo log**
  **事务在进行update或delete时产生的undo log**。不仅在事务回滚时需要，在快照读时也需要；所以不能随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被`purge`线程统一清除。

> **purge**
>
> * 从前面的分析可以看出，为了实现InnoDB的MVCC机制，更新或者删除操作都只是设置一下老记录的deleted_bit，并不真正将过时的记录删除。
>
> * 为了节省磁盘空间，InnoDB有专门的purge线程来清理deleted_bit为true的记录。为了不影响MVCC的正常工作，purge线程自己也维护了一个read view（这个read view相当于系统中最老活跃事务的read view）;如果某个记录的deleted_bit为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。

对MVCC有帮助的实质是`update undo log` ，`undo log`实际上就是存在`rollback segment`中旧记录链，**它的执行流程如下：**

* **1）某个事务在person表插入了一条新记录，记录如下，name为Jerry, age为24岁，隐式主键是1，事务ID和回滚指针，我们假设为NULL**

  ![](https://img-blog.csdnimg.cn/20190313213836406.png)

* **2）现在来了一个事务1对该记录的name做出了修改，改为Tom**

  * 在事务1修改该行(记录)数据时，数据库会先对该行加排他锁

  * 然后把该行数据拷贝到undo log中，作为旧记录，既在undo log中有当前行的拷贝副本

  * 拷贝完毕后，修改该行name为Tom，并且修改隐藏字段的事务ID为当前事务1的ID, 我们默认从1开始，之后递增，回滚指针指向拷贝到undo log的副本记录，既表示我的上一个版本就是它

  * 事务提交后，释放锁

    

    ![](https://img-blog.csdnimg.cn/20190313220441831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

* **3）又来了个事务2修改person表的同一个记录，将age修改为30岁**

  * 在事务2修改该行数据时，数据库也先为该行加锁
  * 然后把该行数据拷贝到undo log中，作为旧记录，发现该行记录已经有undo log了，那么最新的旧数据作为链表的表头，插在该行记录的undo log最前面
  * 修改该行age为30，并且修改隐藏字段的事务ID为当前事务2的ID, 那就是2，回滚指针指向刚刚拷贝到undo log的副本记录
  * 事务提交，释放锁

  ![](https://img-blog.csdnimg.cn/20190313220528630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

从上面，我们就可以看出，不同事务或者相同事务的对同一记录的修改，会导致该记录的undo log成为一条**记录版本线性表**，即链表，**undo log的链首就是最新的旧记录，链尾就是最早的旧记录**（当然就像之前说的该undo log的节点可能是会purge线程清除掉，向图中的第一条insert undo log，其实在事务提交之后可能就被删除丢失了，不过这里为了演示，所以还放在这里）。

## **3. Read View(读视图)**

Read View就是**事务进行快照读操作的时候生产的读视图(Read View)**，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)。

所以我们知道 Read View主要是用来做**可见性判断**的, 即当我们某个事务执行快照读的时候，对该记录创建一个Read View读视图，**把它比作条件用来判断当前事务能够看到哪个版本的数据**，既可能是当前最新的数据，也有可能是该行记录的undo log里面的某个版本的数据。

Read View遵循一个**可见性算法**，主要是将要被修改的数据的最新记录中的DB_TRX_ID（即当前事务ID）取出来，与系统当前其他活跃事务的ID去对比（由Read View维护），如果DB_TRX_ID跟Read View的属性做了某些比较，不符合可见性，那就通过DB_ROLL_PTR回滚指针去取出Undo Log中的DB_TRX_ID再比较，即遍历链表的DB_TRX_ID（**从链首到链尾，即从最近的一次修改查起**），直到找到满足特定条件的DB_TRX_ID, 那么这个DB_TRX_ID所在的旧记录就是当前事务能看见的**最新老版本**。

![](https://img-blog.csdnimg.cn/20190314144440494.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

上图是别处摘过来的源码图，它是一段MySQL判断可见性的一段源码，该方法展示了我们拿DB_TRX_ID去跟Read View某些属性进行怎么样的比较。

在展示之前，先简化一下`Read View`，我们可以把`Read View`简单的理解成有三个全局属性

* **trx_list**（名字随便取的）
  一个数值列表，用来维护`Read View`生成时刻**系统正活跃的事务ID**
* **up_limit_id**
  **记录trx_list列表中事务ID最小的ID**
* **low_limit_id**
  ReadView生成时刻系统尚未分配的下一个事务ID，也就是**目前已出现过的事务ID的最大值+1**

判断流程如下：

* 首先比较`DB_TRX_ID` < `up_limit_id`, 如果小于，则当前事务能看到`DB_TRX_ID` 所在的记录，如果大于等于进入下一个判断
* 接下来判断 `DB_TRX_ID` 大于等于 `low_limit_id` , 如果大于等于则代表`DB_TRX_ID` 所在的记录在`Read View`生成后才出现的，那对当前事务肯定不可见，如果小于则进入下一个判断
* 判断`DB_TRX_ID` 是否在活跃事务之中，`trx_list.contains(DB_TRX_ID)`，如果在，则代表我`Read View`生成时刻，你这个事务还在活跃，还没有Commit，你修改的数据，我当前事务也是看不见的；如果不在，则说明，你这个事务在Read View生成之前就已经Commit了，你修改的结果，我当前事务是能看见的

## **4. 完整流程**

在了解完**隐式字段**，**undo log**以及**Read View**的概念之后，我们来看看MVCC实现的整体流程

* 当事务2对某行数据执行了快照读，数据库为该行数据生成一个Read View读视图，假设当前事务ID为2，此时还有事务1和事务3在活跃中，事务4在事务2快照读前一刻提交更新了，所以Read View记录了系统当前活跃事务1，3的ID，维护在一个列表上，假设我们称为trx_list

  | 事务1    | 事务2    | 事务3    | 事务4        |
  | -------- | -------- | -------- | ------------ |
  | 事务开始 | 事务开始 | 事务开始 | 事务开始     |
  | ...      | ...      | ...      | 修改且已提交 |
  | 进行中   | 快照读   | 进行中   |              |
  | ...      | ...      | ...      |              |

* `Read View`不仅仅会通过一个列表`trx_list`来维护事务2执行快照读那刻系统正活跃的事务ID，还会有两个属性`up_limit_id`（记录`trx_list`列表中事务ID最小的ID），`low_limit_id`(记录`trx_list`列表中事务ID最大的ID+1) ；所以在这里例子中`up_limit_id`就是1，`low_limit_id`就是4 + 1 = 5，`trx_list`集合的值是1,3，Read View如下图

  ![](https://img-blog.csdnimg.cn/20190313224045780.png)

* 我们的例子中，只有事务4修改过该行记录，并在事务2执行快照读前，就提交了事务，所以当前该行当前数据的`undo log`如下图所示；我们的事务2在快照读该行记录的时候，就会拿该行记录的`DB_TRX_ID`去跟`up_limit_id`,`low_limit_id`和活跃事务ID列表(`trx_list`)进行比较，判断当前事务2能看到该记录的版本是哪个。

  ![](https://img-blog.csdnimg.cn/2019031322511052.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

* 所以先拿该记录`DB_TRX_ID`字段记录的事务ID 4去跟`Read View`的的`up_limit_id`比较，看4是否小于`up_limit_id(1)`，所以不符合条件，继续判断 4 是否大于等于 `low_limit_id(5)`，也不符合条件，最后判断4是否处于`trx_list`中的活跃事务, 最后发现事务ID为4的事务不在当前活跃事务列表中, 符合可见性条件，所以事务4修改后提交的最新结果对事务2快照读时是可见的，所以事务2能读到的最新数据记录是事务4所提交的版本，而事务4提交的版本也是全局角度上最新的版本

  ![](https://img-blog.csdnimg.cn/20190314141320189.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

# **转载地址**

[【MySQL笔记】正确的理解MySQL的MVCC及实现原理](https://blog.csdn.net/SnailMann/article/details/94724197)