# **一、前言**

* 在之前的博客介绍[JMS概述](http://xianzilei.cn/blog/17)的时候我们提到过**JMS的可靠性**，包括**持久化**、**事务**和**签收**三个方面，但是这些都是**局限在一台机器**上，如果本机宕机，则消息也就丢失了。
* **ActiveMQ的消息持久化**类似于Redis的持久化，我们可以配置MQ将消息**持久化到硬盘**中，即物理备份，方便宕机后的**数据恢复**。

# **二、ActiveMQ的消息持久化机制**

* **ActiveMQ的消息持久化机制主要包含下面四种**：
  * **AMQ**：日志文件（已基本不用）
  * **JDBC**：持久化到数据库
  * **KahaDB**：`AMQ`基础上改进，**默认选择**
  * **LevelDB**：`KahaDB`的改进版，基于`LevelDB`的索引

* **存储逻辑**：

  在发送者将消息发送出去后，消息中心首先将消息**存储到本地数据文件、内存数据库或远程数据库等**再试图将消息发送给接收者，**成功则将消息从存储中删除**，**失败则继续尝试发送**。消息中心启动后首先要检查指定的存储位置，若有未发送成功的消息，则需要把消息发送出去。

# **三、持久化机制之AMQ（了解）**

* 是一种**文件存储**形式，它具有写入速度快和容易恢复的特定，消息存储在一个个文件中，文件的默认大小为32M，当一个存储文件中的消息已经全部被消费，那么这个文件将被标识为可删除，在下一个清除阶段这个文件被删除，
* AMQ适用于`ActiveMQ5.3`之前的版本。目前基本上已经**不再使用**了。

# **四、持久化机制之KahaDB（重点）**

## **1. 介绍**

* **基于日志文件的持久性数据库**，是自`ActiveMQ 5.4`以来的**默认存储机制**，
* 可用于**任何场景**，提高了性能和恢能力，它是**基于文件的本地数据库存储形式**。
* 它已针对**快速持久性**进行了优化。KahaDB使用较少的文件描述符，并提供**比其前身AMQ消息存储更快的恢复**。

## **2. 配置**

* **activemq.xml配置文件中（以版本5.15.9为例）**

  ```xml
  <!--
  	Configure message persistence for the broker. The default persistence
  	mechanism is the KahaDB store (identified by the kahaDB tag).
  	For more information, see:
  
  	http://activemq.apache.org/persistence.html
  -->
  <persistenceAdapter>
      <kahaDB directory="${activemq.data}/kahadb"/>
  </persistenceAdapter>
  ```

  **可见默认持久化机制为kahaDB模式，其中持久化后的文件路径为安装目录下data文件夹中的kahadb目录下**

* **持久化文件（activemq安装目录/data/kahadb/）**

  ```shell
  [root@localhost kahadb]# cd /usr/local/java/apache-activemq-5.15.9/data/kahadb/
  [root@localhost kahadb]# ls -l
  总用量 188
  -rw-r--r--. 1 root root 33554432 2月  20 23:56 db-1.log
  -rw-r--r--. 1 root root    69632 2月  21 02:33 db.data
  -rw-r--r--. 1 root root    49240 2月  21 02:33 db.redo
  -rw-r--r--. 1 root root        8 2月  20 23:56 lock
  ```

  * **db-<number>.log**：是KahaDB存储消息到预定义大小的**数据记录文件**，主要存储数据。文件命名为`db-<number>.log`，当数据文件已满时，一个新的文件会随之创建，`number`数值也会随之递增，它随之消息数量的增多，如每32M一个文件，文件名按照数字进行编号，如`db-1.log`……当不再有引用到数据文件中的任何消息时，文件会被删除或归档。
  * **db.data**：记录了持久化的BTree索引，索引了消息数据记录中的消息，它是消息的**索引文件**，本质上是`B-Tree`（B树），使用`B-Tree`作为索引指向`db-<number>.log`里面存储的消息。
  * **db.free**：表示当前`db.data`文件哪些页面是空闲的，文件具体内容是所有**空闲页的ID**（上述未显示）。
  * **db.redo**：用来进行**消息恢复**，如果KahaDB消息存储在强制退出后启动，用于**恢复BTree索引**。
  * **lock**：**文件锁**，表示当前**获得KahaDB读写权限**的broker。

* **主要的可配置属性**

  | **属性**              | **默认**      | **说明**                                                     |
  | --------------------- | ------------- | ------------------------------------------------------------ |
  | directory             | activemq-data | 用于存储消息存储数据和日志文件的目录的路径。                 |
  | indexWriteBatchSize   | 1000          | 批量写入的索引数。                                           |
  | indexCacheSize        | 10000         | 内存中缓存的索引页数。                                       |
  | enableIndexWriteAsync | false         | 如果true，索引是异步更新的                                   |
  | journalMaxFileLength  | 32mb          | 设置消息数据日志的最大大小。                                 |
  | cleanupInterval       | 30000         | 连续检查之间的间隔（以毫秒为单位），用于确定哪些日志文件（如果有）可以从邮件存储中删除。符合条件的日志文件是没有未完成引用的文件。 |
  | checkpointInterval    | 5000          | 检查日志之前的时间（ms）                                     |
  | directoryArchive      | null          | 存储被归档的消息文件目录                                     |
  | indexDirectory        |               | 从**ActiveMQ 5.10.0开始**：如果设置，则配置将存储KahaDB索引文件（`db.data`和 `db.redo`）的位置。如果未设置，索引文件将存储在`directory`属性指定的目录中  。 |

* 更多配置属性查看下面的官方链接（这里就不再赘述）

  [kahadb配置](http://activemq.apache.org/kahadb)

## **3. 实现原理**

* **KahaDB的原理图**：

![](http://img.xianzilei.cn/KahaDB%E7%9A%84%E5%8E%9F%E7%90%86%E5%9B%BE.png)

* 在内存(cache)中的那部分`B-Tree`是`Metadata Cache`。通过将索引缓存到内存中，可以加快查询的速度，但是需要定时将 `Metadata Cache` 与 `Metadata Store`同步。这个同步过程就称为：`check point`。由`checkpointInterval`选项 决定每隔多久时间进行一次`check point`操作。
* `BTree Indexes`则是保存在磁盘上的，称为`Metadata Store`，它对应于文件`db.data`，它就是对`Data Logs`以**B树的形式索引**。`Broker`就是根据`Metadata Store`进行快速地重启恢复。若是`Metadata Store`损坏了，则只能扫描整个`Data Logs`来重建`Metadata Store`了。
* `Data Logs`则对应于文件 `db-*.log`，是消息存储的**真正载体**。

# **五、持久化机制之LevelDB（了解）**

## **1. 介绍**

* 该文件系统是从`ActiveMQ5.8`之后引进的，它和`KahaDB`很相似，也是基于文件的本地数据库存储形式，但是它提供**比KahaDB更快的持久性**，但它不再使用自定义B-Tree实现来索引预写日志，而是使用**基于LevelDB的索引**。
* 其索引具有几个不错的属性：
  * **快速更新**（无需进行随机磁盘更新）
  * **并发读取**
  * 使用**硬链接快速索引快照**

## **2. 配置**

修改activemq.xml配置文件如下

```xml
<broker brokerName="broker" ... >
    ...
    <persistenceAdapter>
      <levelDB directory="activemq-data"/>
    </persistenceAdapter>
    ...
</broker>
```

# **六、持久化机制之JDBC（重点）**

## **1. 介绍**

**顾名思义，就是将消息存储到数据库中来实现持久化。**

## **2. 配置**

* **1）添加数据库驱动jar包到ActiveMQ安装目录下的lib文件夹中**

  ```shell
  [root@localhost lib]# ll
  总用量 5740
  -rw-r--r--. 1 root root 1189300 3月  15 2019 activemq-broker-5.15.9.jar
  -rw-r--r--. 1 root root 1431933 3月  15 2019 activemq-client-5.15.9.jar
  -rw-r--r--. 1 root root  195672 3月  15 2019 activemq-console-5.15.9.jar
  -rw-r--r--. 1 root root   37291 3月  15 2019 activemq-jaas-5.15.9.jar
  -rw-r--r--. 1 root root  678142 3月  15 2019 activemq-kahadb-store-5.15.9.jar
  -rw-r--r--. 1 root root  685737 3月  15 2019 activemq-openwire-legacy-5.15.9.jar
  -rw-r--r--. 1 root root  147874 11月  6 2018 activemq-protobuf-1.1.jar
  -rw-r--r--. 1 root root     961 3月  15 2019 activemq-rar.txt
  -rw-r--r--. 1 root root  170053 3月  15 2019 activemq-spring-5.15.9.jar
  -rw-r--r--. 1 root root  115945 3月  15 2019 activemq-web-5.15.9.jar
  drwxr-xr-x. 2 root root     127 2月  16 18:30 camel
  drwxr-xr-x. 2 root root      34 2月  16 18:30 extra
  -rw-r--r--. 1 root root   20220 11月  6 2018 geronimo-j2ee-management_1.1_spec-1.0.1.jar
  -rw-r--r--. 1 root root   32359 11月  6 2018 geronimo-jms_1.1_spec-1.1.1.jar
  -rw-r--r--. 1 root root   14637 11月  6 2018 geronimo-jta_1.0.1B_spec-1.0.1.jar
  -rw-r--r--. 1 root root   50155 11月  6 2018 hawtbuf-1.11.jar
  -rw-r--r--. 1 root root   16515 11月  6 2018 jcl-over-slf4j-1.7.25.jar
  -rw-r--r--. 1 root root  999632 10月  5 10:42 mysql-connector-java-5.1.44.jar
  drwxr-xr-x. 2 root root    4096 2月  16 18:30 optional
  -rw-r--r--. 1 root root   41203 11月  6 2018 slf4j-api-1.7.25.jar
  drwxr-xr-x. 2 root root    4096 2月  16 18:30 web
  ```

* **2）activemq.xml配置**

  * **持久化机制配置**

    ```xml
    <persistenceAdapter>
        <!--<kahaDB directory="${activemq.data}/kahadb"/>-->
        <jdbcPersistenceAdapter dataSource="#mysql-ds" createTablesOnStartup="true"/>
    </persistenceAdapter>
    ```

    * `dataSource`指定将要引用的**持久化数据库的bean名称**
    * `createTablesOnStartup`表示是否在启动时创建数据表，默认值为true，这样每次启动都会创建数据表，**一般是第一次启动的时候设置为true之后改为false**。

  * **数据库连接池配置（使用官方自带的连接池）**，将其配置`</broker>`标签之后，`<import>`标签之前

    ```xml
    <bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close"> 
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/> 
        <property name="url" value="jdbc:mysql://localhost:3306/activemq?relaxAutoCommit=true"/> 
        <property name="username" value="root"/> 
        <property name="password" value="root"/> 
        <property name="poolPreparedStatements" value="true"/> 
    </bean> 
    ```

* **数据库配置**：创建一个名为`activemq`的数据库
* **重启ActiveMQ服务器**
* **查看数据库**，发现`activemq`数据库下生成了三张表
  * **ACTIVEMQ_MEGS**
    * 用于**存储消息**，Topic和Queue消息都会保存在这张表中
    * **各字段含义**
      * ID：自增的数据库主键
      * CONTAINER：消息的Destination
      * MSGID_PROD：消息发送者客户端的主键
      * MSG_SEQ：是发送消息的顺序，`MSGID_PROD+MSG_SEQ`可以组成JMS的`MessageID`
      * EXPIRATION：消息的过期时间，存储的是从1970-01-01到现在的毫秒数
      * MSG：消息本体的Java序列化对象的二进制数据
      * PRIORITY：优先级，从0-9，数值越大优先级越高
  * **ACTIVEMQ_ACKS**
    * 用于**存储订阅关系**。如果是持久化Topic，订阅者和服务器的订阅关系在这个表保存
    * **各字段含义**
      * CONTAINER：消息的Destination
      * SUB_DEST：如果是使用Static集群，这个字段会有集群其他系统的信息
      * CLIENT_ID：每个订阅者都必须有一个唯一的客户端ID用以区分
      * SUB_NAME：订阅者名称
      * SELECTOR：选择器，可以选择只消费满足条件的消息。条件可以用自定义属性实现，可支持多属性AND和OR操作
      * LAST_ACKED_ID：记录消费过的消息的ID。
  * **ACTIVEMQ_LOCK**
    * 在集群环境中才有用，只有一个Broker可以获得消息，称为Master Broker

## 3. 代码验证

**注意：必须开启持久化！**

### **3.1. 队列模式**

* **1） 生产者代码**

  ```java
  public class JMSProduce {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //队列名
      public static final String QUEUE_NAME = "jdbc-queue-01";
  
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
              TextMessage textMessage = session.createTextMessage("jdbc-queue-msg-" + i);
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

  ```java
  public class JMSConsumer {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //队列名
      public static final String QUEUE_NAME = "jdbc-queue-01";
  
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

  * **ActiveMQ控制台**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-02.png)

  * **数据库表**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-01.png)

    **生产的消息被保存在数据库表中**

* **4）启动消费者**

  * **ActiveMQ控制台**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-03.png)

  * **数据库表**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-04.png)

    **可见任意一个消息被消费者消费过，相应的消息将会立即被删除**

