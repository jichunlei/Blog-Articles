# **一、MySQL简介**

**MySQL是一个开放源代码的关系型数据库管理系统，由瑞典MySQL AB公司开发，后来被Sun公司收购，Sun公司后来又被Oracle公司收购，目前属于Oracle旗下产品。因为其速度、可靠性和适应性而备受众多互联网企业青睐。**

# **二、MySQL下载安装**

## **1. 下载地址**

[下载地址](https://downloads.mysql.com/archives/community/)：以5.5.48版本为例

![](http://img.xianzilei.cn/mysql5.5.48%E4%B8%8B%E8%BD%BD%E5%9B%BE.png)

## **2. 安装步骤（RPM安装）**

* **1）上传安装包到指定的目录下（我的放在/opt目录下）**

  ```shell
  [root@vm01 opt]# cd /opt/
  [root@vm01 opt]# ll
  总用量 339244
  -rw-r--r--.  1 root root  17855952 1月  17 2016 MySQL-client-5.5.48-1.linux2.6.x86_64.rpm
  -rw-r--r--.  1 root root  50372369 1月  17 2016 MySQL-server-5.5.48-1.linux2.6.x86_64.rpm
  ```

* **2）分别安装MySQL服务端和客户端**

  ```shell
  # 安装MySQL服务端
  rpm -ivh MySQL-server-5.5.48-1.linux2.6.x86_64.rpm 
  # MySQL客户端
  rpm -ivh MySQL-client-5.5.48-1.linux2.6.x86_64.rpm 
  ```

* **3）查看是否安装成功**

  ```she
  [root@vm01 opt]# mysqladmin --version
  mysqladmin  Ver 8.42 Distrib 5.5.48, for Linux on x86_64
  ```

  安装成功！

* **4）设置root用户密码**

  ```shell
  /usr/bin/mysqladmin -u root password 你的密码
  ```

* **5）设置mysql开机自动启动**

  ```shell
  chkconfig mysql on
  ```

* **6）修改默认字符集**

  * **1）复制一份配置文件到/etc下，名称为my.cnf**

    ```shell
    cp /usr/share/mysql/my-huge.cnf /etc/my.cnf
    ```

  * **2）修改my.cnf**

    ```properties
    # 在[client]标签下添加下面内容
    default-character-set=utf8
    
    # 在[mysqld]标签下添加下面内容
    character-set-server=utf8
    character-set-client=utf8
    collation-server=utf8_general_ci
    
    # 在[mysql]标签下添加下面内容
    default-character-set=utf8
    ```

  * **3）重启MySQL后生效**

  * **4）查看是否修改成功**

    ```shell
    # 登录MySQL
    mysql> show variables like '%char%';
    +--------------------------+----------------------------+
    | Variable_name            | Value                      |
    +--------------------------+----------------------------+
    | character_set_client     | utf8                       |
    | character_set_connection | utf8                       |
    | character_set_database   | utf8                       |
    | character_set_filesystem | binary                     |
    | character_set_results    | utf8                       |
    | character_set_server     | utf8                       |
    | character_set_system     | utf8                       |
    | character_sets_dir       | /usr/share/mysql/charsets/ |
    +--------------------------+----------------------------+
    ```

    **修改成功！**

* **7）赋予远程登录用户权限**

  ```shell
  # 支持root用户允许远程连接mysql数据库,%表示任何ip
  grant all privileges on *.* to 'root'@'%' identified by '你的密码' with grant option;
  # 刷新配置
  flush privileges;
  ```

## **3. 基本操作**

* **MySQL启停**

  ```shell
  # 启动MySQL
  service mysql start
  # 关闭MySQL
  service mysql stop
  # 重启MySQL
  service mysql restart
  ```

* **MySQL登录**

  ```shell
  mysql -u root -p   # 回车输入密码，连接本地数据库
  ```

* **MySQL相关路径（默认）**

  |       路径        |           说明            |
  | :---------------: | :-----------------------: |
  |  /var/lib/mysql   | mysql数据库文件的存放路径 |
  | /user/share/mysql |     mysql配置文件目录     |
  |     /usr/bin      |     mysql相关命令目录     |
  | /etc/init.d/mysql |     mysql启停相关脚本     |

# **三、MySQL配置文件**

**配置文件位置是/etc/my.cnf**

## **1. 总览**

```pro
# Example MySQL config file for very large systems.
#
# This is for a large system with memory of 1G-2G where the system runs mainly
# MySQL.
#
# MySQL programs look for option files in a set of
# locations which depend on the deployment platform.
# You can copy this option file to one of those
# locations. For information about these locations, see:
# http://dev.mysql.com/doc/mysql/en/option-files.html
#
# In this file, you can use all long options that a program supports.
# If you want to know which options a program supports, run the program
# with the "--help" option.

# The following options will be passed to all MySQL clients
[client]
#password	= your_password
port		= 3306
socket		= /var/lib/mysql/mysql.sock
default-character-set=utf8
# Here follows entries for some specific programs

# The MySQL server
[mysqld]
port		= 3306
character-set-server=utf8
character-set-client=utf8
collation-server=utf8_general_ci
socket		= /var/lib/mysql/mysql.sock
skip-external-locking
key_buffer_size = 384M
max_allowed_packet = 1M
table_open_cache = 512
sort_buffer_size = 2M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
myisam_sort_buffer_size = 64M
thread_cache_size = 8
query_cache_size = 32M
# Try number of CPU's*2 for thread_concurrency
thread_concurrency = 8

# Don't listen on a TCP/IP port at all. This can be a security enhancement,
# if all processes that need to connect to mysqld run on the same host.
# All interaction with mysqld must be made via Unix sockets or named pipes.
# Note that using this option without enabling named pipes on Windows
# (via the "enable-named-pipe" option) will render mysqld useless!
# 
#skip-networking

# Replication Master Server (default)
# binary logging is required for replication
log-bin=mysql-bin

# required unique id between 1 and 2^32 - 1
# defaults to 1 if master-host is not set
# but will not function as a master if omitted
server-id	= 1

# Replication Slave (comment out master section to use this)
#
# To configure this host as a replication slave, you can choose between
# two methods :
#
# 1) Use the CHANGE MASTER TO command (fully described in our manual) -
#    the syntax is:
#
#    CHANGE MASTER TO MASTER_HOST=<host>, MASTER_PORT=<port>,
#    MASTER_USER=<user>, MASTER_PASSWORD=<password> ;
#
#    where you replace <host>, <user>, <password> by quoted strings and
#    <port> by the master's port number (3306 by default).
#
#    Example:
#
#    CHANGE MASTER TO MASTER_HOST='125.564.12.1', MASTER_PORT=3306,
#    MASTER_USER='joe', MASTER_PASSWORD='secret';
#
# OR
#
# 2) Set the variables below. However, in case you choose this method, then
#    start replication for the first time (even unsuccessfully, for example
#    if you mistyped the password in master-password and the slave fails to
#    connect), the slave will create a master.info file, and any later
#    change in this file to the variables' values below will be ignored and
#    overridden by the content of the master.info file, unless you shutdown
#    the slave server, delete master.info and restart the slaver server.
#    For that reason, you may want to leave the lines below untouched
#    (commented) and instead use CHANGE MASTER TO (see above)
#
# required unique id between 2 and 2^32 - 1
# (and different from the master)
# defaults to 2 if master-host is set
# but will not function as a slave if omitted
#server-id       = 2
#
# The replication master for this slave - required
#master-host     =   <hostname>
#
# The username the slave will use for authentication when connecting
# to the master - required
#master-user     =   <username>
#
# The password the slave will authenticate with when connecting to
# the master - required
#master-password =   <password>
#
# The port the master is listening on.
# optional - defaults to 3306
#master-port     =  <port>
#
# binary logging - not required for slaves, but recommended
#log-bin=mysql-bin
#
# binary logging format - mixed recommended 
#binlog_format=mixed

# Uncomment the following if you are using InnoDB tables
#innodb_data_home_dir = /var/lib/mysql
#innodb_data_file_path = ibdata1:2000M;ibdata2:10M:autoextend
#innodb_log_group_home_dir = /var/lib/mysql
# You can set .._buffer_pool_size up to 50 - 80 %
# of RAM but beware of setting memory usage too high
#innodb_buffer_pool_size = 384M
#innodb_additional_mem_pool_size = 20M
# Set .._log_file_size to 25 % of buffer pool size
#innodb_log_file_size = 100M
#innodb_log_buffer_size = 8M
#innodb_flush_log_at_trx_commit = 1
#innodb_lock_wait_timeout = 50

[mysqldump]
quick
max_allowed_packet = 16M

[mysql]
no-auto-rehash
default-character-set=utf8
# Remove the next comment character if you are not familiar with SQL
#safe-updates

[myisamchk]
key_buffer_size = 256M
sort_buffer_size = 256M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout
```

## **2. client部分**

**MySQL客户端配置**

* `port`：客户端连接服务端口
* `socket`：socket文件路径
* `default-character-set`：客户端编码

## **3. mysqld部分**

**MySQL服务器配置**

* `port`：服务端端口
* `character-set-server`：服务端编码
* `character-set-client`：客户端编码
* `collation-server`：服务端排序规则
* `socket`：socket文件路径
* `skip-external-locking`：避免外部锁定
* `key_buffer_size`：执行索引的缓冲区大小
* `max_allowed_packet`：服务器发送和接受的最大包长度
* `table_open_cache`：允许缓存已打开表的份数
* `sort_buffer_size`：执行排序使用的缓冲大小
* `read_buffer_size`：随机读缓冲区大小
* `read_rnd_buffer_size`：通信时缓存数据的大小
* `myisam_sort_buffer_size`：线程使用的堆大小
* `thread_cache_size`：在cache中保留多少线程用于重用
* `query_cache_size`：查询缓冲区大小
* `thread_concurrency`：允许的并发线程数量
* `skip-networking`：关闭通过TCP/IP连接MySQL
* `log-bin`：复制二进制日志
* `server-id`：表示本机序号为1,也就master的意思

## **4. mysqldump部分**

**备份数据库工具配置**

* `quick`：不将内存中的整个结果写入磁盘之前缓存
* `max_allowed_packet`：服务器发送和接受的最大包长度

## **5. mysql部分**

**对客户端操作相关的配置**

* `no-auto-rehash`：命令不自动补全
* `default-character-set`：默认编码
* `safe-updates`：仅允许使用带有键值的`UPDATEs`和`DELETEs`操作，即新手模式，默认注释掉的。

## **6. myisamchk部分**

**MyISAM表维护程序配置**

* `key_buffer_size`：关键词缓冲大小, 一般用来缓冲`MyISAM`表的索引块
* `sort_buffer_size`：排序缓冲大小
* `read_buffer`：读取缓冲大小
* `write_buffer`：写入缓冲大小

## **7. mysqlhotcopy部分**

**mysql数据备份程序配置**

* `interactive-timeout`：服务器关闭交互式连接前等待活动的秒数

# **四、MySQL逻辑架构**

与其他数据库相比，MySQL的逻辑架构与众不同。主要体现在它的**存储引擎的架构**上，它提供**插件式的存储引擎架构将查询处理和其他的系统任务以及数据的存储提取相分离**。这种架构可以根据业务的需求和实际需要选择合适的存储引擎，从而可以在多种不同的场景中发挥良好作用。

## **1. MySQL逻辑架构图**

![](http://img.xianzilei.cn/MySQL%E9%80%BB%E8%BE%91%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

## **2. MySQL的分层架构**

**由上面的逻辑架构图可以看出来MySQL从上到下主要分为四层：**

* **1）连接层**

  **最上层是一些客户端和连接服务**，包含本地`socket`通信和大多数基于客户端/服务端工具实现的类似于`TCP/IP`的通信，主要完成一些类似连接处理，授权认证，及相关的安全方案，在该层引入线程池的概念，为通过认证安全接入的客户端提供线程，同样在该层上可以实现基于`SSL`的安全链接，服务器也会为安全接入的每个客户端验证它所具备的操作权限。

* **2）服务层**

  **第二层架构主要完成大多数的核心服务功能**，如`SQL`接口，并完成缓存查询，`SQL`的分析和优化及部分内置函数的执行，所有跨存储引擎的功能也在这一层实现，如过程，函数等，在该层，服务层会解析查询并创建相应的内部解析树，并对其完成响应的优化确认查询表的顺序，是否利用索引等，最后生成相应的执行操作，如果是`select`语句，服务器还会查询内部的缓存，如果缓存空间足够大，这样就解决大量读操作的环境中能够很好的提供系统性能。

* **3）引擎层**

  **存储引擎层**，存储引擎真正负责了`MySQL`中数据的存储和提取，服务器通过`API`与存储引擎进行通信，不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。

* **4）存储层**

  **数据存储层**，主要讲数据存储在运行于裸设备的文件系统之上，并完成存储引擎的交互（文件系统）。

# **五、MySQL存储引擎**

## **1. 什么是存储引擎**

* 数据库存储引擎是**数据库底层软件组织**，数据库管理系统（DBMS）使用数据引擎进行**创建、查询、更新和删除数据**。不同的存储引擎提供不同的存储机制、索引技巧、锁定水平等功能，使用不同的存储引擎，还可以 获得特定的功能。现在许多不同的数据库管理系统都支持多种不同的数据引擎。
* **MySQL的核心就是存储引擎**。使用哪一种引擎需要灵活选择，一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求，使用合适的存储引擎，将会提高整个数据库的性能。

## **2. MySQL主要支持的存储引擎**

MySQL支持的存储引擎有 `InnoDB`、`MyISAM`、`Memory`、`MRG_MYISAM`、`Archive`、`Federated`、`CSV`、`BLACKHOLE` 等，其中**最常用的是InnoDB和MyISAM这两种**。

### **2.1 InnoDB存储引擎**

**从MySQL5.5版本之后，InnoDB成为MySQL的默认内置存储引擎**。他的主要特点有：

* **灾难恢复性比较好**；
* **支持事务**。默认的事务隔离级别为可重复读，通过**MVCC（并发版本控制）**来实现的；
* 使用的锁粒度为**行级锁**，可以支持**更高的并发**；
* **支持外键**；
* 配合一些热备工具可以支持**在线热备份**；
* 在InnoDB中存在着**缓冲管理**，通过缓冲池，将**索引和数据全部缓存起来**，加快查询的速度；
* 对于InnoDB类型的表，其数据的物理组织形式是**聚簇表**。所有的数据按照主键来组织。数据和索引放在一块，都位于B+树的叶子节点上。

### **2.2 MyISAM存储引擎**

**在5.5版本之前，MyISAM是MySQL的默认存储引擎**，该存储引擎**并发性差**，**不支持事务**，所以使用场景比较少。它的主要特点为：

* **不支持事务**；
* **不支持外键**，如果强行增加外键，不会提示错误，只是**外键不其作用**；
* 对数据的查询缓存**只会缓存索引**，不会像**InnoDB一样缓存数据**，而且是利用操作系统本身的缓存；
* 默认的锁粒度为**表级锁**，所以**并发度很差**，加锁快，锁冲突较少，所以**不太容易发生死锁**；
* **支持全文索引**（MySQL5.6之后，InnoDB存储引擎也对全文索引做了支持），但是MySQL的全文索引基本不会使用，对于全文索引，现在有其他成熟的解决方案，比如：ElasticSearch，Solr，Sphinx等；
* 数据库所在主机如果宕机，MyISAM的数据文件**容易损坏**，而且**难恢复**。

### **2.3 MEMORY存储引擎（了解）**

**将数据存在内存中，和市场上的Redis，memcached等思想类似，主要为了提高数据的访问速度**。主要特点：

* **支持的数据类型有限制**，比如：不支持TEXT和BLOB类型，对于字符串类型的数据，只支持固定长度的行，VARCHAR会被自动存储为CHAR类型；
* 支持的锁粒度为**表级锁**。所以，在访问量比较大时，表级锁会成为MEMORY存储引擎的瓶颈；
* 由于数据是存放在内存中，所以在**服务器重启之后，所有数据都会丢失**；
* 查询的时候，如果有用到临时表，而且临时表中有BLOB，TEXT类型的字段，那么这个临时表就会转化为MyISAM类型的表，性能会急剧降低。

### **2.4 ARCHIVE存储引擎（了解）**

**ARCHIVE存储引擎适合的场景有限，由于其支持压缩，故主要是用来做日志，流水等数据的归档**。主要特点：

* **支持Zlib压缩**，数据在插入表之前，会先被压缩；
* 仅支持SELECT和INSERT操作，存入的数据就**只能查询，不能做修改和删除**；
* **只支持自增键上的索引，不支持其他索引**。

## **3. InnoDB和MyISAM的对比**

|     功能     |                            InnoDB                            |                            MyISAM                            |
| :----------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|   **事务**   |      **支持**。每一条SQL语言都默认封装成事务，自动提交       |                          **不支持**                          |
|   **外键**   |                           **支持**                           | **不支持**。如果强行增加外键，不会提示错误，只是外键不其作用 |
|   **索引**   | InnoDB是**聚集索引**，使用B+Tree作为索引结构，数据文件是和（主键）索引绑在一起的（表数据文件本身就是按B+Tree组织的一个索引结构），必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。InnoDB的B+树主键索引的叶子节点就是数据文件，辅助索引的叶子节点是主键的值 | MyISAM是**非聚集索引**，也是使用B+Tree作为索引结构，索引和数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。MyISAM的B+树主键索引和辅助索引的叶子节点都是数据文件的地址指针 |
|  **表行数**  | InnoDB**不保存表的具体行数**，执行select count(\*) from table时需要全表扫描。因为InnoDB的事务特性，在同一时刻表中的行数对于不同的事务而言是不一样的，因此count统计会计算对于当前事务而言可以统计到的行数，而不是将总行数储存起来方便快速查询。InnoDB会尝试遍历一个尽可能小的索引除非优化器提示使用别的索引。如果二级索引不存在，InnoDB还会尝试去遍历其他聚簇索引。 | MyISAM用一个**变量保存了整个表的行数**，执行上述语句时只需要读出该变量即可，速度很快（注意不能加有任何WHERE条件） |
| **全文索引** |  Innodb**不支持**全文索引（5.7以后的InnoDB支持全文索引了）   | MyISAM**支持**全文索引，在涉及全文索引领域的查询效率上MyISAM速度更快高 |
|    **锁**    | InnoDB**支持表、行(默认)级锁**。InnoDB的行锁是实现在索引上的，而不是锁在物理行记录上。即如果访问没有命中索引，也无法使用行锁，将要退化为表锁。 |                    MyISAM**仅支持表级锁**                    |
|   **主键**   | InnoDB表**必须有主键**（用户没有指定的话会自己找或生产一个主键）。InnoDB推荐使用自增ID作为主键，因为自增ID可以保证每次插入时B+索引是从右边扩展的，可以避免B+树和频繁合并和分裂（对比使用UUID）。如果使用字符串主键和随机主键，会使得数据随机插入，效率比较差。 |                      MyISAM**可以没有**                      |
| **存储文件** | **Innodb存储文件有frm、ibd**。frm是表定义文件，ibd是数据文件 | **MyISAM是frm、MYD、MYI**。frm是表定义文件，myd是数据文件，myi是索引文件 |

## **4. 如何选择合适的存储引擎**

* **MyISAM**

  如果应用是以**读操作**和**插入操作**为主，只有**很少的更新和删除操作**，并且对**事务的完整性、并发性要求不是很高**，那么选择这个存 储引擎是非常适合的。MyISAM 是在 Web、数据仓储和其他应用环境下常使用的存储引擎之一。

* **InnoDB**

  如果应用对**事务**的完整性有比较高的要求，在并发条件下要求数据的**一致性**，数据操作除了插入和查询以外，还包括很多的**更新**、 **删除**操作，那么 InnoDB 存储引擎应该是比较合适的选择。InnoDB 存储引擎除了有效地降低由于删除和更新导致的锁定，还可以确保事务的完整交（Commit）和回滚（Rollback）， 对于类似计费系统或者财务系统等对数据准确性要求比较高的系统，InnoDB 都是合适的选择。 