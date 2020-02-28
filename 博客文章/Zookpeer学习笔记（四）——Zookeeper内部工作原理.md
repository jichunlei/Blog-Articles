# **一、前言**

## **1. Stat结构体**

* **1）czxid**

  **创建节点的事务zxid**。每次修改`ZooKeeper`状态都会收到一个`zxid`形式的时间戳，也就是`ZooKeeper`事务ID。事务ID是`ZooKeeper`中所有修改总的次序。每个修改都有唯一的`zxid`，如果`zxid1`小于`zxid2`，那么`zxid1`在`zxid2`之前发生。

* **2）ctime - znode**

  **被创建的毫秒数(从1970年开始)**

* **3）mzxid - znode**

  **最后更新的事务zxid**

* **4）mtime - znode**

  **最后修改的毫秒数(从1970年开始)**

* **5）pZxid-znode**

  **最后更新的子节点zxid**

* **6）cversion - znode**

  **子节点变化号，znode子节点修改次数**

* **7）dataversion - znode**

  **数据变化号**

* **8）aclVersion - znode**

  **访问控制列表的变化号**

* **9）ephemeralOwner**

  **如果是临时节点，这个是znode拥有者的session id。如果不是临时节点则是0**

* **10）dataLength**

  **znode的数据长度**

* **11）numChildren**

  **znode子节点数量**

## **2. Zookeeper中角色**

**Zookeeper中的角色主要有以下三类**

* **1）Leader：领导者**

  为客户端提供**读和写**的服务，负责**投票的发起和决议**，**更新系统状态**。

* **2）Learner：学习者**

  Learner包含以下两种角色

  * **Follower：跟随者**

    为客户端提供**读**服务，如果是**写服务则转发给Leader**。在选举过程中**参与投票**。

  * **Observer：观察者**

    为客户端提供**读**服务器，如果是**写服务则转发给Leader**。**不参与选举过程中的投票**，也**不参与“过半写成功”策略**。在不影响写性能的情况下提升集群的读性能。此角色于zookeeper3.3系列新增的角色。

* **3）Client：客户端**

  连接zookeeper服务器的使用者，请求的发起者。独立于zookeeper服务器集群之外的角色。

## 3. 基本术语

* **1）quorum**

  集群中**超过半数的节点集合**

* **2）epoch**

  可以理解为当前集群所处的周期，类似皇帝的年号。

* **3）ZXID**

  - **低32位为counter，高32位为epoch**

  * 每当Leader发生变化的时候，counter=0，epoch+1

* **4）SID**

  **该投票信息所属的serverId**

## **4. ZAB协议**

### **4.1 什么是ZAB协议**

- **Zookeeper Atomic Broadcast** （Zookeeper原子广播）
- `ZAB`协议是为分布式协调服务`Zookeeper`专门设计的一种 **支持崩溃恢复** 的 **原子广播协议** ，是Zookeeper保证数据一致性的核心算法。`ZAB`借鉴了`Paxos`算法，但又不像`Paxos`那样，是一种通用的分布式一致性算法。**它是特别为Zookeeper设计的支持崩溃恢复的原子广播协议**。

### **4.2 ZAB模式**

包含**崩溃恢复（选主）**和**消息广播（同步）**

- **崩溃恢复**
  - 当**服务启动**或者**Leader崩溃**之后，`ZAB`就进入了崩溃恢复模式
  - 当**Leader被选举出来**，且**超过半数Server完成和Leader的同步**之后，恢复模式结束。
- **消息广播**
  - 正常`ZAB`协议工作时会**一直处于**广播模式。
  - 一旦Leader和半数Follower完成**同步完成之后**，就开始进行消息广播，即进入**消息广播**模式。
  - 这个时候，如果有`Server`加入集群，那么这个`Server`就会先进入**崩溃恢复**模式，进行**状态同步**，待状态同步**完成**之后，也**参与消息广播**。
  - `ZooKeeper`一直维持`Broadcast`，直到**Leader崩溃**或者**Leader失去半数Follower**。

### **4.3 ZAB协议四个阶段**