### **3.2. 主题模式**

* **1）订阅者代码**

  ```java
  public class JMSConsumer {
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //主题名
      public static final String TOPIC_NAME = "jdbc-topic-01";
  
      public static void main(String[] args) throws JMSException, IOException {
          System.out.println("我是1号订阅者");
          //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
          ActiveMQConnectionFactory activeMQConnectionFactory =
                  new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
          //2. 通过前面获得的连接工厂获取连接connection
          Connection connection = activeMQConnectionFactory.createConnection();
          //4. 设置客户端的id
          connection.setClientID("subscriber-001");
          //5. 创建会话session（第一个参数：事务；第二个参数：签收）
          Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
          //6. 创建目的地（队列Queue或主题Topic）,此案例是主题
          Topic topic = session.createTopic(TOPIC_NAME);
          //7. 创建订阅者
          TopicSubscriber durableSubscriber = session.createDurableSubscriber(topic, "1号订阅者");
          //8. 开始连接
          connection.start();
          //9. 消费消息
          Message message = durableSubscriber.receive();
          while (message != null) {
              TextMessage textMessage=(TextMessage) message;
              System.out.println("消费主题消息："+textMessage.getText());
              message = durableSubscriber.receive();
          }
          //10. 关闭资源
          //关闭会话
          session.close();
          //关闭连接
          connection.close();
      }
  ```

