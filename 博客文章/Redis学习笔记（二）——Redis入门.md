[TOC]

# 一、Redis 的介绍

## 1. 什么是Redis

* 全称：Remote Dictionary Server (远程字典服务)
* redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。
* Redis 是一个高性能的key-value数据库。 redis的出现，很大程度补偿了memcached这类key/value存储的不足，在部 分场合可以对关系数据库起到很好的补充作用。它提供了Java，C/C++，C#，PHP，JavaScript，Perl，Object-C，Python，Ruby，Erlang等客户端，使用很方便。
* Redis支持主从同步。数据可以从主服务器向任意数量的从服务器上同步，从服务器可以是关联其他从服务器的主服务器。这使得Redis可执行单层树复制。存盘可以有意无意的对数据进行写操作。由于完全实现了发布/订阅机制，使得从数据库在任何地方同步树时，可订阅一个频道并接收主服务器完整的消息发布记录。同步对读取操作的可扩展性和数据冗余很有帮助。

## 2. Redis特性

### 2.1 性能极高 

* Redis能读的速度是110000次/s,写的速度是81000次/s 。

### 2.2 基于键值对的数据结构

* 几乎所有的编程语言都提供了类似字典的功能，例如Java里的map、Python里的dict，类似于这种组织数据的方式叫作基于键值的方式，与很多键值对数据库不同的是，Redis中的值不仅可以是字符串，而且还可以是具体的数据结构，这样不仅能便于在许多应用场景的开发，同时也能够提高开发效率。Redis的全称是REmote Dictionary Server，它主要提供了5种数据结构：字符串、哈希、列表、集合、有序集合

### 2.3 丰富的功能 

除了5种数据结构，Redis还提供了许多额外的功能：

- 提供了键过期功能，可以用来实现缓存。
- 提供了发布订阅功能，可以用来实现消息系统。
- 支持Lua脚本功能，可以利用Lua创造出新的Redis命令。
- 提供了简单的事务功能，能在一定程度上保证事务特性。
- 提供了流水线（Pipeline）功能，这样客户端能将一批命令一次性传到Redis，减少了网络的开销。

### 2.4 简单稳定

Redis的简单主要表现在三个方面。

- Redis的源码很少。
- Redis使用**单线程模型**，这样不仅使得Redis服务端处理模型变得简单，而且也使得客户端开发变得简单。
- Redis不需要依赖于操作系统中的类库（例如Memcache需要依赖libevent这样的系统类库），Redis自己实现了事件处理的相关功能。

### 2.5 客户端语言多

Redis提供了简单的TCP通信协议，很多编程语言可以很方便地接入到Redis，并且由于Redis受到社区和各大公司的广泛认可，所以支持Redis的客户端语言也非常多，几乎涵盖了主流的编程语言，例如Java、PHP、Python、C、C++、Nodejs等。

### 2.6持久化

通常看，将数据放在内存中是不安全的，一旦发生断电或者机器故障，重要的数据可能就会丢失，因此Redis提供了两种持久化方式：**RDB**和**AOF**（这块后续会单独写篇文章讨论），即可以用两种策略将内存的数据保存到硬盘中（如图所示）这样就保证了数据的可持久性。

### 2.7 主从复制

Redis提供了复制功能，实现了多个相同数据的Redis副本（如图所示），复制功能是分布式Redis的基础。

### 2.8 高可用和分布式

Redis从2.8版本正式提供了高可用实现Redis Sentinel，它能够保证Redis节点的故障发现和故障自动转移。Redis从3.0版本正式提供了分布式实现**Redis Cluster**，它是Redis真正的分布式实现，提供了高可用、读写和容量的扩展性。

## 3. Redis与其他key-value存储区别

* Redis有着更为复杂的数据结构并且提供对他们的原子性操作，这是一个不同于其他数据库的进化路径。Redis的数据类型都是基于基本数据结构的同时对程序员透明，无需进行额外的抽象。
* Redis运行在内存中但是可以持久化到磁盘，所以在对不同数据集进行高速读写时需要权衡内存，因为数据量不能大于硬件内存。在内存数据库方面的另一个优点是，相比在磁盘上相同的复杂的数据结构，在内存中操作起来非常简单，这样Redis可以做很多内部复杂性很强的事情。同时，在磁盘格式方面他们是紧凑的以追加的方式产生的，因为他们并不需要进行随机访问。

## 4. Redis数据结构

### 4.1 字符串(strings)

* 最简单Redis类型
* 一个key对应一个value
* String类型是二进制安全的，即Redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象
* 应用场景
  * 缓存： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力。
  * 计数器：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。
  * session：常见方案spring session + redis实现session共享

### 4.2 哈希(hashes)

* 值本身又是一种键值对结构，如key={{field1,value1},......fieldN,valueN}}
* 类似Java中的Map<String,Map>结构
* 应用场景
  * 缓存： 能直观，相比string更节省空间，的维护缓存信息，如用户信息，视频信息等。

