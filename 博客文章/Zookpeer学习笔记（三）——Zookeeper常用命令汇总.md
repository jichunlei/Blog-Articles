# **一、常用shell脚本操作命令**

**shell脚本在zookeeper安装根目录的bin/路径下**

## **1. 启动**

```shell
./zkServer.sh start
```

## **2. 关闭**

```shell
./zkServer.sh stop
```

## **3. 查看服务状态**

```shell
./zkServer.sh status
或
jps
```

## **4. 重启**

```shell
./zkServer.sh restart
```

## **5. 登录客户端**

```shell
./zkCli.sh
```

# **二、常用客户端操作命令**

## **1. 显示所有操作命令**

* **语法**：`help`

  ```shell
  [zk: localhost:2181(CONNECTED) 0] help
  ZooKeeper -server host:port cmd args
  	stat path [watch]
  	set path data [version]
  	ls path [watch]
  	delquota [-n|-b] path
  	ls2 path [watch]
  	setAcl path acl
  	setquota -n|-b val path
  	history 
  	redo cmdno
  	printwatches on|off
  	delete path [version]
  	sync path
  	listquota path
  	rmr path
  	get path [watch]
  	create [-s] [-e] path data acl
  	addauth scheme auth
  	quit 
  	getAcl path
  	close 
  	connect host:port
  ```

## **2. 查看指定路径下包含的节点**

* **语法**：`ls <节点路径>`

* 例：查看`/`路径下包含的节点

  ```shell
  [zk: localhost:2181(CONNECTED) 3] ls /
  [zookeeper]
  ```

## **3. 查看当前节点详细数据**

* **语法**：`ls2 <节点>`

  ```shell
  [zk: localhost:2181(CONNECTED) 4] ls2 /
  [zookeeper]
  cZxid = 0x0
  ctime = Thu Jan 01 08:00:00 CST 1970
  mZxid = 0x0
  mtime = Thu Jan 01 08:00:00 CST 1970
  pZxid = 0x0
  cversion = -1
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 0
  numChildren = 1
  ```

## **4. 创建节点**

* **1）创建普通节点**

  * **语法**：`create <节点> <节点值>`

  * **例**：在`/`路径下创建节点`node1`，值为`value1`

    ```shell
    [zk: localhost:2181(CONNECTED) 5] create /node1 "value1"
    Created /node1
    [zk: localhost:2181(CONNECTED) 6] ls /
    [zookeeper, node1]
    ```

* **2）创建短暂节点（客户端失效后节点就会被删除）**

  * **语法**：`create -e <节点> <节点值>`

  * **例**：在`/`路径下创建短暂节点`node2`，值为`value2`

    ```shell
    #创建短暂节点node2
    [zk: localhost:2181(CONNECTED) 7] create -e /node2 "value2"
    Created /node2
    [zk: localhost:2181(CONNECTED) 8] ls /
    [node2, zookeeper, node1]
    #退出客户端
    [zk: localhost:2181(CONNECTED) 9] quit
    Quitting...
    2020-02-26 18:41:19,635 [myid:] - INFO  [main:ZooKeeper@684] - Session: 0x17080ec139b0000 closed
    2020-02-26 18:41:19,637 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@519] - EventThread shut down for session: 0x17080ec139b0000
    #重新连接客户端
    [root@vm01 bin]# ./zkCli.sh
    #node2已经被删除
    [zk: localhost:2181(CONNECTED) 0] ls /
    [zookeeper, node1]
    ```

* **3）创建带序号的节点**

  * **语法**：`create -s <节点> <节点值>`

  * **例**：在`/node1`路径下创建l两个带序号节点，前缀为`node11`，值为`value11`

    ```shell
    [zk: localhost:2181(CONNECTED) 1] create -s /node1/node11 "value11"
    Created /node1/node110000000000
    [zk: localhost:2181(CONNECTED) 3] create -s /node1/node11 "value11"
    Created /node1/node110000000001
    [zk: localhost:2181(CONNECTED) 4] ls /node1                        
    [node110000000000, node110000000001]
    ```

    **如果原来没有序号节点，序号从0开始依次递增。如果原节点下已有2个节点，则再排序时从2开始，以此类推。**

## **5. 获得节点的值**

* **语法**：`get <节点>`

* **例**：获取节点`/node1`的值

  ```shell
  [zk: localhost:2181(CONNECTED) 5] get /node1                
  value1
  cZxid = 0x200000002
  ctime = Wed Feb 26 18:34:50 CST 2020
  mZxid = 0x200000002
  mtime = Wed Feb 26 18:34:50 CST 2020
  pZxid = 0x200000007
  cversion = 2
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 6
  numChildren = 2
  ```

## **6. 修改节点的值**

* **语法**：`set <节点> <新的节点值>`