* **2）发布者代码**

  ```java
  public class JMSProduce {
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //主题名
      public static final String TOPIC_NAME = "jdbc-topic-01";
  
      public static void main(String[] args) throws JMSException {
          //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
          ActiveMQConnectionFactory activeMQConnectionFactory =
                  new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
          //2. 通过前面获得的连接工厂获取连接connection
          Connection connection = activeMQConnectionFactory.createConnection();
          //3. 创建会话session（第一个参数：事务；第二个参数：签收）
          Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
          //4. 创建目的地（队列Queue或主题Topic）,此案例是主题
          Topic topic = session.createTopic(TOPIC_NAME);
          // 5. 创建消息的生产者
          MessageProducer messageProducer = session.createProducer(topic);
          //6. 设置持久化
          messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);
          //7. 需要先设置完持久化再start，这样实现持久化
          connection.start();
          //8. 生产消息
          for (int i = 1; i <= 5; i++) {
              //创建消息
              TextMessage textMessage = session.createTextMessage("持久化msg---" + i);
              //通过messageProducer发送给mq
              messageProducer.send(textMessage);
          }
          //9. 关闭资源
          //关闭生产者
          messageProducer.close();
          //关闭会话
          session.close();
          //关闭连接
          connection.close();
          System.out.println(">>>持久化主题发布成功！");
      }
  }
  ```