- **1）Leader Election 选举阶段**

  - 这一阶段的主要目的是选出**准Leader**,然后进入下一阶段。**准Leader**节点的保准是：**得到超过半数节点的票数**。
  - 节点处于选举阶段。只有达到广播阶段时，准Leader才会成为真正的Leader。

- **2）Discovery 发现阶段**

  这一阶段的主要目的是：**发现大多数节点接收的最新提案，并且准Leader生成新的epoch，让Follower接受，更新它们的acceptEpoch**

- **3）Synchronization 同步阶段**

  这个阶段的主要任务是：利用`Leader`前一阶段获取的最新提案历史，**同步集群中的所有副本**。只有当`Quorum`都同步完成，准`Leader`才会成为真正的`Leader`。`Follower`只会接受`zxid`比自己的`lastZxid`大的提案。

- **4）Broadcast 广播阶段**

  到这个阶段，ZK集群才**向外提供事务服务**，并且Leader可以进行消息广播。如果有新节点加入，需要**对新节点进行同步**。

### **4.4 基于Java实现的ZAB协议**

基于`Java`实现的`ZAB`跟上面的定义有些不同，选举阶段使用的是 `Fast Leader Election（FLE）`，它包含了第二步的发现职责。因为 `FLE` 会选举拥有最新提议历史的节点作为 `Leader`，这样就省去了发现最新提议的步骤。实际的实现将上述第二步选举阶段和第三步同步阶段合并为 `Recovery Phase`（恢复阶段）。所以，ZAB 的实现只有三个阶段：

- **1）Fast Leader ELection**
- **2）Recovery Phase**
- **3）Broadcast Phase**

# **二、工作原理**

* `Zookeeper`的核心是**原子广播**，这个机制保证了各个`Server`之间的同步。**实现这个机制的协议就是ZAB协议**。ZAB协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。
  * 当服务启动或者在领导者崩溃后，ZAB就进入了恢复模式，
  * 当领导者被选举出来，且大多数Server完成了和 leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。
* 为了**保证事务的顺序一致性**，`zookeeper`采用了**递增的事务id**号（`zxid`）来标识事务。所有的提议（`proposal`）都在被提出的时候加上 了`zxid`。实现中`zxid`是一个64位的数字，它高32位是`epoch`用来标识`leader`关系是否改变，每次一个`leader`被选出来，它都会有一个 新的`epoch`，标识当前属于那个`leader`的统治时期。低32位用于递增计数。

## **1. 选主流程**

**当leader崩溃或者leader失去大多数的follower，这时候zookeeper进入恢复模式**，恢复模式需要重新选举出一个新的leader，让所有的 Server都恢复到一个正确的状态。zookeeper的选举算法有两种：一种是**基于basic paxos实现**的，另外一种是**基于fast paxos算法实现的FastLeaderElection**。**系统默认的选举算法为fast paxos**。接下来重点介绍**FastLeaderElection**选举机制。

### **1.1 相关概念**

* **外部投票**：特指其他服务器发来的投票。
* **内部投票**：服务器自身当前的投票。
* **选举轮次**：Zookeeper服务器Leader选举的轮次，即`logicalclock`（**zxid的前32位，即epoch**），每次投票都会自增。
* **PK**：对**内部投票**和**外部投票**进行对比来确定是否需要变更内部投票。
* **选票管理**
  * **sendqueue**：**选票发送队列**，用于保存待发送的选票。
  *  **recvqueue**：**选票接收队列**，用于保存接收到的外部投票。
  * **WorkerReceiver**：**选票接收器**。其会不断地从`QuorumCnxManager`中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到`recvqueue`中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。
  * **WorkerSender**：**选票发送器**，不断地从`sendqueue`中获取待发送的选票，并将其传递到底层`QuorumCnxManager`中。

### **1.2 基本流程**

* **1）自增选举轮次**

  Zookeeper规定所有有效的投票都**必须在同一轮次**中，在开始新一轮投票时，会首先对`logicalclock`进行**自增操作**。

* **2）初始化选票**

  在开始进行新一轮投票之前，每个服务器都会**初始化自身的选票**，并且在初始化阶段，每台服务器都会**将自己推举为Leader**，票信息为`(SID, ZXID)`。

