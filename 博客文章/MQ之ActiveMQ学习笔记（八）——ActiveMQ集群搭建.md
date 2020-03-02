# **一、为什么要使用集群**

* 实现**高可用**，以排除单点故障引起的服务中断。
* 实现**负载均衡**，以提升效率为更多的客户提供服务。

# **二、ActiveMQ集群部署方式**

## **1. 失效转移连接（了解）**

* 顾名思义，即消费者连接到集群中的某一个broker，当该broker宕机无法访问时，消费者会**自动连接到另一个正常的broker**。故称为失效转移。
* 消费者使用**failover 协议**来连接broker，通常叫做 **失效转移（也叫故障转移，断线重连机制，FailOver）策略**

## **2. Broker Clusters 部署（了解）**

* **各个broker通过网络互相连接**，并共享queue，保证消息同步。因此可以**解决负载均衡的问题，提高消息处理能力**。
* 各个broker进行消息同步使用的是**NetworkConnection (网络连接器)**，主要用于配置各个broker之间的网络通讯方式，用于服务器传递信息。 分为**静态连接器**和**动态连接器**。
  * **静态连接器**：就是将所有已知的broker uri连接时手工进行配置，对client端uri地址做相应修改
  * **动态连接器**：部署前不需要知道所有MQ实例的uri地址，只要进行相关配置，启动后让MQ自己检测

## **3. Master Slave 部署（主从）（重点）**

**该方式通过部署多个broker实例，同时选举产生一个Master和多个Slave，Master宕机时Slave接管服务，从而实现系统的高可用**。主要包括下面两种配置方案。

### **3.1 Share storage master slave(共享存储)（了解）**

此方案中`Master`和`Slave`的数据是**共享**的（相当于共享同一个数据库），当`Master`失效后，`slave`会自动**接管服务**，**无需手动进行数据的copy与同步**，因为`Master`存储数据之后，这些**数据在任何时候对slave都是可见的**。`Master`与`Slave`之间，**通过共享文件的“排他锁”或者分布式排他锁(ZooKeeper)来决定Broker的状态与角色**，获取锁权限的Broker作为Master，其它的Broker则作为Slave。如果Master失效，它必将失去锁权限，那么其它的Slave将通过锁竞争来选举新Master，没有获取锁权限的Broker作为Slave，并等待锁的释放（间歇性尝试获取锁）。该方案包括：**Shared File System Master slave** 和 **JDBC Store Master Slave** 两种模式。

* **Shared File System Master Slave**：这种方式是最常用的模式，架构简单，可靠实用。我们只需要一个SAN文件系统即可。使用文件系统来共享数据文件，多个Broker共享同一个文件系统。但是只适合单台主机部署，不适合多台主机部署
* **JDBC Store Master Slave**：使用数据库（MySQL/Oracle等）存储数据，适合多台主机部署，但是低效。

### **3.2 Replicated LevelDB Store（使用ZooKeeper协调多个Broker）（重点）**

**基于复制的LevelDB Store模式是ActiveMQ 5.9以后新增的特性，这是ActiveMQ全力打造的HA存储引擎。 一般都使用这种方式。由于利用ZooKeeper进行配置管理，可以方便监控，同时配置也相对简单**。

* 使用ZooKeeper（集群）注册所有的ActiveMQ Broker。只有其中的一个Broker可以对外提供服务（也就是Master节点），其他的Broker处于待机状态，被视为Slave。如果Master因故障而不能提供服务，则利用ZooKeeper的内部选举机制会从Slave中选举出一个Broker充当Master节点，继续对外提供服务。

## **4. Master Slave和Broker Cluster结合使用（常用方式）（了解）**

**Master-Slave的方式虽然能解决多服务热备的高可用问题，但无法解决负载均衡和分布式的问题**。**Broker Cluster的部署方式刚好可以解决负载均衡的问题。一般两者结合使用。**

| **集群部署方式**   | **高可用** | **负载均衡** |
| ------------------ | ---------- | ------------ |
| **Master Slave**   | 是         | 否           |
| **Broker Cluster** | 否         | 是           |

`Master Slave`和`Broker Cluster` 结合使用可以实现`高可用`和`负载均衡`，如下图：
按 A->B->C 顺序启动节点服务。