* **例**：修改获取节点`/node1`的值为`value01`

  ```shell
  [zk: localhost:2181(CONNECTED) 6] set /node1 "value01"
  cZxid = 0x200000002
  ctime = Wed Feb 26 18:34:50 CST 2020
  mZxid = 0x200000008
  mtime = Wed Feb 26 19:18:20 CST 2020
  pZxid = 0x200000007
  cversion = 2
  dataVersion = 1
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 7
  numChildren = 2
  [zk: localhost:2181(CONNECTED) 7] get /node1          
  value01
  cZxid = 0x200000002
  ctime = Wed Feb 26 18:34:50 CST 2020
  mZxid = 0x200000008
  mtime = Wed Feb 26 19:18:20 CST 2020
  pZxid = 0x200000007
  cversion = 2
  dataVersion = 1
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 7
  numChildren = 2
  ```

## **7. 节点的值变化监听（值变化）**

* **语法**：`get <节点> watch`

* **例**：

  * 1）在128主机上监听节点/node1的数据变化

    ```shell
    [zk: localhost:2181(CONNECTED) 8] get /node1 watch
    value01
    cZxid = 0x200000002
    ctime = Wed Feb 26 18:34:50 CST 2020
    mZxid = 0x200000008
    mtime = Wed Feb 26 19:18:20 CST 2020
    pZxid = 0x200000007
    cversion = 2
    dataVersion = 1
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 7
    numChildren = 2
    ```

  * 2）在129主机上修改节点/node1的值

    ```shell
    [zk: localhost:2181(CONNECTED) 0] set /node1 "xxxxxxx"
    cZxid = 0x200000002
    ctime = Wed Feb 26 18:34:50 CST 2020
    mZxid = 0x20000000a
    mtime = Wed Feb 26 19:31:28 CST 2020
    pZxid = 0x200000007
    cversion = 2
    dataVersion = 2
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 7
    numChildren = 2
    ```

  * 3）观察128主机上的监听

    ```shell
    WATCHER::
    
    WatchedEvent state:SyncConnected type:NodeDataChanged path:/node1
    ```

  * 4）再次在129主机上修改节点/node1的值

    ```shell
    [zk: localhost:2181(CONNECTED) 1] set /node1 "xx"     
    cZxid = 0x200000002
    ctime = Wed Feb 26 18:34:50 CST 2020
    mZxid = 0x20000000b
    mtime = Wed Feb 26 19:33:02 CST 2020
    pZxid = 0x200000007
    cversion = 2
    dataVersion = 3
    aclVersion = 0
    ephemeralOwner = 0x0
    dataLength = 2
    numChildren = 2
    ```

  * 5）128主机上无监听，表示**只监听一次修改，一旦监听到修改后自动关闭监听**。

## **8. 节点的子节点变化监听（路径变化）**

* **语法**：`ls <节点> watch`

* **例**：

  * 1）在128主机上监听节点/node1的子节点变化

    ```shell
    [zk: localhost:2181(CONNECTED) 9] ls /node1 watch
    [node110000000000, node110000000001]
    ```

  * 2）在129主机上新增节点/node1/node12

    ```shell
    [zk: localhost:2181(CONNECTED) 3] create /node1/node12 "value12"
    Created /node1/node12
    ```

  * 3）观察128主机上的监听

    ```shell
    WATCHER::
    
    WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/node1
    
    ```

  * 4）再次在129主机上新增节点/node1/node13

    ```shell
    [zk: localhost:2181(CONNECTED) 4] create /node1/node13 "value13"
    Created /node1/node13
    ```

  * 5）同理128主机上无监听，只监听一次

  * **注意：如果修改子节点的值是不会触发监听的，只会对路径变化敏感**

## **9. 删除节点**

* **1）普通删除（只能删除无子节点的节点）**

  * **语法**：`delete <节点>` 

  * **例**：

    * 删除无子节点的节点，删除成功

      ```shell
      [zk: localhost:2181(CONNECTED) 15] delete /node1/node12
      ```

    * 删除有子节点的节点，删除失败

      ```shell
      [zk: localhost:2181(CONNECTED) 12] delete /node1
      Node not empty: /node1
      ```

* **2）递归删除（删除该节点也会同时删除该节点下的子节点，以此递归）**

  * **语法**：`rmr <节点>`

  * **例**：删除节点`/node1`

    ```shell
    [zk: localhost:2181(CONNECTED) 18] rmr /node1
    ```

## **10. 查看节点状态**

* **语法**：`stat <节点>`

* **例**：查看节点`/node1`状态

  ```shell
  [zk: localhost:2181(CONNECTED) 17] stat /node1
  cZxid = 0x200000002
  ctime = Wed Feb 26 18:34:50 CST 2020
  mZxid = 0x20000000b
  mtime = Wed Feb 26 19:33:02 CST 2020
  pZxid = 0x200000010
  cversion = 5
  dataVersion = 3
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 2
  numChildren = 3
  ```

