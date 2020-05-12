# **一、前言**

## **1. SQL执行慢的原因**

* **从sql本身角度来说**
  * 没有创建索引
  * **索引失效**（一些原因导致没有使用到）
  * **关联查询太多的join**
* **从服务器角度来说**
  * 服务器磁盘空间不足
  * 服务器**调优配置参数设置不合理**

## **2. 如何优化**

接下来的三、四和五章将为大家详细介绍如何优化。

# **二、MySQL性能分析工具**

在讲解优化前，先介绍MySQL几个性能分析工具。

## **1. Explain**

**MySQL查看sql执行计划的关键字**，使用explain关键字可以模拟优化器执行sql查询语句，从而得知MySQL 是如何处理sql语句。

### 1.1 语法

```sql
explain <sql语句>
```

### 1.2 结果详解

执行查看执行计划语句后返回如下10列（**MySQL5.5.48版本**）

```tex
+----+-------------+-----------+------+---------------+------+---------+------+------+-------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-----------+------+---------------+------+---------+------+------+---
```

* **id**：select 查询的**序列号**，包含一组可以重复的数字，表示查询中执行sql语句的顺序。一般以下三种情况
  * **id全部相同**：sql的执行顺序是由上到下执行
  * **id全部不同**：sql的执行顺序是id大优先执行
  * **id既存在相同，也存在不同**：先根据id大优先执行，再根据id相同由上到下执行
* **select_type**：select 查询的类型，主要是用于区别普通查询，联合查询，嵌套的复杂查询，主要包含以下几种类型
  * **SIMPLE**：**简单查询**，查询中不包含子查询或者UNION
  * **PRIMARY**：**主查询**，查询中若包含任何复杂的子部分，最外层查询则被标记为该类型
  * **SUBQUERY**：**子查询**，在SELECT或者WHERE列表中包含了的子查询
  * **DERIVED**：**衍生查询**，在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表里。
  * **UNION**：**联合查询**，有UNION的第二个和以后的查询
  * **UNION RESULT**：从UNION表获取结果的SELECT
* **table**：显示这一行的数据是关于哪张表的，对应衍生查询则表名为`derived+id`

* **type**：对表访问方式，主要包含以下几种

  * **system**：表中只有一行数据，这是const类型的特例。
  * **const**：表示通过索引一次就找到了，const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快。如将主键至于where列表中，MySQL就能将该查询转换为一个常量。
  * **eq_ref**：唯一性索引，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描。简单来说，就是多表连接中使用primary key或者 unique key作为关联条件。
  * **ref**：非唯一索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而它可能会找到多个符合条件的行。
  * **range**：只检索给定范围的行，使用一个索引来选择行。一般出现在where语句中出现了between、<、>、in等的查询，这种范围扫描索引扫描比全表扫描要好，因为他只需要开始索引的某一点，而结束语另一点，不用扫描全部索引。
  * **index**：Full Index Scan。index与ALL区别为index类型只遍历索引树。这通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）
  * **all**：Full Table Scan。将遍历全表以找到匹配的行

  **性能从最优到最差的排序：system > const > eq_ref > ref > range > index > all**，一般来说，尽量优化查询达到range级别，最好达到ref。

* **possible_keys**：显示查询语句可能用到的索引(一个或多个或为null)，不一定被查询实际使用。仅供参考使用

* **key**：显示查询语句实际使用的索引。若为null，则表示没有使用索引。

* **key_len**：显示索引中使用的字节数，可通过key_len计算查询中使用的索引长度。在不损失精确性的情况下索引长度越短越好。key_len 显示的值为索引字段的最可能长度，并非实际使用长度，即key_len是根据表定义计算而得，并不是通过表内检索出的。

* **ref**：列与索引的比较，表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

* **rows**：根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数