### 4.3 列表(lists)

* 实质就是链表（双向循环列表）
* value可以重复，可以通过下标取出对应的value值，
* 左右两边都能进行插入和删除数据。
* 使用技巧
  * lpush+lpop=Stack(栈)
  * lpush+rpop=Queue（队列）
  * lpush+ltrim=Capped Collection（有限集合）
  * lpush+brpop=Message Queue（消息队列）

* 应用场景
  * timeline：例如微博的时间轴，有人发布微博，用lpush加入时间轴，展示新的列表信息

### 4.4 集合(sets)

* 保存多个字符串的元素
* 与列表区别
  * 不允许有重复的元素
  * 集合中的元素是无序的，不能通过索引下标获取元素
  * 支持集合间的操作，可以取多个集合取交集、并集、差集

* 应用场景
  * 标签（tag）,给用户添加标签，或者用户给消息添加标签，这样有同一标签或者类似标签的可以给推荐关注的事或者关注的人。
  * 点赞，或点踩，收藏等，可以放到set中实现

### 4.5 有序集合(sorted sets)

* 实质为集合+score(分数)
* 元素是可以排序的（根据score排序）
* 应用场景
  * 排行榜：例如小说视频等网站需要对用户上传的小说视频做排行榜，榜单可以按照用户关注数，更新时间，字数等打分，做排行。

# 二、Redis 的下载、安装与卸载

## 1. 下载

* [官方下载地址](https://redis.io/download)

* [中文网站下载地址](http://redis.cn/download.html)

## 2. 安装

安装教程非常简单，官网上也有详细讲解，以5.0.5版本为例

### 2.1 解压压缩包

将下载好的压缩包放到linux上某个路径下，执行解压命令，我默认解压到/opt/install目录下

```shell
[root@iz2zehcv8qx0677zmjq8wnz software]# tar xzf redis-5.0.5.tar.gz -C /opt/install/
[root@iz2zehcv8qx0677zmjq8wnz software]# cd /opt/install/
[root@iz2zehcv8qx0677zmjq8wnz install]# ll
total 4
drwxrwxr-x 6 root root 4096 May 16  2019 redis-5.0.5
```

### 2.2 编译

进入解压后的文件夹中

```shell
cd redis-5.0.5
#执行make
make
```

等候安装完成

### 2.3 安装redis

* 进入src目录执行安装命令

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz redis-5.0.5]# cd src/
  [root@iz2zehcv8qx0677zmjq8wnz src]# make install
      CC Makefile.dep
  
  Hint: It's a good idea to run 'make test' ;)
  
      INSTALL install
      INSTALL install
      INSTALL install
      INSTALL install
      INSTALL install
  ```

* 启动命令默认安装在`/usr/local/bin/`目录下

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz src]# cd /usr/local/bin/
  [root@iz2zehcv8qx0677zmjq8wnz bin]# ll
  total 32744
  -rwxr-xr-x 1 root root 4365984 Jan 10 18:20 redis-benchmark
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-check-aof
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-check-rdb
  -rwxr-xr-x 1 root root 4806328 Jan 10 18:20 redis-cli
  lrwxrwxrwx 1 root root      12 Jan 10 18:20 redis-sentinel -> redis-server
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-server
  ```

* 安装完毕

## 3.卸载

卸载redis也是非常的简单

### 3.1 停止redis服务

* 首先查看redis服务是否在运行

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz ~]# ps aux | grep redis
  root     19345  0.1  0.1 163060  2448 ?        Ssl  Jan06   7:47 ./redis-server *:6379
  root     31599  0.0  0.0 112660   968 pts/2    R+   17:54   0:00 grep --color=auto redis
  ```

* 关闭redis服务

  * 如果之前没有设置密码

    ```shell
    redis-cli -p 6379 shutdown
    ```

  * 如果之前设置过密码

    * 修改/etc/init.d/redis启动脚本

    * 添加如下内容

      ```
      EXEC=/usr/local/bin/redis-server
      CLIEXEC=/usr/local/bin/redis-cli
      PIDFILE=/var/run/redis_6379.pid
      CONF="/etc/redis/redis.conf"
      REDISPORT="6379"
      PASSWORD=$(cat $CONF|grep '^\s*requirepass'|awk '{print $2}'|sed 's/"//g')
      if [ -z $PASSWORD ]
      then
          $CLIEXEC -p $REDISPORT shutdown
      else
          $CLIEXEC -a $PASSWORD -p $REDISPORT shutdown
      fi
      ```

    * 执行关闭命令

      ```shell
      redis-cli -a 你的密码 -p 6379 shutdown
      ```

* 删除make的时候生成的几个redisXXX的文件

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz bin]# ll /usr/local/bin/
  total 32748
  -rw-r--r-- 1 root root     133 Jan 10 18:02 dump.rdb
  -rwxr-xr-x 1 root root 4365984 Jan  5 10:29 redis-benchmark
  -rwxr-xr-x 1 root root 8116520 Jan  5 10:29 redis-check-aof
  -rwxr-xr-x 1 root root 8116520 Jan  5 10:29 redis-check-rdb
  -rwxr-xr-x 1 root root 4806328 Jan  5 10:29 redis-cli
  lrwxrwxrwx 1 root root      12 Jan  5 10:29 redis-sentinel -> redis-server
  -rwxr-xr-x 1 root root 8116520 Jan  5 10:29 redis-server
  [root@iz2zehcv8qx0677zmjq8wnz bin]# rm -rf /usr/local/bin/redis*
  [root@iz2zehcv8qx0677zmjq8wnz bin]# rm -rf /usr/local/bin/dump*
  [root@iz2zehcv8qx0677zmjq8wnz bin]# ll
  total 0
  ```

