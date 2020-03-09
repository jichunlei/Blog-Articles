# **一、索引的定义**

MySQL官方定义：

> 索引是存储引擎用于快速找到记录的一种数据结构

- 索引的本质：**索引是数据结构**
- 可以理解为**排好序的快速查找数据结构**。数据本身之外,数据库还维护着一个满足特定查找算法的数据结构，这些数据结构以某种方式指向数据，这样就可以在这些数据结构的基础上实现高级查找算法，这种数据结构就是索引
- 一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以**文件形式**存储在硬盘上

# **二、索引的优劣势**

- **优势**
  - 1）索引大大减少了服务器需要扫描的数据量
  - 2）索引可以帮助服务器避免排序和临时表
  - 3）索引可以将随机I/O变为顺序I/O
- **劣势**
  - 1）索引占用磁盘或者内存空间
  - 2）减慢了插入更新操作的速度

# **三、MySQL索引分类**

## **1. 从逻辑角度**

- **普通索引**：最基本的索引，没有任何限制。

  - **创建索引**

    - 1）直接创建索引

      ```sql
      CREATE INDEX index_name ON table(column(length));
      ```

    - 2）修改表结构的方式添加索引

      ```sql
      ALTER TABLE table_name ADD INDEX index_name ON (column(length));
      ```

    - 3）创建表的同时创建索引

      ```sql
      CREATE TABLE `table` (
          `id` int(11) NOT NULL AUTO_INCREMENT ,
          `title` char(255) CHARACTER NOT NULL ,
          `content` text CHARACTER NULL ,
          `time` int(10) NULL DEFAULT NULL ,
          PRIMARY KEY (`id`),
          INDEX index_name (title(length))
      );
      ```

  - **删除索引**

    ```sql
    DROP INDEX index_name ON table;
    ```

- **唯一索引**：与前面的普通索引类似，不同的就是：索引列的值必须**唯一**，但允许有空值。如果是组合索引，则列值的组合必须**唯一**。

  - **创建索引**

    - 1）直接创建索引

      ```sql
      CREATE UNIQUE INDEX indexName ON table(column(length));
      ```

    - 2）修改表结构的方式添加索引

      ```sql
      ALTER TABLE table_name ADD UNIQUE indexName ON (column(length));
      ```

    - 3）创建表的同时创建索引

      ```sql
      CREATE TABLE `table` (
          `id` int(11) NOT NULL AUTO_INCREMENT ,
          `title` char(255) CHARACTER NOT NULL ,
          `content` text CHARACTER NULL ,
          `time` int(10) NULL DEFAULT NULL ,
          UNIQUE indexName (title(length))
      );
      ```

- **主键索引**：特殊的唯一索引，一个表只能有一个**主键**，**不允许有空值**。一般是在建表的时候同时创建主键索引：

  ```sql
  CREATE TABLE `table` (
      `id` int(11) NOT NULL AUTO_INCREMENT ,
      `title` char(255) NOT NULL ,
      PRIMARY KEY (`id`)
  );
  ```

- **组合索引**：指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用组合索引时遵循最左前缀集合

  ```sql
  ALTER TABLE `table` ADD INDEX index_name (column1,column2,column3); 
  ```

- **全文索引**：主要用来查找文本中的关键字，而不是直接与索引中的值相比较。fulltext索引跟其它索引大不相同，它更像是一个搜索引擎，而不是简单的where语句的参数匹配。fulltext索引配合match against操作使用，而不是一般的where语句加like。它可以在create table，alter table ，create index使用，不过目前只有char、varchar，text 列上可以创建全文索引。值得一提的是，在数据量较大时候，现将数据放入一个没有全局索引的表中，然后再用CREATE index创建fulltext索引，要比先为一张表建立fulltext然后再将数据写入的速度快很多。

  - 创建表的适合添加全文索引

    ```sql
    CREATE TABLE `table` (
        `id` int(11) NOT NULL AUTO_INCREMENT ,
        `title` char(255) CHARACTER NOT NULL ,
        `content` text CHARACTER NULL ,
        `time` int(10) NULL DEFAULT NULL ,
        PRIMARY KEY (`id`),
        FULLTEXT (content)
    );
    ```

  - 修改表结构添加全文索引

    ```sql
    ALTER TABLE article ADD FULLTEXT index_content(content);
    ```

  - 直接创建索引

    ```sql
    CREATE FULLTEXT INDEX index_content ON article(content);
    ```

## **2. 从数据结构角度**