![](http://img.xianzilei.cn/Master%20Slave%E5%92%8CBroker%20Cluster%E7%BB%93%E5%90%88%E9%9B%86%E7%BE%A4%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

这个集群是综合了**Broker Cluster**和**Master/Slave**两种基本集群方式，其中Master/Slave（B和C）是基于共享存储实现的。**A和B、A和C组成消息同步**是为实现**负载均衡**，**B和C组成Master/Slave**是为了实现**高可用**。

* 如果A宕机，集群退化成标准Master/Slave集群，只是了失去负载均衡能力。

* 如果B宕机，C会继续提供服务，集群退化成Broker Cluster集群，失去高可用能力。

* 如果C宕机也会失去高可用能力（同B）。

**ABC无论哪一台宕机，集群都不会崩溃，但是需要迅速恢复**。

# **三、基于ZooKeeper+Replicated LevelDB Store部署案例（重点）**

**准备三台服务器，IP分别为192.168.245.128、192.168.245.129和192.168.245.130，分别部署三个ZooKeeper和ActiveMQ实例。**

* **Zookeeper版本：3.4.10**
* **ActiveMQ版本：5.15.9**

## **1.  Zookeeper集群配置**

* **1）环境规划**

  |   **主机IP**    | **服务端口（默认）** | **集群通信端口（默认）** | 集群选举的端口（默认） |              **节点目录**               |
  | :-------------: | :------------------: | :----------------------: | :--------------------: | :-------------------------------------: |
  | 192.168.245.128 |         2181         |           2881           |          3881          | `/usr/local/zookeeper/zookeeper-3.4.10` |
  | 192.168.245.129 |         2181         |           2881           |          3881          | `/usr/local/zookeeper/zookeeper-3.4.10` |
  | 192.168.245.130 |         2181         |           2881           |          3881          | `/usr/local/zookeeper/zookeeper-3.4.10` |

* **2）配置修改**

  * **1号主机（192.168.245.128）**

    * **zoo.cfg**

      ```properties
      #数据和日志目录配置
      dataDir=/usr/local/zookeeper/zookeeper-3.4.10/data
      dataLogDir=/usr/local/zookeeper/zookeeper-3.4.10/log
      
      #集群信息配置
      server.1=192.168.245.128:2881:3881
      server.2=192.168.245.129:2881:3881
      server.3=192.168.245.130:2881:3881
      ```

    * **myid**

      ```shell
      # 在上述配置的dataDir目录下创建文件myid
      mkdir /usr/local/zookeeper/zookeeper-3.4.10/data/myid
      # 由于1号主机（192.168.245.128）是server.1，所以在myid中设置数字1
      ```

  * **2号主机（192.168.245.129）**

    * **zoo.cfg**：配置同上
    * **myid**：设置数字2

  * **3号主机（192.168.245.130）**

    * **zoo.cfg**：配置同上
    * **myid**：设置数字3

* **3）启动集群**

  分别启动1号、2号和3号主机的`Zookeeper`服务，查看`Zookeeper`服务状态

  ```shell
  # 1号主机
  [root@vm01 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: follower
  
  # 2号主机
  [root@vm02 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: leader
  
  # 3号主机
  [root@vm03 bin]# ./zkServer.sh status
  ZooKeeper JMX enabled by default
  Using config: /usr/local/zookeeper/zookeeper-3.4.10/bin/../conf/zoo.cfg
  Mode: follower
  ```

  **可以看到2号主机为leader，启动成功！**

## **2. ActiveMQ集群配置**

- **1）环境规划**

  | **主机IP**      | **服务端口（默认）** | ****复制协议端口（动态）**** | **jetty控制台端口（默认）** | **节点目录**                             |
  | --------------- | -------------------- | ---------------------------- | --------------------------- | ---------------------------------------- |
  | 192.168.245.128 | 61616                | `tcp://0.0.0.0:0`            | 8161                        | `/usr/local/java/apache-activemq-5.15.9` |
  | 192.168.245.129 | 61616                | `tcp://0.0.0.0:0`            | 8161                        | `/usr/local/java/apache-activemq-5.15.9` |
  | 192.168.245.130 | 61616                | `tcp://0.0.0.0:0`            | 8161                        | `/usr/local/java/apache-activemq-5.15.9` |

- **2）配置修改**

  - **1号主机（192.168.245.128）**

    - **activemq.xml**

      ```xml
      <!-- 持久化配置为replicatedLevelDB-->  
      <persistenceAdapter>
          <replicatedLevelDB  
                 directory="${activemq.data}/leveldb"  
                 replicas="3"  
                 bind="tcp://0.0.0.0:0"                                zkAddress="192.168.245.128:2181,192.168.245.129:2181,192.168.245.130:2181" 
                 zkPath="/activemq/leveldb-stores"  
                 hostname="192.168.245.128"  
           />
      </persistenceAdapter>
      ```

      **配置说明**：

      * **directory**：存储数据的路径
      * **replicas**：集群中的节点数【(replicas/2)+1公式表示集群中至少要正常运行的服务数量】，3台集群那么允许1台宕机， 另外两台要正常运行
      * **bind**：当该节点成为master后，它将绑定已配置的地址和端口来为复制协议提供服务。还支持使用动态端口。只需使用`tcp://0.0.0.0:0`进行配置即可，默认端口为**61616**。
      * **zkPassword**：当连接到`ZooKeeper`服务器时用的密码，没有密码则不配置。
      * **zkAddress**：`ZooKeeper`的`ip`和`port`， 如果是集群，则用逗号隔开(这里作为简单示例`ZooKeeper`配置为单点， 这样已经适用于大多数环境了， 集群也就多几个配置) 
      * **zkPath**：`ZooKeeper`选举信息交换的存贮路径，启动服务后`actimvemq`会到`zookeeper`上注册生成此路径 
      * **hostname**：`ActiveMQ`所在主机的IP

  - **2号主机（192.168.245.129）**

    - **activemq.xml**

      ```xml
      <!-- 持久化配置为replicatedLevelDB-->  
      <persistenceAdapter>
          <replicatedLevelDB  
                 directory="${activemq.data}/leveldb"  
                 replicas="3"  
                 bind="tcp://0.0.0.0:0"                                zkAddress="192.168.245.128:2181,192.168.245.129:2181,192.168.245.130:2181" 
                 zkPath="/activemq/leveldb-stores"  
                 hostname="192.168.245.129"  
           />
      </persistenceAdapter>
      ```

      

  - **3号主机（192.168.245.130）**

    - **activemq.xml**

      ```xml
      <!-- 持久化配置为replicatedLevelDB-->  
      <persistenceAdapter>
          <replicatedLevelDB  
                 directory="${activemq.data}/leveldb"  
                 replicas="3"  
                 bind="tcp://0.0.0.0:0"                                zkAddress="192.168.245.128:2181,192.168.245.129:2181,192.168.245.130:2181" 
                 zkPath="/activemq/leveldb-stores"  
                 hostname="192.168.245.130"  
           />
      </persistenceAdapter>
      ```

- **3）启动集群**

  * **分别启动1号、2号和3号主机的activemq服务**

    ```shell
    # 1号主机
    [root@vm01 bin]# cd /usr/local/java/apache-activemq-5.15.9/bin/
    [root@vm01 bin]# ./activemq start
    INFO: Loading '/usr/local/java/apache-activemq-5.15.9//bin/env'
    INFO: Using java '/usr/local/java/jdk1.8.0_144/bin/java'
    INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
    INFO: pidfile created : '/usr/local/java/apache-activemq-5.15.9//data/activemq.pid' (pid '1728')
    
    # 2号主机
    [root@vm02 bin]# cd /usr/local/java/apache-activemq-5.15.9/bin/
    [root@vm02 bin]# ./activemq start
    INFO: Loading '/usr/local/java/apache-activemq-5.15.9//bin/env'
    INFO: Using java '/usr/local/java/jdk1.8.0_144/bin/java'
    INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
    INFO: pidfile created : '/usr/local/java/apache-activemq-5.15.9//data/activemq.pid' (pid '1732')
    
    # 3号主机
    [root@vm03 bin]# cd /usr/local/java/apache-activemq-5.15.9/bin/
    [root@vm03 bin]# ./activemq start
    INFO: Loading '/usr/local/java/apache-activemq-5.15.9//bin/env'
    INFO: Using java '/usr/local/java/jdk1.8.0_144/bin/java'
    INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
    INFO: pidfile created : '/usr/local/java/apache-activemq-5.15.9//data/activemq.pid' (pid '1639')
    ```

  * **查看启动状态**

    ```shell
    # 1号主机
    [root@vm01 bin]#  ps -ef|grep activemq 
    root       1728      1  3 20:21 pts/0    00:00:08 /usr/local/java/jdk1.8.0_144/bin/java -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dcom.sun.management.jmxremote -Djava.awt.headless=true -Djava.io.tmpdir=/usr/local/java/apache-activemq-5.15.9//tmp -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data -jar /usr/local/java/apache-activemq-5.15.9//bin/activemq.jar start
    root       1787   1319  0 20:26 pts/0    00:00:00 grep --color=auto activemq
    
    # 2号主机
    [root@vm02 bin]#  ps -ef|grep activemq 
    root       1732      1  2 20:23 pts/0    00:00:04 /usr/local/java/jdk1.8.0_144/bin/java -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dcom.sun.management.jmxremote -Djava.awt.headless=true -Djava.io.tmpdir=/usr/local/java/apache-activemq-5.15.9//tmp -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data -jar /usr/local/java/apache-activemq-5.15.9//bin/activemq.jar start
    root       1755   1329  0 20:26 pts/0    00:00:00 grep --color=auto activemq
    
    # 3号主机
    [root@vm03 bin]#  ps -ef|grep activemq 
    root       1639      1  2 20:23 pts/0    00:00:04 /usr/local/java/jdk1.8.0_144/bin/java -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dcom.sun.management.jmxremote -Djava.awt.headless=true -Djava.io.tmpdir=/usr/local/java/apache-activemq-5.15.9//tmp -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data -jar /usr/local/java/apache-activemq-5.15.9//bin/activemq.jar start
    root       1663   1343  0 20:26 pts/0    00:00:00 grep --color=auto activemq
    ```

    **均启动成功！**

  * **查看ZooKeeper是否注册了ActiveMQ节点**

    ```shell
    #连接任意一台zk客户端查看节点信息
    [zk: localhost:2181(CONNECTED) 0] ls /
    [activemq, zookeeper]
    [zk: localhost:2181(CONNECTED) 1] ls /activemq
    [leveldb-stores]
    [zk: localhost:2181(CONNECTED) 2] ls /activemq/leveldb-stores
    [00000000002, 00000000000, 00000000001]
    ```

  * **查看各节点详细信息**

    ```shell
    # 00000000000节点
    [zk: localhost:2181(CONNECTED) 5] get /activemq/leveldb-stores/00000000000
    {"id":"localhost","container":null,"address":"tcp://192.168.245.128:37221","position":-1,"weight":1,"elected":"0000000000"}
    cZxid = 0x50000002c
    ctime = Mon Mar 02 20:21:46 CST 2020
    mZxid = 0x500000035
    mtime = Mon Mar 02 20:23:14 CST 2020
    pZxid = 0x50000002c
    cversion = 0
    dataVersion = 4
    aclVersion = 0
    ephemeralOwner = 0x3709acc619c0002
    dataLength = 123
    numChildren = 0
    
    # 00000000001节点
    [zk: localhost:2181(CONNECTED) 6] get /activemq/leveldb-stores/00000000001
    {"id":"localhost","container":null,"address":null,"position":-1,"weight":1,"elected":null}
    cZxid = 0x500000030
    ctime = Mon Mar 02 20:23:13 CST 2020
    mZxid = 0x500000033
    mtime = Mon Mar 02 20:23:13 CST 2020
    pZxid = 0x500000030
    cversion = 0
    dataVersion = 2
    aclVersion = 0
    ephemeralOwner = 0x2709acc0b440001
    dataLength = 90
    numChildren = 0
    
    # 00000000002节点
    [zk: localhost:2181(CONNECTED) 7] get /activemq/leveldb-stores/00000000002
    {"id":"localhost","container":null,"address":null,"position":-1,"weight":1,"elected":null}
    cZxid = 0x500000037
    ctime = Mon Mar 02 20:23:22 CST 2020
    mZxid = 0x500000037
    mtime = Mon Mar 02 20:23:22 CST 2020
    pZxid = 0x500000037
    cversion = 0
    dataVersion = 0
    aclVersion = 0
    ephemeralOwner = 0x3709acc619c0003
    dataLength = 90
    numChildren = 0
    ```

    **00000000000节点的elected不为null的节点为master节点，其余为follower**

## **3. 集群可用性测试**

* **测试ActiveMQ控制台是否可访问**

  **因为ActiveMQ的客户端只能访问Master的Broker，其他处于Slave的Broker不能访问。前面的master节点为00000000000节点，对应的主机ip为192.168.245.128，故只能访问128主机的控制台界面，其余主机无法访问**。

  ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-01.png)

  ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-02.png)

  ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-03.png)