* **Extra**：包含不适合在其他列中显示但十分重要的额外信息，主要包括以下几种：

  * **Using filesort**：说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成的排序操作称为“文件排序” 。出现这个就要立刻优化sql。
  * **Using temporary**：使用了临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于排序 order by 和 分组查询 group by。 出现这个更要立刻优化sql
  * **Using index**：表示相应的select 操作中使用了覆盖索引（Covering index），避免访问了表的数据行，效果不错！如果同时出现Using where，表明索引被用来执行索引键值的查找。如果没有同时出现Using where，表示索引用来读取数据而非执行查找动作。覆盖索引（Covering Index） ：也叫索引覆盖，就是select 的数据列只用从索引中就能够取得，不必读取数据行，MySQL可以利用索引返回select 列表中的字段，而不必根据索引再次读取数据文件。
  * **Using where**：表明使用了where 过滤
  * **Using join buffer**：表明使用了连接缓存
  * **Impossible where**：where 语句的值总是false，不可用，不能用来获取任何元素
  * **select tables optimized away**：在没有GROUPBY子句的情况下，基于索引优化MIN/MAX操作或者
    对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，
    查询执行计划生成的阶段即完成优化。
  * **distinct**：优化distinct，在找到第一匹配的元组后即停止找同样值的工作
  

### **1.3 补充说明**

* **EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况**
* **EXPLAIN不考虑各种Cache**
* **EXPLAIN不能显示MySQL在执行查询时所作的优化工作**
* **部分统计信息是估算的，并非精确值**
* **EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划**

## **2. 慢查询日志**

### **2.1 什么是慢查询日志**

MySQL的慢查询日志是MySQL提供的一种日志记录，它用来记录在MySQL中**响应时间超过阀值的语句，即具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志**中。

### **2.2 相关参数**

* **slow_query_log**：**是否开启慢查询日志**，1表示开启，0表示关闭。
* **log-slow-queries**：**旧版（5.6以下版本）MySQL数据库慢查询日志存储路径**。可以不设置该参数，系统则会默认给一个缺省的文件`host_name-slow.log`
* **slow-query-log-file**：**新版（5.6及以上版本）MySQL数据库慢查询日志存储路径**。可以不设置该参数，系统则会默认给一个缺省的文件`host_name-slow.log`
* **long_query_time**：**慢查询阈值**，当查询时间多于设定的阈值时，记录日志。
* **log_queries_not_using_indexes**：**未使用索引的查询也被记录到慢查询日志中（可选项，默认关闭）**。
* **log_output**：**日志存储方式**。log_output='FILE'表示将日志存入文件，**默认值是'FILE'**。log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。MySQL数据库支持同时两种日志存储方式，配置的时候以逗号隔开即可，如：log_output='FILE,TABLE'。日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。

### **2.3 如何使用**

**默认情况下，Mysql数据库是关闭慢查询日志**，需要我们手动来设置这个参数，当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。

* **1）查看慢查询相关参数**

  ```shell
  # 查询慢查询是否开启和日志存储路径
  mysql> show variables like 'slow_query%';
  +---------------------+------------------------------+
  | Variable_name       | Value                        |
  +---------------------+------------------------------+
  | slow_query_log      | OFF                          |
  | slow_query_log_file | /var/lib/mysql/vm01-slow.log |
  +---------------------+------------------------------+
  2 rows in set (0.00 sec)
  
  # 查询慢查询阈值
  mysql> show variables like 'long_query_time';
  +-----------------+-----------+
  | Variable_name   | Value     |
  +-----------------+-----------+
  | long_query_time | 10.000000 |
  +-----------------+-----------+
  1 row in set (0.00 sec)
  ```

  **可以看到MySQL默认关闭慢查询日志，默认的日志存储地址为/var/lib/mysql/vm01-slow.log（默认文件名是hostname-slow.log），且默认的慢查询阈值是10s**

* **2）设置开启慢查询及相关参数**

  慢查询配置有两种方式，一种是临时配置，MySQL重启后就会失效，另一种是永久配置，MySQL重启会一直保持。建议临时配置，避免不必要的性能损耗。

  * 方法一：临时配置

    ```shell
    # 开启慢查询
    set global slow_query_log='ON';
    # 设置慢查询日志存放路径，当然你也可以使用默认路径
    set global slow_query_log_file='/var/lib/mysql/vm01-slow.log';
    # 设置慢查询时间阈值，超过这个值就记录，单位为s
    set global long_query_time=2;
    ```

    **需要关闭当前客户端，重新开启另一个客户端才能生效。**

  * 方法二：永久配置

    修改配置文件my.cnf，在[mysqld]下的下方加入

    ```properties
    slow_query_log = ON
    slow_query_log_file = /var/lib/mysql/vm01-slow.log
    long_query_time = 1
    ```

    **然后重启MySQL永久生效。**

