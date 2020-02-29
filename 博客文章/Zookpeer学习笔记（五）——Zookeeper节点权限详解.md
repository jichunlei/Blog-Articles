# **一、前言**

## **1. ACL**

### **1.1 什么是ACL**

* **全称：Access Control List（访问控制列表）**
* 用于控制资源的**访问权限**
* `ACL`实现与`UNIX`文件访问权限非常相似：**它使用权限位来允许/禁止针对节点的各种操作以及位应用的范围**
* `ZooKeeper`使用`ACL`来控制对其`znode`（`ZooKeeper`数据树的数据节点）的访问，如节点数据读写、节点创建、节点删除、读取子节点列表、设置节点权限等。

### **1.2 Zookeeper的ACL与传统的文件系统对比**

* **传统的文件系统中**：一个文件拥有某个组的权限即拥有了组里的所有权限，文件或子目录默认会继承自父目录的ACL
* **Zookeeper的ACL**：`znode`的ACL是没有继承关系的，每个`znode`的权限都是独立控制的，只有客户端满足`znode`设置的权限要求时，才能完成相应的操作。

### **1.3 Zookeeper中的ACL构成**

**Zookeeper的ACL，分为以下三个维度**

* **1）scheme**：授权策略
* **2）id**：用户
* **3）permission**：权限

权限信息通常表示为：`scheme:id:permission`，二、三和四章将详细讨论每部分的含义。

## **2. ZooKeeper的ACL支持的权限**

|      | **操作** | **描述**                          |
| ---- | -------- | --------------------------------- |
| c    | CREATE   | 创建一个子节点                    |
| r    | READ     | 获取子节点列表和与znode相关的数据 |
| w    | WRITE    | 将数据设置（写入）到znode         |
| d    | DELETE   | 删除一个孩子子节点                |
| a    | ADMIN    | 设置ACL（权限）                   |

**一般使用第一个字母表示，例如cdrwa**

## **3. ZooKeeper客户端权限设置操作**

* **查看权限**

  ```shell
  getAcl <节点>
  ```

* **设置权限**

  ```shell
  setAcl <节点> <权限信息>
  ```

* **添加认证用户**

  ```she
  addauth <授权策略> <用户名>:<密码>
  ```

* **创建Znode时同时指定ACL 权限**

  ```shell
  create <节点> <节点表达式> <权限信息>
  ```

# **二、scheme（授权策略）**

scheme即**授权策略**。**每种授权策略对应不同的权限校验方式**，`ZooKeeper`常用的几种scheme如下所示：

## **1. world**

* **默认方式，表示无权限（全世界都可以访问）**

* **语法**：`world:anyone:cdrwa`

* **例**：创建节点默认的scheme

  ```shell
  [zk: localhost:2181(CONNECTED) 2] create /node1 "value1"
  Created /node1
  [zk: localhost:2181(CONNECTED) 3] getAcl /node1         
  'world,'anyone
  : cdrwa
  ```

## **2. digest**

* 即**用户名:密码这种方式认证**，这也是业务系统中最常用的。使用`username:password`字符串生成**MD5哈希**，然后将其用作ACL的**ID标识**。

* **语法**：`digest:username:BASE64(SHA1(password)):cdrwa`

  * `digest`：是授权方式

  * `username:BASE64(SHA1(password))`：id部分，是**用户名和密码做sha1加密再做BASE64加密后的组合**

  * `cdrwa`：权限部份

  * 密码加密需要用到`ZooKeeper`的一个工具类来生成（我的`zookpeer`默认安装在`/usr/local/zookeeper`目录下）

    ```shell
    #以xianzilei:123456为例
    # 执行命令
    java -Djava.ext.dirs=/usr/local/zookeeper/zookeeper-3.4.10/lib -cp /usr/local/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider xianzilei:123456
    # 返回结果
    xianzilei:123456->xianzilei:KO4p79djkmRWcuTkp42PqkzXKRo=
    ```