* **3）发送初始化选票**

  完成选票的初始化后，服务器就会发起第一次投票。Zookeeper会将刚刚初始化好的选票**放入sendqueue**中，由**发送器WorkerSender负责发送**出去。

* **4）接收外部投票**

  每台服务器会不断地**从recvqueue队列中获取外部选票**。如果服务器发现无法获取到任何外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票。

* **5）判断选举轮次**

  在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。

  * **外部投票的选举轮次大于内部投票**：立即**更新自己的选举轮次(logicalclock)**，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。
  * **外部投票的选举轮次小于内部投票**：Zookeeper会直接忽略该外部投票，不做任何处理，并**返回步骤4**。
  * **外部投票的选举轮次等于内部投票**：此时可以开始进行**选票PK**。

* **6）选票PK**

  * 若选举轮次一致，那么就对比两者的ZXID，若外部投票的ZXID大，那么需要变更投票。
  * 若两者的ZXID一致，那么就对比两者的SID，若外部投票的SID大，那么就需要变更投票
  * 其余情况不变更投票

* **7）变更投票**

  经过PK后，若确定了外部投票优于内部投票，那么就变更投票，使用外部投票的选票信息来覆盖内部投票，变更完成后，再次将这个变更后的内部投票发送出去。

* **8）选票归档**

  无论是否变更了投票，都会将刚刚收到的那份外部投票**放入选票集合recvset中进行归档**。`recvset`用于记录当前服务器在本轮次的`Leader`选举中收到的所有外部投票（按照服务队的SID区别，如{(1, vote1), (2, vote2)...}）。

* **9）统计投票**

  统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有**过半服务器认可**了该投票，则终止投票。否则返回步骤4。

* **10）更新服务器状态**

  若已经确定可以终止投票，那么就开始更新服务器状态，服务器首选判断当前被过半服务器认可的投票所对应的Leader服务器是否是自己，若是自己，则将自己的服务器状态更新为LEADING，若不是，则根据具体情况来确定自己是`FOLLOWING`或是`OBSERVING`。

**以上10个步骤就是FastLeaderElection的核心，其中步骤4-9会经过几轮循环，直到有Leader选举产生。**

## **2. 同步流程**

**ZooKeeper选完leader以后，ZooKeeper就进入状态同步过程**

* 1）Leader**等待server连接**;
* 2）`Follower`连接`leader`，将**最大的zxid发送给leader**;
* 3）Leader**根据Follower的zxid确定同步点**；
* 4）完成同步后**通知Follower成为uptodate状态**;
* 5）Follower收到uptodate消息后，又可以**重新接受client的请求进行服务**了。

## **3. 工作流程**

### **3.1 Leader工作流程**

* **基本功能**

  * **恢复数据**；
  * **维持与Learner的心跳**，**接收Learner请求**并判断Learner的请求消息类型；
  * Learner的消息类型主要有**PING消息**、**REQUEST消息**、**ACK消息**、**REVALIDATE消息**，根据不同的消息类型，进行不同的处理
    * **PING消息**：Learner的心跳信息
    * **REQUEST消息**：Follower发送的提议信息，包括写请求及同步请求
    * **ACK消息**： Follower的对提议的回复，超过半数的Follower通过，则commit该提议
    * **REVALIDATE消息**：延长SESSION有效时间

* **流程图**

  ![](http://img.xianzilei.cn/Leader%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

### **3.2 Follower工作流程**

- **基本功能**

  * 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；
  * 接收Leader消息并进行处理；
  * 接收Client的请求，如果为写请求，发送给Leader进行投票；
  * 返回Client结果。

- **Follower的消息循环处理如下几种来自Leader的消息**

  - **PING消息**： 心跳消息；
  - **PROPOSAL消息**：Leader发起的提案，要求Follower投票；
  - **COMMIT消息**：服务器端最新一次提案的信息；
  - **UPTODATE消息**：表明同步完成；
  - **REVALIDATE消息**：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
  - **SYNC消息**：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新

- **流程图**

  ![](http://img.xianzilei.cn/Follower%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

### **3.3 Observer工作流程**

**observer流程和Follower的唯一不同的地方就是observer不会参加leader发起的投票**。