* **3）小试牛刀**

  * a）先执行一条慢查询sql

    ```shell
    mysql> select sleep(3);
    +----------+
    | sleep(3) |
    +----------+
    |        0 |
    +----------+
    1 row in set (3.00 sec)
    ```

  * b）查看日志

    ```shell
    [root@vm01 mysql]# cat /var/lib/mysql/vm01-slow.log
    /usr/sbin/mysqld, Version: 5.5.48-log (MySQL Community Server (GPL)). started with:
    Tcp port: 3306  Unix socket: /var/lib/mysql/mysql.sock
    Time                 Id Command    Argument
    # Time: 200312 19:13:40
    # User@Host: root[root] @ localhost []
    # Query_time: 3.000278  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
    SET timestamp=1584011620;
    select sleep(3);
    ```

* **4）查询有多少条慢查询记录**

  ```shell
  mysql> show global status like '%Slow_queries%';
  +---------------+-------+
  | Variable_name | Value |
  +---------------+-------+
  | Slow_queries  | 1     |
  +---------------+-------+
  1 row in set (0.00 sec)
  ```

### 2.4 **日志分析工具mysqldumpslow**

**在生产环境中，如果要手工分析日志，查找、分析SQL，显然是个体力活，MySQL提供了日志分析工具mysqldumpslow。**

* **查看mysqldumpslow帮助信息**

  ```shell
  [root@vm01 mysql]# mysqldumpslow --help
  Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]
  
  Parse and summarize the MySQL slow query log. Options are
  
    --verbose    verbose
    --debug      debug
    --help       write this text to standard output
  
    -v           verbose
    -d           debug
    -s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
                  al: average lock time
                  ar: average rows sent
                  at: average query time
                   c: count
                   l: lock time
                   r: rows sent
                   t: query time  
    -r           reverse the sort order (largest last instead of first)
    -t NUM       just show the top n queries
    -a           don't abstract all numbers to N and strings to 'S'
    -n NUM       abstract numbers with at least n digits within names
    -g PATTERN   grep: only consider stmts that include this string
    -h HOSTNAME  hostname of db server for *-slow.log filename (can be wildcard),
                 default is '*', i.e. match all
    -i NAME      name of server instance (if using mysql.server startup script)
    -l           don't subtract lock time from total time
  ```

* **常用命令说明**

  * -s：表示按照何种方式排序
    * c：访问计数
    * l：锁定时间
    * r：返回记录
    * t：查询时间
    * al：平均锁定时间
    * ar：平均返回记录数
    * at：平均查询时间
  * -t：top n的意思，即返回前面多少条数据
  * -g：后面可以写一个正则表达式，大小写不敏感

* **例**

  * 获取返回记录集最多的10个SQL。

    ```shell
    mysqldumpslow -s r -t 10 <慢查询sql存储路径>
    ```

  * 得到访问次数最多的10个SQL

    ```shell
    mysqldumpslow -s c -t 10 <慢查询sql存储路径>
    ```

  * 得到按照时间排序的前10条里面含有左连接的查询语句。

    ```shell
    mysqldumpslow -s t -t 10 -g “left join” <慢查询sql存储路径>
    ```

  * 另外建议在使用这些命令时结合 | 和more 使用 ，否则有可能出现刷屏的情况。

    ```shell
    mysqldumpslow -s r -t 20 <慢查询sql存储路径> | more
    ```

## **3. Show profiles**

### **3.1 介绍**

Query Profiler是MYSQL自带的一种**query诊断分析工具**，通过它可以分析出一条SQL语句的**性能瓶颈**在什么地方。**通常我们是使用的explain和slow query log都无法做到精确分析，但是Query Profiler却可以定位出一条SQL语句执行的各种资源消耗情况，比如CPU，IO等，以及该SQL执行所耗费的时间等。**不过该工具只有在MYSQL 5.0.37以及以上版本中才有实现。

### **3.2 使用**

**默认的情况下，Show profiles功能是关闭的，需要自己手动启动**。Profile 功能由MySQL会话变量profiling控制，即只能在当前会话中使用。

* **查看是否开启profiles性能分析功能**

  ```sql
  mysql> select @@profiling;
  +-------------+
  | @@profiling |
  +-------------+
  |           0 |
  +-------------+
  1 row in set (0.00 sec)
  ```

  **0表示关闭，1表示开启。**

* **开启profiles功能**

  ```shell
  mysql> SET profiling = 1;
  Query OK, 0 rows affected (0.00 sec)
  ```

  **只能在当前会话使用。**

