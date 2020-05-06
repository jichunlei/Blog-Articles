# **一、前言**

- 在前面几篇博客中已经详细介绍了Redis的基本操作、配置及持久化等内容，尤其持久化特性可以保证redis宕机情况的数据的**基本完整性**。
- 但是如果redis发生了重大故障（例**如系统崩溃**或**硬盘损坏**等），导致数据的直接丢失，这样很可能对业务造成灾难性的打击。
- 所以为了**避免单点故障**造成的数据丢失，，在实际生产环境下通常将**数据复制多个副本保存在不同的服务器**上，这样可以避免单一服务器故障导致数据丢失的情况。
- `Redis`提供了多种高可用的方案：包括**主从复制**、**哨兵模式**及**集群**。

# **二、主从复制介绍**

## **1. 简介**

- 在主从复制中，`Redis`数据库分为两类：**主库（master）**和**从库（slave）**。主库可以进行读写操作，其中的写操作导致数据变化会自动同步到从库。从库一般是只读的（如果从库下也有从库，则可读可写，可以通过参数`replica-read-only`指定），同时接受来自主库的数据，主库与从库的关系是一对多关系。

- redis的主从架构有如下**两种模式**：

  - **一主多从**

    ![](http://img.xianzilei.cn/Redis一主多从架构图.jpg)

	**主库M下有从库S1、S2等，从库接受主库M的数据同步**

  - **链式主从**

    ![](http://img.xianzilei.cn/Redis链式主从架构图.jpg)

    主库M下有从库S1和S2，从库S1下有从库S11和S12，主库M的数据会同步到从库S1和S2，同时从库S1的数据会同步到从库S11和S22，即主库M和从库S1都可读可写，但是如果在从库S1上的写操作，无法同步到主库M和丛库S2。因此这样会造成数据的不一致，故该种架构模式不推荐，也很少见。

## **2. 基本实践（以一主二从为例）**

- **拷贝三份`redis.conf`文件**：名称以**端口号**为区分，分为`redis6379.conf`、`redis6380.conf`和`redis6381.conf`三个文件

  - 创建三个redis目录（存放这三个配置文件）

    ```shell
    mkdir -p /opt/db/{redis6379,redis6380,redis6381}
    ```

  - 拷贝三个配置文件（我的redis安装在/opt/install目录下0）

    ```shell
    cp /opt/install/redis-5.0.5/redis.conf /opt/db/redis6379/redis6379.conf
    cp /opt/install/redis-5.0.5/redis.conf /opt/db/redis6380/redis6380.conf
    cp /opt/install/redis-5.0.5/redis.conf /opt/db/redis6381/redis6381.conf
    ```

- **修改每个配置文件对应内容**：主要修改**端口号**及对应的文件名称

  - `redis6379.conf`

    ```properties
    #修改redis为后台运行模式
    daemonize yes   
    #修改运行的redis实例的pid，不能重复
    pidfile /var/run/redis_6379.pid  
    #指明日志文件
    logfile "/opt/db/redis6379/6379.log"  
    #工作目录，存放持久化数据的目录
    dir "/opt/db/redis6379"   
    #监听地址，如果是单机多个示例可以不用修改
    bind 127.0.0.1
    #监听端口，保持和配置文件名称端口一致
    port 6379         
    ```

  - `redis6380.conf`

    ```properties
    #修改redis为后台运行模式
    daemonize yes   
    #修改运行的redis实例的pid，不能重复
    pidfile /var/run/redis_6380.pid  
    #指明日志文件
    logfile "/opt/db/redis6380/6380.log"  
    #工作目录，存放持久化数据的目录
    dir "/opt/db/redis6380"   
    #监听地址，如果是单机多个示例可以不用修改
    bind 127.0.0.1   
    #监听端口，保持和配置文件名称端口一致
    port 6380    
    ```

  - `redis6381.conf`

    ```properties
    #修改redis为后台运行模式
    daemonize yes   
    #修改运行的redis实例的pid，不能重复
    pidfile /var/run/redis_6381.pid  
    #指明日志文件
    logfile "/opt/db/redis6381/6381.log"  
    #工作目录，存放持久化数据的目录
    dir "/opt/db/redis6381"   
    #监听地址，如果是单机多个示例可以不用修改
    bind 127.0.0.1   
    #监听端口，保持和配置文件名称端口一致
    port 6381
    ```

- **启动这三个redis实例**

  ```shell
  redis-server /opt/db/redis6379/redis6379.conf 
  redis-server /opt/db/redis6380/redis6380.conf 
  redis-server /opt/db/redis6381/redis6381.conf 
  ```

  查看进程，启动成功

  ```shell
  [root@iz2zehcv8qx0677zmjq8wnz bin]# ps -ef|grep redis
  root     17447     1  0 14:39 ?        00:00:00 redis-server 127.0.0.1:6379
  root     17452     1  0 14:39 ?        00:00:00 redis-server 127.0.0.1:6380
  root     17457     1  0 14:39 ?        00:00:00 redis-server 127.0.0.1:6381
  root     17467 17123  0 14:40 pts/1    00:00:00 grep --color=auto redis
  ```

- **设置主从关系（以6379作为主库，其余作为从库）**

  - **在从库执行命令**：`slaveof 主库ip 主库端口号`

    ```shell
    #配置6380从库
    [root@iz2wegcv8qx0877zmjq9wnz bin]# redis-cli -p 6380
    127.0.0.1:6380> SLAVEOF 127.0.0.1 6379
    OK
    127.0.0.1:6380> CONFIG REWRITE
    OK
    
    #配置6381从库
    [root@iz2wegcv8qx0877zmjq9wnz bin]# redis-cli -p 6381
    127.0.0.1:6381> SLAVEOF 127.0.0.1 6379
    OK
    127.0.0.1:6381> CONFIG REWRITE
    OK
    ```

  - **查看配置结果**，执行命令`INFO replication`

    - **主库信息**

      ```shell
      127.0.0.1:6379> INFO replication
      # Replication
      role:master
      connected_slaves:2
      slave0:ip=127.0.0.1,port=6380,state=online,offset=392,lag=0
      slave1:ip=127.0.0.1,port=6381,state=online,offset=392,lag=0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:1
      repl_backlog_histlen:392
      ```

      可以看到主库的`role`为`master`，`connected_slaves`表示其下有两个从库6380和6381

    - **从库信息**

      ```shell
      #从库6380
      127.0.0.1:6380> INFO replication
      # Replication
      role:slave
      master_host:127.0.0.1
      master_port:6379
      master_link_status:up
      master_last_io_seconds_ago:7
      master_sync_in_progress:0
      slave_repl_offset:392
      slave_priority:100
      slave_read_only:1
      connected_slaves:0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:1
      repl_backlog_histlen:392
      
      
      #从库6381
      127.0.0.1:6381> INFO replication
      # Replication
      role:slave
      master_host:127.0.0.1
      master_port:6379
      master_link_status:up
      master_last_io_seconds_ago:10
      master_sync_in_progress:0
      slave_repl_offset:392
      slave_priority:100
      slave_read_only:1
      connected_slaves:0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:15
      repl_backlog_histlen:378
      ```

      从库的`role`为`slave`，其主库：`master_host:127.0.0.1`和`master_port:6379`

- **测试**

  - **我们先主库写入些数据**

    ```shell
    127.0.0.1:6379> set k1 v1
    OK
    127.0.0.1:6379> set k2 v2
    OK
    ```

  - **然后从库读取**

    ```shell
    #从库6380读取
    127.0.0.1:6380> get k1
    "v1"
    127.0.0.1:6380> get k2
    "v2"
    
    #从库6381读取
    127.0.0.1:6381> get k1
    "v1"
    127.0.0.1:6381> get k2
    "v2"
    ```

    **测试成功！**

  - **如果我在从库写数据会怎么样，试一下**

    ```shell
    127.0.0.1:6381> set k3 v3
    (error) READONLY You can't write against a read only replica.
    ```

    **可见当前配置的从库只有读权限，无法写入数据。**

## **3. 注意细节**

- 1）上述我们是通过命令进行配置的，如果服务重启，则配置失效，角色关系会丢失。如果想重启后仍保持这种角色关系，可以在**从库的配置文件**中配置如下信息：

  ```properties
  #redis5.0.5版本配置文件
  replicaof 127.0.0.1 6379
  ```

- 2）当执行命令`slaveof <masterip> <masterport>`或者通过配置文件配置并重启时，**从库会全量同步主库的信息，之后进行增量同步**。

- 3）从库默认无法进行写操作，这是由于配置文件中有如下的默认配置

  ```properties
  #redis5.0.5版本配置文件
  replica-read-only yes
  ```

  可以将其修改为no，这样从库就可以进行写操作，但是数据不能同步到主库，会导致主从数据不一致，**不推荐修改**。

- 4）默认情况下，如果主库宕机，从库的角色并不会变化始终是`slave`，可以理解为**原地待命**；并且如果**主库重新启动成功，主库的角色又会恢复，即一切正常**。

# **三、主从复制原理**

## **1. 复制流程**

**Redis复制可以分为以下三个阶段**

- **1）复制初始阶段**

  - 当从库执行`slaveof  <masterip>  <masterport>`命令后，从库会根据ip和端口向主库**发起socket连接**，主库收到socket连接后将连接信息保存，此时连接建立。 
  - socket连接建立完成后，从库会**向主库发送`PING`命令**，来确认主库的可用性，如果主库返回`PONG`则代表主库可用，如果出现超时或阻塞那么从库会断开socket连接，然后重试。
  - 注意：如果主库设置了密码，那么从库需要**设置`masterauth`参数**（可在配置文件中配置），此时从库会发送`auth`命令，命令格式为`auth+password`。如果从库未配置`masterauth`参数，则会报错，socket连接断开。
  - 当身份验证完成后，**从库发送自己的监听端口**，**主库保存其端口信息**，此时**进入数据同步阶段**。                  

- **2）数据同步阶段**

  - 当完成初始阶段后，主从库便可以进行数据同步，此时从库会**第一次向从库发送同步命令**（具体的命令细节在后面的内容中会进行讲解），主库收到命令后执行`BGSAVE` 命令**生成或更新RDB文件**，并且**使用一个缓冲区记录从此刻开始执行的所有写命令**。
  - 当主库的上述操作执行完成后，会将**生成或更新后的RDB文件发送给从库**
  - 从库接收到`RDB`文件后，会**将自己的数据状态更新为RDB中内容** 
  - 主库**将缓冲区的所有写命令也发送给从库**，从库执行相应命令
  - 至此，主从库的**全量同步**完成

  ![Redis全量复制流程](http://img.xianzilei.cn/Redis全量同步流程.png)

  ​							                                **Redis全量复制流程**

- **3）命令传播阶段**

  - 当全量同步结束后，如果主库进行修改操作，为了保持主从库的一致性，**主库会将执行的写操作命令传播到从库中**，从库执行相应的命令后，主从库继续保持一致
  - 注意：如果从库在同步主库期间因为某些原因或故障连接断开，而在断开连接的这段时间内主库进行了一些写操作，这时候如果从服务器重新连接后，主库如何将那段时间内写操作同步到从库中？
    - 最初的做法同初次连接做法一样，进行主从库的**全量更新**（即通过全量的RDB文件同步）
    - 由于全量同步**耗时长**，**性能差**，在Redis2.8版本后使用新的同步命令`PSYNC`来替代老的`SYNC`命令（细节后面内容讲解），**主库只需将断开连接后执行的写命令发送给从库执行**即可。

## **2. Redis复制各版本演进**

**Redis 各版本的复制流程基本在于复制命令的优化，下面将分别介绍**：

- **1）2.6<=Redis版本<2.8**

  - **复制命令**：复制**采用`SYNC`命令**
  - **复制流程**：即无论是**启动和断线重连**都采用**全量复制**进行数据的同步
  - **缺点**：如果数据量过大，会**加大同步过程的耗时**，**影响性能**。

- **2）2.8<=Redis版本<4.0**

  - **复制命令**：复制**采用`PSYNC`命令**

  - **三个关键概念：** 

    - **offset（复制偏移量）**

      主库和从库各自维护一个**复制偏移量**（可以提供命令`info replication`查看），**用于标识自己的复制情况**。**主库的复制偏移量表示主库向从节点传递的字节数，从库表示从库同步的字节数**。每当主库向从库发送N个字节数据，主库的offset增加N，从库每接收到主库传来的N字节数据时，从库的offset增加N。**若主从库的offset相同则表示数据同步，否则不同步**，因为可以作为判断主从数据是否一致的标志。例如上述实践中的信息：

      ```shell
      #主6379库信息
      #master_repl_offset为392
      127.0.0.1:6379> INFO replication
      # Replication
      role:master
      connected_slaves:2
      slave0:ip=127.0.0.1,port=6380,state=online,offset=392,lag=0
      slave1:ip=127.0.0.1,port=6381,state=online,offset=392,lag=0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:1
      repl_backlog_histlen:392
      
      #从库6380
      #slave_repl_offset为392
      127.0.0.1:6380> INFO replication
      # Replication
      role:slave
      master_host:127.0.0.1
      master_port:6379
      master_link_status:up
      master_last_io_seconds_ago:7
      master_sync_in_progress:0
      slave_repl_offset:392
      slave_priority:100
      slave_read_only:1
      connected_slaves:0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:1
      repl_backlog_histlen:392
      
      
      #从库6381
      #slave_repl_offset为392
      127.0.0.1:6381> INFO replication
      # Replication
      role:slave
      master_host:127.0.0.1
      master_port:6379
      master_link_status:up
      master_last_io_seconds_ago:10
      master_sync_in_progress:0
      slave_repl_offset:392
      slave_priority:100
      slave_read_only:1
      connected_slaves:0
      master_replid:8425a07915495564ff9a732e99657ebc1a1d08ec
      master_replid2:0000000000000000000000000000000000000000
      master_repl_offset:392
      second_repl_offset:-1
      repl_backlog_active:1
      repl_backlog_size:1048576
      repl_backlog_first_byte_offset:15
      repl_backlog_histlen:378
      ```

    - **replication backlog buffer（复制积压缓冲区）**

      **复制积压缓冲区**是一个固定长度的FIFO队列，大小默认为1MB（可以通过配置参数`repl-backlog-size`指定），该队列有且仅有一个，所有的slave共享此缓冲区，主要用于备份最近主库发送给从库的数据。在主从命令传播阶段，主库除了将写命令发送给从库外，还会发送一份到改复制积压缓冲区，作为备份。除了存储最近的写命令，该缓冲区还存储了每个字节对应的复制偏移量。

    - **run_id(服务器运行的唯一ID)** 

      每个Redis实例在启动的时候都会随机生成一个**长度为40位的唯一字符串来标识当前运行的redis节点**（可以通过命令`info server`查看）。当主从库初次复制时主节点将自己的run_id发送给从库，从库会保存起来，当从库断线重连时，从库会将这个run_id发送给主节点，主节点根据run_id判断进行全量还是部分复制：

      - **如果run_id与自己的不同，则表示从库之前的主库并不是自己，这时候会进行全量复制**；
      - **如果run_id与自己的一致。则会查询偏移量之后的数据是否还在复制积压缓冲区里，如果在，则进行部分复制，否则全量复制**。

  - **复制流程**：

    - 如果从节点以前**没有复制过任何主节点**，或者之前执行过`slaveof no one`命令，那么从节点在开始第一次复制时会向主节点发送命令`PSYNC ? -1`的命令，主动请求主节点进行全量复制。
    - 如果从节点**复制过某个服务器**，那么从节点在第一次复制的时候会向主节点发送命令`PSYNC <runid> <offset>`（runid是上一次复制的主节点的id。offset是对应的偏移量），接收到这个命令的主节点会通过这两个参数判断对从节点进行哪种同步操作（主节点只有在**runid与自己相同**和**偏移量存在自己的缓冲区内**才进行增量复制，否则全量复制）
    - 主节点判断结束后会发送如下**三种回复**：
      - a）**如果主节点回复`+FULLRESYNC <runid> <offset>`：**则表示主节点将要与从节点进行全量复制，其中runid是主节点的运行id，从节点会报存runid和offset，作为下次发送`PSYNC`命令的参数。
      - b）**如果主节点回复`+CONTINUE`：**表示主节点将于从节点执行增量复制，从节点接收主节点发送自己缺少的那部分数据即可。
      - c）**如果主节点回复`-ERR`：**表示主节点版本低于`Redis 2.8`，识别不了命令`PSYNC`，从节点将向主节点发送`SYNC`命令，与主节点进行全量复制。

  - **缺点**：缺点很明显，如果从节点**重新启动**，那么它之前保存的**主节点的runid就会丢失**（**runid不会持久化**），那么意味着从库必然会进行全量复制。即只有在**非重启情况下的断线重连**才有可能进行增量复制。

- **3）Redis版本>=4.0**

  - **复制命令**：优化后的`PSYNC`命令
  - **两个关键概念**
    - **master_replid**：长度为41个字节（40个随机串+'0'）的字符串，每个redis实例都有，与`run_id`没有直接关联，但是生成规则相同。如果实例变为从节点时，自己的`master_replid`会被主节点的`master_replid`覆盖。也就是说**主节点的`master_replid`是自己随机生成的，从节点的`master_replid`是主节点的`master_replid`。**
    - **master_replid2**：默认初始化为全0，用于**存储上次主节点的`master_replid`（如果没有上次主节点，则全0）**。

  - **复制流程**：
    - **1）存储复制信息**：redis在关闭的时候，会调用`rdbSaveInfoAuxFields`函数，把当实例的`repl-id`和`repl-offset`保存到RDB文件中（可通过`redis-check-rdb <RDB文件路径>`命令查看）；
    - **2）重启后加载RDB文件中的复制信息**：`redis`加载`RDB`文件的时候，会专门处理文件中的辅助字段（`AUX FIELDS`）信息，把其中的`repl-id`和`repl-offset`变量值分别赋值到`master_replid`和`master_repl_offset`上（注意：**如果从节点开启了AOF持久化，redis会优先加载AOF文件，但是AOF文件没有存储对应的复制信息，所以导致重启后只能进行全量复制**）；
    - **3）向主库发送复制信息**：从节点将`master_replid`和`master_repl_offset+1`的信息发送到主节点，**从节点同时满足下面两个条件即可进行增量复制，否则全量复制**：
      - a）从节点发送的`master_replid`与主节点的`master_replid`和`master_replid2`有一个相等（用于判断主从关系）
      - b）从节点发送的`master_repl_offset+1`还存在与主节点的复制积压缓冲区中（用于判断从库缺失的部分是否还存在缓冲区内）

  - **优点**：
    - **支持从节点重启的部分同步机制**
    - **支持主节点故障时候从节点切换为主节点时候的部分同步机制**

# **四、哨兵模式**

前面的介绍说到，主节点只有一个，一旦主节点挂掉，默认情况下，从节点保持“**原地待命**”的状态，那么就会导致整个系统无法运作。如主节点挂掉之后，从节点能够“**主动请缨**”，自动变成主节点，是不是就可以完美解决问题了？因此**哨兵模式**就诞生了。

## **1. 介绍**

哨兵模式就是**不时地监控redis是否按照预期良好地运行**（至少是保证主节点是存在的），若一台主机出现问题时，哨兵会自动将该主机下的某一个从机设置为新的主机，并让其他从机和新主机建立主从关系。

## 2. 配置步骤

- 1）在配置文件目录下新建 `sentinel.conf` （文件名不可修改），并配置如下信息

  ```properties
  #sentinel monitor 被监控机器的名字(自己起名字) 被监控的ip地址 端口号 得票数
  sentinel monitor sentinel6379 127.0.0.1 6379 1
  ```

  得票数1表示主机挂掉之后，slave投票让谁接替成为主机，得票数大于1即可成为主机

- 2）**启动哨兵**

  ```shell
  redis-sentinel <sentinel.conf文件地址>
  ```

- 3）这样就**配置成功**了（注意：哨兵模式也存在着单点故障问题，即哨兵机器挂了，就无法进行监控了，解决办法是哨兵也建立集群，这里就不在赘述了）。

# **五、主从配置参数总览**

- **主库参数**

  ```properties
  repl-disable-tcp-nodelay no
  #在slave和master同步后（发送psync/sync），后续的同步是否设置成TCP_NODELAY假如设置成yes，则redis会合并小的TCP包从而节省带宽，但会增加同步延迟（40ms），造成master与slave数据不一致假如设置成no，则redis master会立即发送同步数据，没有延迟
  #前者关注性能，后者关注一致性
  
  repl-ping-slave-period 10
  #从库会按照一个时间间隔向主库发送PING命令来判断主服务器是否在线，默认是10秒
  
  repl-backlog-size 1mb
  #复制积压缓冲区大小设置
  
  repl-backlog-ttl 3600
  #master没有slave一段时间会释放复制缓冲区的内存，repl-backlog-ttl用来设置该时间长度。单位为秒。
  
  min-slaves-to-write 3
  min-slaves-max-lag 10
  #设置某个时间断内，如果从库数量小于该某个值则不允许主机进行写操作，以上参数表示10秒内如果主库的从节点小于3个，则主库不接受写请求，min-slaves-to-write 0代表关闭此功能。
  ```

- **从库参数**

  ```properties
  slaveof <masterip> <masterport> 
  #设置该数据库为其他数据库的从数据库
  
  masterauth <master-password>
  #主从复制中，设置连接master服务器的密码（前提master启用了认证）
  
  slave-serve-stale-data yes
  # 当从库同主库失去连接或者复制正在进行，从库有两种运行方式：
  # 1) 如果slave-serve-stale-data设置为yes(默认设置)，从库会继续相应客户端的请求
  # 2) 如果slave-serve-stale-data设置为no，除了INFO和SLAVOF命令之外的任何请求都会返回一个错误"SYNC with master in progress"
  
  slave-priority 100
  #当主库发生宕机时候，哨兵会选择优先级最高的一个称为主库，从库优先级配置默认100，数值越小优先级越高
  
  slave-read-only yes
  #从节点是否只读；默认yes只读，为了保持数据一致性，应保持默认
  ```

# **参考**

- [Redis复制官方文档](https://redis.io/topics/replication)
- [Redis复制官方中文文档](http://www.redis.cn/topics/replication.html)