* **例**：使用上述加密的密码为例

  ```shell
  [zk: localhost:2181(CONNECTED) 0] ls /
  [zookeeper]
  # 创建节点/node1
  [zk: localhost:2181(CONNECTED) 1] create /node1 111
  Created /node1
  # 设置digest类型权限
  [zk: localhost:2181(CONNECTED) 2] setAcl /node1 digest:xianzilei:KO4p79djkmRWcuTkp42PqkzXKRo=:cdrwa
  cZxid = 0x400000009
  ctime = Sat Feb 29 20:23:22 CST 2020
  mZxid = 0x400000009
  mtime = Sat Feb 29 20:23:22 CST 2020
  pZxid = 0x400000009
  cversion = 0
  dataVersion = 0
  aclVersion = 1
  ephemeralOwner = 0x0
  dataLength = 3
  numChildren = 0
  # 查看是否设置成功，设置成功
  [zk: localhost:2181(CONNECTED) 3] getAcl /node1
  'digest,'xianzilei:KO4p79djkmRWcuTkp42PqkzXKRo=
  : cdrwa
  # 未获取权限的情况下查看节点，访问被拒绝
  [zk: localhost:2181(CONNECTED) 4] get /node1
  Authentication is not valid : /node1
  # 未获取权限的情况下新建子节点，被拒绝
  [zk: localhost:2181(CONNECTED) 5] create /node1/node11 2222 
  Authentication is not valid : /node1/node11
  # 当前session添加授权信息
  [zk: localhost:2181(CONNECTED) 6] addauth digest xianzilei:123456
  # 再次查看节点，访问成功
  [zk: localhost:2181(CONNECTED) 7] get /node1                     
  111
  cZxid = 0x400000009
  ctime = Sat Feb 29 20:23:22 CST 2020
  mZxid = 0x400000009
  mtime = Sat Feb 29 20:23:22 CST 2020
  pZxid = 0x400000009
  cversion = 0
  dataVersion = 0
  aclVersion = 1
  ephemeralOwner = 0x0
  dataLength = 3
  numChildren = 0
  # 新建子节点，创建成功
  [zk: localhost:2181(CONNECTED) 8] create /node1/node11 2222      
  Created /node1/node11
  ```

## **3. auth**

- 不使用id，即**任何经过身份验证的用户都可访问**

- **语法**：`auth:<可写可不写>:cdrwa`

- **注意**：

  * 为节点设置`auth` 认证模式时, 需实现使用`addauth` 添加**认证信息**，否则报错：`Acl is not valid`
  * 语法中**中间的id无意义，可写可不写**，该语法命令就是**给当前节点赋予权限给当前session中认证的所有用户**，后续使用`addauth` 添加认证信息不在其内。
  * **所以auth可以为某一节点赋予一组权限用户，只不过这些用户的权限一致**。

- **auth与digest区别**

  * `digest` 认证和`auth` 认证非常相似, 只不过digest 认证**无须实现通过addauth 添加认证用户信息**, 而是**直接用户名和密码设置权限**, 只不过密码不是明文, 而是密文
  * **digest 认证下一个节点只能有一个用户权限**

- **例**：

  ```shell
  [zk: localhost:2181(CONNECTED) 0] ls /
  [zookeeper]
  # 未添加认证信息时使用auth赋予权限，报错
  [zk: localhost:2181(CONNECTED) 1] create /node 123 auth::crdwa
  Acl is not valid : /node
  # 添加认证信息
  [zk: localhost:2181(CONNECTED) 2] addauth digest user1:pwd1
  [zk: localhost:2181(CONNECTED) 3] addauth digest user2:pwd2
  [zk: localhost:2181(CONNECTED) 4] addauth digest user3:pwd3
  # 创建节点并赋予权限
  [zk: localhost:2181(CONNECTED) 5] create /node 123 auth::crdwa
  Created /node
  # 查看权限信息
  [zk: localhost:2181(CONNECTED) 6] getAcl /node
  'digest,'user1:a9l5yfb9zl8WCXjVmi5/XOC0Ep4=
  : cdrwa
  'digest,'user2:LJcj8Pt1rGm2pXKbdJDGH8+Bn+0=
  : cdrwa
  'digest,'user3:vTWpf7+XOMH/ifDkxE6KmhSUCpA=
  : cdrwa
  
  #退出客户端重新登录
  # 尝试获取节点信息，访问被拒绝
  [zk: localhost:2181(CONNECTED) 1] get /node
  Authentication is not valid : /node
  # 添加任一认证用户信息
  [zk: localhost:2181(CONNECTED) 2] addauth digest user2:pwd2
  #访问成功
  [zk: localhost:2181(CONNECTED) 3] get /node                
  123
  cZxid = 0x400000038
  ctime = Sat Feb 29 21:44:02 CST 2020
  mZxid = 0x400000038
  mtime = Sat Feb 29 21:44:02 CST 2020
  pZxid = 0x400000038
  cversion = 0
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x0
  dataLength = 3
  numChildren = 0
  ```

## **4. ip**

* 基于**客户端IP地址校验**，限制只允许指定的客户端能操作znode
* 语法：`ip:<ip地址>:cdrwa`
* **例**：略

# **三、id（用户）**

**id即用户，不同的scheme，id也不同。**

* **scheme为world时**，id的值为anyone
* **scheme为digest时**，id的值为：username:BASE64(SHA1(password))
* **scheme为auth时**，id可有可无（即不使用id）
* **scheme为ip时**，id的值为客户端的ip地址

# **四、permission（权限）**

第二章介绍的`cdrwa`即为`permission`，`ZooKeeper`的ACL支持的权限如下所示：

* `CREATE(r)`：**创建子节点的权限**
* `DELETE(d)`：**删除节点的权限**
* `READ(r)`：**读取节点数据的权限**
* `WRITE(w)`：**修改节点数据的权限**
* `ADMIN(a)`：**设置子节点权限的权限**