* **查询分析列表**

  ```shell
  # 执行sql
  mysql> select sleep(3);
  +----------+
  | sleep(3) |
  +----------+
  |        0 |
  +----------+
  1 row in set (3.00 sec)
  
  # 查询分析列表
  mysql> show profiles;
  +----------+------------+-----------------+
  | Query_ID | Duration   | Query           |
  +----------+------------+-----------------+
  |        1 | 3.00040300 | select sleep(3) |
  +----------+------------+-----------------+
  1 row in set (0.00 sec)
  ```

* **查询列表中某一条sql的执行细节**

  ```shell
  # 查询分析列表
  mysql> show profile for query 1;
  +--------------------------------+----------+
  | Status                         | Duration |
  +--------------------------------+----------+
  | starting                       | 0.000012 |
  | Waiting for query cache lock   | 0.000002 |
  | checking query cache for query | 0.000035 |
  | checking permissions           | 0.000005 |
  | Opening tables                 | 0.000005 |
  | init                           | 0.000007 |
  | optimizing                     | 0.000002 |
  | executing                      | 0.000006 |
  | User sleep                     | 3.000226 |
  | end                            | 0.000012 |
  | query end                      | 0.000002 |
  | closing tables                 | 0.000003 |
  | freeing items                  | 0.000035 |
  | logging slow query             | 0.000004 |
  | logging slow query             | 0.000047 |
  | cleaning up                    | 0.000003 |
  +--------------------------------+----------+
  16 rows in set (0.00 sec)
  
  # 可指定资源类型查询
  mysql> show  profile cpu ,swaps for query 1;
  +--------------------------------+----------+----------+------------+-------+
  | Status                         | Duration | CPU_user | CPU_system | Swaps |
  +--------------------------------+----------+----------+------------+-------+
  | starting                       | 0.000012 | 0.000004 |   0.000007 |     0 |
  | Waiting for query cache lock   | 0.000002 | 0.000000 |   0.000000 |     0 |
  | checking query cache for query | 0.000035 | 0.000012 |   0.000024 |     0 |
  | checking permissions           | 0.000005 | 0.000001 |   0.000003 |     0 |
  | Opening tables                 | 0.000005 | 0.000002 |   0.000003 |     0 |
  | init                           | 0.000007 | 0.000002 |   0.000004 |     0 |
  | optimizing                     | 0.000002 | 0.000000 |   0.000001 |     0 |
  | executing                      | 0.000006 | 0.000002 |   0.000004 |     0 |
  | User sleep                     | 3.000226 | 0.000650 |   0.001300 |     0 |
  | end                            | 0.000012 | 0.000003 |   0.000005 |     0 |
  | query end                      | 0.000002 | 0.000000 |   0.000001 |     0 |
  | closing tables                 | 0.000003 | 0.000001 |   0.000002 |     0 |
  | freeing items                  | 0.000035 | 0.000012 |   0.000024 |     0 |
  | logging slow query             | 0.000004 | 0.000001 |   0.000002 |     0 |
  | logging slow query             | 0.000047 | 0.000016 |   0.000032 |     0 |
  | cleaning up                    | 0.000003 | 0.000001 |   0.000002 |     0 |
  +--------------------------------+----------+----------+------------+-------+
  16 rows in set (0.00 sec)
  ```

### 3.3 注意细节

**上述的分析结果一般出现以下情况代表SQL可能需要进行优化**：

* **converting HEAP to MyISAM** 

  查询结果太大，内存都不够用了，需要往磁盘上迁移。

* **Creating tmp table** 

  创建临时表

* **Copying to tmp table on disk** 

  把内存中临时表复制到磁盘

* **locked**

  使用到了锁，可能是影响性能的因素。

# **三、索引优化**

MySQL索引即为**排好序的快速查找数据结构**，创建索引的目前就是为了加快查询。那么索引的优化主要就是**创建合适的索引，同时使其能够发挥最大作用，同时避免索引的失效**。如果对索引不太熟悉的小伙伴可以查看之前的文章