* 删除解压后的所有文件

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz software]# rm -rf redis-5.0.5/
  ```

* 卸载完毕

# 三、Redis启动与关闭

## 1.启动

* 进入启动命令目录`/usr/local/bin/`

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz src]# cd /usr/local/bin/
  [root@iz2zehcv8qx0677zmjq8wnz bin]# ll
  total 32744
  -rwxr-xr-x 1 root root 4365984 Jan 10 18:20 redis-benchmark
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-check-aof
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-check-rdb
  -rwxr-xr-x 1 root root 4806328 Jan 10 18:20 redis-cli
  lrwxrwxrwx 1 root root      12 Jan 10 18:20 redis-sentinel -> redis-server
  -rwxr-xr-x 1 root root 8116520 Jan 10 18:20 redis-server
  ```

* 启动Redis，出现下面表示启动成功

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz bin]# redis-server 
  3460:C 10 Jan 2020 18:26:25.809 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
  3460:C 10 Jan 2020 18:26:25.809 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=3460, just started
  3460:C 10 Jan 2020 18:26:25.809 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
                  _._                                                  
             _.-``__ ''-._                                             
        _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
    .-`` .-```.  ```\/    _.,_ ''-._                                   
   (    '      ,       .-`  | `,    )     Running in standalone mode
   |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
   |    `-._   `._    /     _.-'    |     PID: 3460
    `-._    `-._  `-./  _.-'    _.-'                                   
   |`-._`-._    `-.__.-'    _.-'_.-'|                                  
   |    `-._`-._        _.-'_.-'    |           http://redis.io        
    `-._    `-._`-.__.-'_.-'    _.-'                                   
   |`-._`-._    `-.__.-'    _.-'_.-'|                                  
   |    `-._`-._        _.-'_.-'    |                                  
    `-._    `-._`-.__.-'_.-'    _.-'                                   
        `-._    `-.__.-'    _.-'                                       
            `-._        _.-'                                           
                `-.__.-'                                               
  
  3460:M 10 Jan 2020 18:26:25.811 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
  3460:M 10 Jan 2020 18:26:25.811 # Server initialized
  3460:M 10 Jan 2020 18:26:25.811 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
  3460:M 10 Jan 2020 18:26:25.811 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
  3460:M 10 Jan 2020 18:26:25.811 * Ready to accept connections
  ```

* 相关配置修改

  * 上述启动是前台启动，操作不方便，可以修改redis配置文件使其后台启动

    * 进入redis配置文件（redis安装目录中的redis.conf文件）

      ```shell
      [root@iz2zehcv8qx0677zmjq8wnz redis-5.0.5]# vi /opt/install/redis-5.0.5/redis.conf
      ```

    * 修改其中的参数daemonize

      ```shell
      daemonize no
      修改为：
      daemonize yes
      ```

  * 设置外网可访问

    * 注释 bind 127.0.0.1：#bind 127.0.0.1
    * 修改protected-mode为no：protected-mode no
    * 建议设置密码：
      * 去除requirepass的注释
      * 设置密码：requirepass 你的密码

    * 重启redis即可生效

* 客户端连接

  ```shell
  ./redis-cli [-h 127.0.0.1 -p 6379 -a 你的密码]
  ```

  如果未设置密码，去除-a参数即可

  ```shell
  ./redis-cli [-h 127.0.0.1 -p 6379]
  ```

## 2.关闭

```shell
[root@iz2zehcv8qx0677zmjq8wnz bin]# redis-cli shutdown
```

# 参考

* [Redis详解（一）------ redis的简介与安装](https://www.cnblogs.com/ysocean/p/9074353.html#_label6)
* [Redis基础知识详解(非原创）](https://www.cnblogs.com/WUXIAOCHANG/p/10832330.html)
* [redis的八大特性](https://blog.51cto.com/13581826/2308263)