* **3）启动订阅者**

  * **ActiveMQ控制台**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-05.png)

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-06.png)

    **可以看到有一个在线的订阅者**

  * **数据库表**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-07.png)

    **存储了订阅者与主题的信息**

* **4）启动发布者**

  * **ActiveMQ控制台**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-08.png)

    ​    **消息被消费**

  * **数据库表**

    ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E4%B9%8BJDBC%E7%94%A8%E4%BE%8B%E5%9B%BE-09.png)

    **可见消息消息被消费后并不会被删除，而是一直保存**

## **4. 总结**

* **必须开启持久化**，才能实现JDBC持久化机制
* **队列模式**：在没有消费者消费的情况下会将消息保存到`ACTIVEMQ_MEGS`表中，只要有任意一个消费者消费了，就会删除消费过的消息
* **主题模式**：持久订阅者永久保存到`ACTIVEMQ_ACKS`表中，而消息则永久保存在`ACTIVEMQ_MEGS`表中，在`ACTIVEMQ_ACKS`表中的订阅者有一个`last_ack_id`对应了`ACTIVEMQ_MEGS`中的`id`字段，这样就知道订阅者最后收到的消息是哪一条。

## **5. JDBC持久化机制增强（ JDBC message store with activemq journal）**

* **高性能的持久消息传递**，官方强烈推荐使用

* 指的是消息（以及事务提交/回滚和消息确认）将以尽可能快的速度**写入日志**，然后**每隔一段时间便将日志内容持久化**到对应的存储位置，从而实现**高性能**。

* 当消费者的速度能够及时跟上生产者消息的生产速度时，journal文件能够大大减少需要写入到DB中的消息。
  举个例子：生产者生产了1000条消息，这1000条消息会保存到journal文件，如果消费者的消费速度很快的情况下，在journal文件还没有同步到DB之前，消费者已经消费了90%的以上消息，那么这个时候只需要同步剩余的10%的消息到DB。如果消费者的速度很慢，这个时候journal文件可以使消息以批量方式写到DB。

* **配置**

  ```xml
  <persistenceFactory>
  	<journalPersistenceAdapterFactory 
      	journalLogFiles="4" 
          journalLogFileSize="32768"
   　　　　useJournal="true" 
          useQuickJournal="true" 
          dataSource="#mysql_ds" 
          dataDirectory="activemq-data" />
  </persistenceFactory>
  ```

  