* **测试集群是否可以正常工作**

  * **1）生产者代码**

    **后续需要验证集群的高可用性，故url配置成失效转移形式**，如`failover:(tcp://192.168.245.128:61616,tcp://192.168.245.129:61616,tcp://192.168.245.130:61616)?randomize=false`。

    ```java
    package com.jicl.queue;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    
    public class JMSProduce {
    
        //activeMQ的url（失效转移）
        public static final String DEFAULT_BROKER_URL = "failover:(tcp://192.168.245.128:61616,tcp://192.168.245.129:61616,tcp://192.168.245.130:61616)?randomize=false";
        //队列名
        public static final String QUEUE_NAME = "cluster-queue-01";
    
        public static void main(String[] args) throws JMSException {
            //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
            ActiveMQConnectionFactory activeMQConnectionFactory =
                    new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
            //2. 通过前面获得的连接工厂获取连接connection
            Connection connection = activeMQConnectionFactory.createConnection();
            connection.start();
            //3. 创建会话session（第一个参数：事务；第二个参数：签收）
            Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            //4. 创建目的地（队列Queue或主题Topic）,此案例是队列
            Queue queue = session.createQueue(QUEUE_NAME);
            // 5. 创建消息的生产者
            MessageProducer messageProducer = session.createProducer(queue);
            //6. 生产消息
            for (int i = 1; i <= 5; i++) {
                //7. 创建消息
                TextMessage textMessage = session.createTextMessage("cluster-queue-msg-" + i);
                //8. 通过messageProducer发送给mq
                messageProducer.send(textMessage);
            }
            //9. 关闭资源
            //关闭生产者
            messageProducer.close();
            //关闭会话
            session.close();
            //关闭连接
            connection.close();
            System.out.println(">>>消息发布成功！");
        }
    }
    ```

  * **2）消费者代码**

    **url同样需要配置成失效转移形式**

    ```java
    package com.jicl.queue;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    import java.io.IOException;
    
    public class JMSConsumer {
    
        //activeMQ的url
        public static final String DEFAULT_BROKER_URL = "failover:(tcp://192.168.245.128:61616,tcp://192.168.245.129:61616,tcp://192.168.245.130:61616)?randomize=false";
        //队列名
        public static final String QUEUE_NAME = "cluster-queue-01";
    
        public static void main(String[] args) throws JMSException {
            //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
            ActiveMQConnectionFactory activeMQConnectionFactory =
                    new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
            //2. 通过前面获得的连接工厂获取连接connection
            Connection connection = activeMQConnectionFactory.createConnection();
            connection.start();
            //3. 创建会话session（第一个参数：事务；第二个参数：签收）
            Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            //4. 创建目的地（队列Queue或主题Topic）,此案例是队列
            Queue queue = session.createQueue(QUEUE_NAME);
            // 5. 创建消息的消费者
            MessageConsumer messageConsumer = session.createConsumer(queue);
            //6. 消费消息：消费方式一--同步阻塞方式（receive()或receive(timeout)）
            while (true){
                TextMessage textMessage = (TextMessage)messageConsumer.receive();
                //TextMessage textMessage = (TextMessage)messageConsumer.receive(10000L);
                if(textMessage!=null){
                    System.out.println(">>>消费者接受到消息："+textMessage.getText());
                }else {
                    break;
                }
            }
            //7. 关闭资源
            //关闭生产者
            messageConsumer.close();
            //关闭会话
            session.close();
            //关闭连接
            connection.close();
        }
    }
    ```

  * **3）启动生产者**

    * IDEA控制台

      ```tex
      16:43:52.575 [ActiveMQ Task-1] INFO org.apache.activemq.transport.failover.FailoverTransport - Successfully connected to tcp://192.168.245.128:61616
      >>>消息发布成功！
      ```

      **成功连接到192.168.245.128服务器，消息发送成功！**

    * **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-04.png)

      **消息发送成功！**

  * **4）启动消费者**

    * **IDEA控制台**

      ```tex
      17:06:29.160 [ActiveMQ Task-1] INFO org.apache.activemq.transport.failover.FailoverTransport - Successfully connected to tcp://192.168.245.128:61616
      >>>消费者接受到消息：cluster-queue-msg-1
      >>>消费者接受到消息：cluster-queue-msg-2
      >>>消费者接受到消息：cluster-queue-msg-3
      >>>消费者接受到消息：cluster-queue-msg-4
      >>>消费者接受到消息：cluster-queue-msg-5
      ```

      **成功连接到192.168.245.128服务器，消息消费成功！**

    * **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-05.png)

      **消息消费成功！**