* [MySQL高级之索引详解（三）](http://xianzilei.cn/blog/33)

## **1. 索引失效场景**

前提：查询条件字段已经建立了索引

* 1）**like 以%开头**，索引失效。当like前缀没有%，后缀有%时，索引有效
* 2）**语句前后没有同时使用索引**。当or左右查询字段只有一个是索引，该索引失效，只有当or左右查询字段均为索引时，才会生效。
* 3）**组合索引，不是使用第一列索引**，索引失效（即不满足最左前缀法则）。
* 4）**数据类型出现隐式转化**。如varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。
* 5）**在索引列上使用 IS NULL 或 IS NOT NULL操作**。索引是不索引空值的，所以这样的操作不能使用索引，可以用其他的办法处理，例如：数字类型，判断大于0，字符串类型设置一个默认值，判断是否等于默认值即可。
* 6）**在索引字段上使用not，<>，!=**。不等于操作符是永远不会用到索引的，因此对它的处理只会产生全表扫描。 优化方法： key<>0 改为 key>0 or key<0。
* 7）**对索引字段进行计算操作或使用使用函数**会导致索引失效
* 8）**当全表扫描速度比索引速度快时**，mysql会使用全表扫描，此时索引失效。

## **2. 如何优化索引**

**索引的优化需要结合具体的场景和查询执行计划进行具体情况具体分析，以下概述几点基本原则**：

* 1）对于单键索引，尽量选择针**对当前query过滤性更好**的索引
* 2）在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前越好，尽量遵循**最左前缀法则**。
* 3）在选择组合索引的时候，尽量选择可以能包含当前query中的where子句中更多字段的索引，即尽量使用**覆盖索引**（只访问索引的查询（索引列和查询列一致）），减少select *
* 4）在无法避免使用like 以%开头的情况下，可以考虑使用**覆盖索引**，即查询的字段和like条件字段被索引包含。

# **四、联表优化**

在MySQL中通常使用NLJ（嵌套循环）算法来实现join。即将驱动表/外部表的结果集作为循环基础数据，然后循环从该结果集每次一条获取数据作为下一个表的过滤条件查询数据，然后合并结果。如果有多表join，则将前面的表的结果集作为循环数据，取到每行再到联接的下一个表中循环匹配，获取结果集返回给客户端。伪代码表示如下：

```sql
for each row in t1 matching range {                //外层循环
  for each row in t2 matching reference key {      //次内层循环
    for each row in t3 {                           //最内层循环
      if row satisfies join conditions,            //进行条件匹配，若满足，发给client
      send to client
    }
  }
```

**所以针对join的优化，一般核心的在于它的内外循环。**

* **1）用小结果集驱动大的结果集**

  这里驱动表即为外层循环表，使用小表作为驱动表，可以减少**被驱动表的检索次数**。当然，此优化的前提条件是通过Join条件对各个表的每次访问的资源消耗差别不是太大。如果访问存在较大的差别的时候（一般都是因为索引的区别），我们就不能简单的通过结果集的大小来判断需要Join语句的驱动顺序，而是要通过比较循环次数和每次循环所需要的消耗的乘积的大小来得到如何驱动更优化。

* **2）优先优化内层循环**

  内层循环是循环中执行次数最多的，每次循环节约很小的资源，在整个循环中就能节约很大的资源。

* **3）保证内层循环中的筛选条件使用到内表的索引**

  正如第二点所说，利用索引减少每次内层循环消耗的资源，**这也正是优化内层循环的实际优化方法**

* **4）当无法保证被驱动表的Join条件字段被索引且内存资源充足的前提下，不要太吝惜JoinBuffer的设置**

  当在某些特殊的环境中，我们的Join必须是All，Index，range或者是index_merge类型的时候，JoinBuffer就会派上用场了。在这种情况下，JoinBuffer的大小将对整个Join语句的消耗起到非常关键的作用。

# **五、分组排序优化**

## **1. order by优化**

### 1.1 order by排序方式

在MySQL中order by有两种排序方式

* **1）利用有序索引获取有序数据**

  所需的数据在index中即可全部获得且排序，无需再到表中取数据和查询排序。`Explain`分析结果中`extra`列显示`Using index`。

  * 如果是聚合索引，则order by满足最左前缀匹配原则，当前也不是必要的，例如一张表`t1`聚合索引为`(a,b,c)`，查询sql为`select a,b,c from t1 where a='1' order by b,c`。虽然order by不满足最左前缀匹配原则，但是由于a是常量，此时b，c在索引树中是排好序的，所以这里满足`Using index`
  * 如果查询了多余的字段，将会导致`using filesort`。
  * order by 多个字段如果不是同升序或同降序，则会导致`using filesort`。
  * 总之就是只在一棵索引树上就完成了查找排序，表示Using index，否则就会用到文件排序

* **2）文件排序**

  `order by`无法利用索引完成排序操作时，Mysql Query Optimizer就会选择相应的排序算法来实现。这个`filesort`并不是通过磁盘文件进行排序，而只是告诉我们进行了一个排序操作。即在MySQL Query Optimizer 所给出的执行计划中被称为文件排序（filesort）。文件排序是通过相应的排序算法，将取得的数据在内存中进行排序，MySQL需要将数据在内存中进行排序，所使用的内存区域也就是我们通过`sort_buffer_size`系统变量设置的排序区。这个排序区是每个线程独享的，所以同一时刻在MySQL中可能存在多个`sort buffer`内存区域。
  
  **MySQL支持的文件排序包含以下三种**
  
  * **1）双路排序（常规排序）**
  
    根据条件取出排序字段和可以直接定位数据的指针信息，在sort buffer中排好序后，再根据指针信息取出对应的每条数据。因此需要两次IO，效率较低。它的详细流程如下所示：
  
    * a）从表中获取满足where条件的记录
    * b）将每条记录的主键和排序键取出放到sort buffer中
    * c）如果sort buffer可以存放满足条件的（主键，排序列）对，则直接进行排序；否则会另外使用临时文件。排序采用的是**快排算法**。
    * d）扫描排好序的（主键，排序列），利用主键去获取其余需要返回的列，并将结果返回给客户端，流程结束。
  
    注意：
  
    * 是否使用临时文件主要看**sort buffer是否能容下需要排序的（主键，排序列）的结果集**，这个sort  buffer的大小由`sort_buffer_size`参数控制。
    * 两次IO：
      * 第一次：取排序字段（主键，排序列）到sort buffer中，
      * 第二次：通过上面取出的主键来取其他所需要返回列。由于返回的结果集是按排序列排序，因此主键可能是乱序的，通过乱序的主键取其余列时会产生大量的随机IO。因此对于第二次IO取MySQL本身会优化，即在取之前先将主键排序，并放入缓冲区，这个缓存区大小由参数`read_rnd_buffer_size`控制，然后有序去取记录，将随机IO转为顺序IO。
  
  * **2）单路排序（优化排序）**
  
    MySQL4.1开始的单路排序相对于双路排序来说减少了一次IO，即**一次性取出所有需要查询出来的列**放到sort buffer中，这样排好序后无需补充缺少的列，提高了排序的效率。但是比较消耗内存，属于典型的**空间换时间**的思想。当前由于放入到sort buffer的数据变多，如果sort buffer不够大，会导致需要借助临时文件的帮助，造成额外的IO。为了避免临时文件带来过大的IO，MySQL中提供了参数`max_length_for_sort_data`，只有当排序sql中出现的**所有字段类型大小总和小于该参数值**时，才能进行单路排序，否则只能用双路排序方式。
  
  * **3）优先队列排序**
  
    MySQL5.6版本针对Order by limit M，N语句，在空间层面做了优化，加入了一种新的排序方式--优先队列，这种方式采用**堆排序**实现。堆排序算法特征正好可以解limit M,N 这类排序的问题，虽然仍然需要所有字段参与排序，但是只需要M+N个元组的sort buffer空间即可，对于M，N很小的场景，基本不会因为sort buffer不够而导致需要临时文件进行归并排序的问题。对于**升序采用大顶堆**，最终堆中的元素组成了最小的N个元素，**对于降序采用小顶堆**，最终堆中的元素组成了最大的N的元素。

### 1.2 如何优化

其实上述的MySQL排序原理就已经包含了如何优化的内容，**简要来说就是尽量使用索引排序，如果不得已非要文件排序，则尽量使用单路排序减少IO消耗**。下面概述几点：

* 排序字段尽量遵循索引的**最左匹配原则**
* 对于多字段的排序尽量**升降序保持一致**
* **查询的字段尽量能够包含在索引中**
* 当无法使用索引排序时，可以根据实际情况**增大sort_buffer_size和max_length_for_sort_data的参数值**，从而减少IO的消耗。

## **2. group by优化**

**group by本质是先排序后分组，其优化策略同order by**，这里就不再赘述。**同时注意where高于having，能写在where中的限定条件就不要去having限定了**。