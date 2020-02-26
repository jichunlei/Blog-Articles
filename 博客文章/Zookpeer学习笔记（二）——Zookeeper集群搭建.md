# **一、下载**

* [下载地址](https://archive.apache.org/dist/zookeeper/)
* 选择合适的版本，下面以**3.4.10版本**为例。

# **二、单机搭建**

## **1. 环境准备**

* **1） 安装JDK**

* **2）上传安装包到服务器上**

* **3）解压到指定目录下**（我的在/usr/local/zookeeper目录下，需在/usr/local目录下创建zookeeper文件夹）

  ```shell
  mkdir /usr/local/zookeeper
  tar -zxvf zookeeper-3.4.10.tar.gz -C /usr/local/zookeeper
  ```

## **2. 配置修改**

* **1）在zookeeper安装目录下创建data和log文件夹**

  ```shell
  mkdir /usr/local/zookeeper/zookeeper-3.4.10/data
  mkdir /usr/local/zookeeper/zookeeper-3.4.10/log
  ```

* **2）data文件夹下创建文件名为myid的文件（内容为zookeeper的编号，比如1）**

  ```shell
  cd /usr/local/zookeeper/zookeeper-3.4.10/data
  touch myid
  vim myid #写入1
  ```

* **3）修改zoo_sample.cfg文件并改名为zoo.cfg**

  ```shell
  mv zoo_sample.cfg zoo.cfg
  ```

* **4）修改zoo.cfg内容，主要修改dataDir（数据目录）和dataLogDir（日志目录）为步骤1创建的两个文件夹路径，同时最后加上集群信息（具体的配置后面讲解）**

  ```properties
  # The number of milliseconds of each tick
  tickTime=2000
  # The number of ticks that the initial 
  # synchronization phase can take
  initLimit=10
  # The number of ticks that can pass between 
  # sending a request and getting an acknowledgement
  syncLimit=5
  # the directory where the snapshot is stored.
  # do not use /tmp for storage, /tmp here is just 
  # example sakes.
  dataDir=/usr/local/zookeeper/zookeeper-3.4.10/data
  dataLogDir=/usr/local/zookeeper/zookeeper-3.4.10/log
  # the port at which the clients will connect
  clientPort=2181
  # the maximum number of client connections.
  # increase this if you need to handle more clients
  #maxClientCnxns=60
  #
  # Be sure to read the maintenance section of the 
  # administrator guide before turning on autopurge.
  #
  # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
  #
  # The number of snapshots to retain in dataDir
  #autopurge.snapRetainCount=3
  # Purge task interval in hours
  # Set to "0" to disable auto purge feature
  #autopurge.purgeInterval=1
  
  server.1=192.168.245.128:2881:3881
  ```
  
* **zoo.cfg配置参数详解**

  * **tickTime**

    **通信心跳数**，服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个`tickTime`时间就会发送一个心跳，时间单位为毫秒。它用于心跳机制，并且设置最小的session超时时间为**两倍心跳时间**。(session的最小超时时间是`2*tickTime`)

  * **initLimit**

    **初始通信时限**，集群中的`Follower`跟随者服务器与`Leader`领导者服务器之间**初始连接时能容忍的最多心跳数**（`tickTime`的数量），用它来限定集群中的`Zookeeper`服务器连接到`Leader`的时限。

  * **syncLimit**

    集群中`Leader`与`Follower`之间的**最大响应时间单位**，假如响应超过`syncLimit * tickTime`，`Leader`认为`Follwer`死掉，从服务器列表中删除`Follwer`。

  * **dataDir**

    zookeeper**保存数据的目录**

  * **dataLogDir**

    保存数据的**日志文件目录**（如果没有配置则默认与保存数据的目录一致）

  * **server.A=B:C:D**

    * **A**：一个数字，表示这个是**第几号服务器**。集群模式下配置一个文件`myid`，这个文件在`dataDir`目录下，这个文件里面有一个数据就是A的值，`Zookeeper`启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个`server`。
    * **B**：**服务器的IP地址**
    * **C**：**服务器与集群中的Leader服务器交换信息的端口**
    * **D**：当集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行**选举时服务器相互通信的端口**。

## **3. 启动**

**启动关闭等脚本在安装路径下的bin目录中**

* **1）启动**

  ```shell
  ./zkServer.sh start
  ```

* **2）关闭**

  ```shell
  ./zkServer.sh stop
  ```

* **3）查看服务状态**

  ```shell
  ./zkServer.sh status
  或
  jps
  ```

* **4）重启**

  ```shell
  ./zkServer.sh restart
  ```

* **5）登录客户端**

  ```shell
  ./zkCli.sh
  ```

# **三、集群搭建**

## **1. 环境准备**

* **1）准备三台服务器**
  * 192.168.245.128
  * 192.168.245.129
  * 192.168.245.130
* **2）每台服务器均按照单机搭建步骤配置好Zookeeper**

## **2. 配置修改**

* **1）修改myid文件内容**

  由于`myid`文件内容表示服务器编号，因此三台服务器编号分别为1,2,3

* **2）修改zoo.cfg配置文件内容**

  在每台服务器的`zoo.cfg`文件中添加如下配置

  ```properties
  server.1=192.168.245.128:2881:3881
  server.2=192.168.245.129:2881:3881
  server.3=192.168.245.130:2881:3881
  ```

## **3. 启动**

* **1）安装顺序依次启动1号、2号和3号服务器**

* **2）查看各服务器状态**

  ```shell
  # 1号服务器
  [root@vm01 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: follower
  
  # 2号服务器
  [root@vm02 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: leader
  
  # 3号服务器
  [root@vm03 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: follower
  ```

  可以看到2号服务器为`leader`，1号和3号为`follower`。

* **3）配置成功！**