* **MQ的master节点宕机后情况**

  * **1）手动停止128服务器MQ节点（模拟宕机）**

    ```shell
    [root@vm01 ~]# ps -ef|grep activemq
    root       1705      1  1 00:40 pts/0    00:00:31 /usr/local/java/jdk1.8.0_144/bin/java -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dcom.sun.management.jmxremote -Djava.awt.headless=true -Djava.io.tmpdir=/usr/local/java/apache-activemq-5.15.9//tmp -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data -jar /usr/local/java/apache-activemq-5.15.9//bin/activemq.jar start
    root       1873   1854  0 01:10 pts/1    00:00:00 grep --color=auto activemq
    [root@vm01 ~]# kill -9 1705
    [root@vm01 ~]# ps -ef|grep activemq
    root       1876   1854  0 01:11 pts/1    00:00:00 grep --color=auto activemq
    ```

  * **2）启动生产者**

    - IDEA控制台

      ```tex
      17:13:59.560 [ActiveMQ Task-1] DEBUG org.apache.activemq.transport.failover.FailoverTransport - Attempting 0th connect to: tcp://192.168.245.128:61616
      17:14:01.576 [ActiveMQ Task-1] DEBUG org.apache.activemq.transport.failover.FailoverTransport - Connect fail to: tcp://192.168.245.128:61616, reason: {}
      17:14:01.582 [ActiveMQ Task-1] INFO org.apache.activemq.transport.failover.FailoverTransport - Successfully connected to tcp://192.168.245.129:61616
      >>>消息发布成功！
      ```

      **连接192.168.245.128服务器失败，成功连接到192.168.245.129服务器，消息发送成功！**

    - **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-06.png)

      **由于master自动切换到了129服务器，消息发送成功！**

  * **4）启动消费者**

    - **IDEA控制台**

      ```tex
      17:24:00.569 [ActiveMQ Task-1] DEBUG org.apache.activemq.transport.failover.FailoverTransport - Attempting 0th connect to: tcp://192.168.245.128:61616
      17:24:02.576 [ActiveMQ Task-1] DEBUG org.apache.activemq.transport.failover.FailoverTransport - Connect fail to: tcp://192.168.245.128:61616, reason: {}
      17:24:02.582 [ActiveMQ Task-1] INFO org.apache.activemq.transport.failover.FailoverTransport - Successfully connected to tcp://192.168.245.129:61616
      >>>消费者接受到消息：cluster-queue-msg-1
      >>>消费者接受到消息：cluster-queue-msg-2
      >>>消费者接受到消息：cluster-queue-msg-3
      >>>消费者接受到消息：cluster-queue-msg-4
      >>>消费者接受到消息：cluster-queue-msg-5
      ```

      **同上，消息消费成功！**

    - **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/ActiveMQ%E9%9B%86%E7%BE%A4%E6%A1%88%E4%BE%8B%E5%9B%BE-07.png)

      **由于master自动切换到了129服务器，消息消费成功！**

## **4. 案例总结**

* **ActiveMQ的客户端只能访问Master的Broker**，其他处于`Slave`的`Broker`不能访问，所以客户端连接的`Broker`应该使用**failover协议（失败转移）**
* 当一个`ActiveMQ`节点挂掉或者一个`Zookeeper`节点挂点，`ActiveMQ`服务依然正常运转，如果仅剩一个`ActiveMQ`节点，由于不能选举`Master`，所以`ActiveMQ`不能正常运行（以上述三个节点为例）。
* 同样的，如果`zookeeper`仅剩一个活动节点，不管`ActiveMQ`各节点存活，`ActiveMQ`也不能正常提供服务（以上述三个节点为例）。**即ActiveMQ集群的高可用依赖于Zookeeper集群的高可用**。