- **B/B+Tree索引**：如果不熟悉B/B+树，可以参考之前的文章（[一文彻底弄懂B树和B+树](http://xianzilei.cn/blog/31)）

- **Hash索引**：基于hash表结构实现的索引，只支持精确查找，不支持范围查找，不支持排序。mysql中只有MEMORY/HEAP和NDB存储引擎支持

- **FULLTEXT索引**：主要用来查找文本中的关键字，而不是直接与索引中的值相比较。fulltext索引跟其它索引大不相同，它更像是一个搜索引擎，而不是简单的where语句的参数匹配。fulltext索引配合match against操作使用。目前MyISAM和InnoDB引擎都支持了FULLTEXT索引。

- **R-Tree索引**：空间索引。用于对GIS数据类型创建SPATIAL索引。

  | 索引              | MyISAM引擎 | InnoDB引擎         |
  | ----------------- | ---------- | ------------------ |
  | **B-Tree索引**    | 支持       | 支持               |
  | **HASH索引**      | 不支持     | 不支持             |
  | **R-Tree索引**    | 支持       | 不支持             |
  | **Full-text索引** | 支持       | 暂不支持(现在支持) |

## **3. 从存储结构角度**

- **聚簇索引**

  > 定义：数据行的物理顺序与列值（一般是主键的那一列）的逻辑顺序相同，一个表中只能拥有一个聚集索引。

  **类似于新华字典的拼音目录**。新华字典的拼音目录按照a-z的拼音顺序排序，相应的正文内容也是按照a-z的拼音顺序排序。如果我们新增个新汉字，它以拼音B开头，相应的它会被插入到A和C之间的位置。**聚集索引对于那些经常要搜索范围值的列特别有效。**使用聚集索引找到包含第一个值的行后，便可以确保包含后续索引值的行在物理相邻。**在mysql**中**InnoDB引擎是唯一支持聚集索引的存储引擎。InnoDB按照主键（Primary Key）进行聚集**，如果没有定义主键，InnoDB会试着使用唯一的非空索引来代替。如果没有这种索引，InnoDB就会定义隐藏的主键然后在上面进行聚集。

- **非聚簇索引**

  > 定义：该索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同，一个表中可以拥有多个非聚集索引。

  **类似于新华字典的偏旁目录**。偏旁的排序与实际正文的排序不一定一致。**MyISAM存储引擎采用的是非聚簇索引**。如果涉及到大数据量的排序、全表扫描、count之类的操作的话，MyISAM占优势些，因为索引所占空间小，这些操作是需要在内存中完成的。

# **四、MySQL索引实现**

目前大部分数据库系统及文件系统都采用**B-Tree(B树)**或其变种**B+Tree(B+树)**作为索引结构。**B+Tree是数据库系统实现索引的首选数据结构**。在 MySQL 中,索引属于存储引擎级别的概念,不同存储引擎对索引的实现方式是不同的,本文主要讨论 `MyISAM` 和 `InnoDB` 两个存储引擎的索引实现方式。

## **1. MyISAM索引实现**

MyISAM存储引擎使用**B+树**作为索引结构，其中**叶节点的data域存放的是数据记录的地址**。

* **主索引原理图**

  ![](http://img.xianzilei.cn/MyISAM%E5%BC%95%E6%93%8E%E4%B8%BB%E7%B4%A2%E5%BC%95.png)

  上图所示是MyISAM存储引擎主索引样例图，右下角为一张表的三个列col1、col2和col3，其中col1位主键，该图以主键为索引，其中0x07为十六进制，表示表中某一列数据的存放地址。由此看出叶子节点的data域存放的是数据块的引用（即地址），并非存储真实数据。

* **辅助索引原理图**

  ![](http://img.xianzilei.cn/MyISAM%E5%BC%95%E6%93%8E%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95.png)

  同样我们也可以建立辅助索引，以col2位索引建立辅助索引。在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。

* **MyISAM 的索引方式也叫做非聚簇索引**。因为它的叶子节点的data域存放的是数据记录的地址，所以它的索引的顺序与实际数据记录的位置没有必然的联系，即顺序不一定一致，所以又叫非聚簇索引。

## **2. InnoDB索引实现**

 InnoDB 也使用**B+树**作为索引结构，但具体实现方式与MyISAM有所不同

* **主索引原理图**

  ![](http://img.xianzilei.cn/InnoDB%E5%BC%95%E6%93%8E%E4%B8%BB%E7%B4%A2%E5%BC%95.png)

  由上图可以看出：`InnoDB`的**数据文件本身就是索引文件**，即它的叶子节点的data域包含了**完整的数据记录**。**该索引的key是数据表的主键**。`InnoDB`的数据文件本身要按主键聚集，所以`InnoDB`要求表**必须有主键**（`MyISAM`可以没有），如果没有显式指定，则MySQL系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则MySQL自动为`InnoDB`表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整型。

* **辅助索引原理图**

  ![](http://img.xianzilei.cn/InnoDB%E5%BC%95%E6%93%8E%E8%BE%85%E5%8A%A9%E7%B4%A2%E5%BC%95.png)

  上图为辅助索引图，即非主键列作为索引key，但是这里需注意的是辅助索引的叶子节点的data域存放的是该数据记录的主键值而不是数据记录本身，因为使用辅助索引查询的时候需要进行二次查询（第一次查询到数据记录的主键值，第二次根据主索引查询到实际的数据记录）

## **3. 二者索引区别**

* **1）主键索引**

  `MyISAM`的主键索引中**索引文件与数据文件是分开的**，索引文件的叶子节点的**data域存放数据记录的十六进制存储地址**，因此查询数据时需要进行二次查找；而`InnoDB`的**索引文件和数据文件是一体**的，它的叶子节点的**data域存放完整的数据记录**，因此一次查询即可获得数据记录。

* **2） 辅助索引**

  `MyISAM`的主键索引的叶子节点**data域存放的是数据记录的十六进制存储地址**，而`InnoDB`存放的是**该数据记录的主键值**，也类似于”存储地址“。二者均需要进行**二次查找**。

* **3）索引类型**

  `MyISAM`主键索引属于**非聚簇索引**，即索引顺序与数据记录顺序没有必然联系；`InnoDB`主键索引属于**聚簇索引**，索引顺序与数据记录顺序一致。

# **五、MySQL索引创建时机**

## **1. 哪些情况需要创建索引**

* 主键会自动创建唯一索引
* 频繁作为查询条件的字段应该创建索引
* 查询中与其他表关联的字段，外键关系建立索引
* 高并发情况下推荐创建组合索引
* 排序字段建议创建索引
* 统计或分组的字段建议创建索引

## **2. 哪些情况不要创建索引**

* 表记录数据量过少时无需创建索引
* 查询条件用不到的字段不用创建索引
* 频繁更新的字段不适合创建索引
* 经常增删改的表不建议创建索引
* 数据重复且分布平均的表字段，创建索引作用不大，不建议创建索引