## **11. 四字命令**

`ZooKeeper` 支持某些特定的**四字命令字母**与其的交互。它们大多是**查询命令**，用来获取 `ZooKeeper` 服务的**当前状态及相关信息**。用户在客户端可以通过 `telnet`或 `nc` 向 `ZooKeeper` 提交相应的命令。

* **四字命令如下**

  |    类别    | 命令   | 描述                                                         |
  | :--------: | ------ | ------------------------------------------------------------ |
  | 服务器状态 | `ruok` | 测试服务是否处于正确状态。如果服务器正在运行且未出错，则返回“imok ”，否则返回空。 |
  | 服务器状态 | `conf` | 输出相关服务配置的详细信息(根据配置文件zoo.cfg)。3.3.0版本引入。 |
  | 服务器状态 | `envi` | 输出关于服务环境的详细信息（区别于 conf命令），包括Zookeeper版本，Java版本和其他系统属性。 |
  | 服务器状态 | `srvr` | 输出服务器的统计信息，包括延迟统计，znode数量和服务器运行模式(standalone/leader/followewr)。3.3.0版本引入。 |
  | 服务器状态 | `stat` | 输出的统计信息和已连接的客户端。                             |
  | 服务器状态 | `srst` | 重置服务器的统计                                             |
  | 服务器状态 | `isro` | 显示服务器所属模式，只读（ro）模式或读写（rw）模式           |
  | 客户端连接 | `dump` | 列出集合中所有会话和临时znode。这个命令只能在leader节点上有用。参考srvr命令。 |
  | 客户端连接 | `cons` | 列出所有连接到这台服务器的客户端详细信息。包括”接受/发送”的包数量、会话id、操作延迟、最后的操作执行等。3.3.0版本引入。 |
  | 客户端连接 | `crst` | 重置所有连接统计信息。3.3.0版本引入。                        |
  |    观察    | `wchs` | 列出服务器上所有watch的详细信息。3.3.0版本引入。             |
  |    观察    | `wchc` | 通过session列出服务器watch的详细信息，它的输出是一个与watch相关的会话的列表。 注意：如果watch数量较多，此命令会影响服务器的性能。3.3.0版本引入。 |
  |    观察    | `wchp` | 通过znode路径列出服务器上所有watch。它输出一个与session相关的路径。 注意：如果watch数量较多，此命令会影响服务器的性能。3.3.0版本引入。 |
  |    监控    | `mntr` | mntr	按Java属性格式列出服务器的统计信息，输出可用于检测集群健康状态的变量列表。适合用做Ganglia和Nagios等监控系统的信息源。3.4.0版本引入.。 注：除了mntr外，还可以通过JMX来提供Zookeeper服务器的统计信息。在安装目录src/contrib子目录下提供了相关监控工具和方法 |

* **使用方式**

  * **telnet方式**

    * **1）执行命令连接服务器**

      ```shell
      telnet <ip地址> <端口号>
      ```

      如果出现"-bash: telnet: 未找到命令"这种提示，需要安装telnet

      ```shell
      yum install telnet
      ```

    * **2）输入四字命令**

    * **3）例**

      ```shell
      [root@vm01 bin]# telnet 192.168.245.128 2181
      Trying 192.168.245.128...
      Connected to 192.168.245.128.
      Escape character is '^]'.
      # 输入conf四字命令查看配置信息
      conf
      clientPort=2181
      dataDir=/usr/local/zookeeper/zookeeper-3.4.10/data/version-2
      dataLogDir=/usr/local/zookeeper/zookeeper-3.4.10/log/version-2
      tickTime=2000
      maxClientCnxns=60
      minSessionTimeout=4000
      maxSessionTimeout=40000
      serverId=1
      initLimit=10
      syncLimit=5
      electionAlg=3
      electionPort=3881
      quorumPort=2881
      peerType=0
      Connection closed by foreign host.
      ```

  * **nc(netcat)方式**

    * **语法**：`echo <四字命令> | nc <ip地址> <端口号>`

      如果提示"-bash: nc: 未找到命令"，同理需要安装nc

      ```shell
      yum install nc
      ```

    * **例**

      ```shell
      [root@vm01 bin]# echo conf |nc 192.168.245.128 2181
      clientPort=2181
      dataDir=/usr/local/zookeeper/zookeeper-3.4.10/data/version-2
      dataLogDir=/usr/local/zookeeper/zookeeper-3.4.10/log/version-2
      tickTime=2000
      maxClientCnxns=60
      minSessionTimeout=4000
      maxSessionTimeout=40000
      serverId=1
      initLimit=10
      syncLimit=5
      electionAlg=3
      electionPort=3881
      quorumPort=2881
      peerType=0
      ```

  * **直接连接客户端输入命令**