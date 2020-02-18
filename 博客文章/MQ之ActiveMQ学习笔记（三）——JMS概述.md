# **一、JMS介绍**

## **1. JavaEE 概述及主要核心规范**

* **什么是JavaEE**

  JavaEE是一套使用Java进行企业级应用开发的大家一致遵循的**13个核心规范工业标准**。JavaEE平台提供了一个基于组件的方法来加快设计，开发。装配及部署企业应用程序。

* **JavaEE的13种核心技术**

  * `XML`：`Extensible Markup Language`

    XML是一种简单灵活的文本格式的**可扩展标记语言**，被设计用来传输和存储数据。

  * `Servlet`

    * Servlet是Java Servlet的简称，称为**小服务程序**或**服务连接器**，用Java编写的服务器端程序，主要功能在于交互式地浏览和修改数据，生成动态Web内容。
    * 狭义的Servlet是指Java语言实现的一个接口，广义的Servlet是指任何实现了这个Servlet接口的类，一般情况下，人们将Servlet理解为后者。Servlet运行于支持Java的应用服务器中。从原理上讲，Servlet可以**响应任何类型的请求**，但绝大多数情况下Servlet只用来扩展基于HTTP协议的Web服务器。

  * `JSP`：`Java Server Pages`

    中文名叫**Java服务器页面**，其本质是一个**简化的Servlet**。`JSP`将网页逻辑与网页设计的显示分离，支持可重用的基于组件的设计，使基于Web的应用程序的开发变得迅速和容易。 `JSP`是一种动态页面技术，它的主要目的是将表示逻辑从Servlet中分离出来。`Java Servlet`是`JSP`的技术基础，而且大型的Web应用程序的开发需要`Java Servlet`和`JSP`配合才能完成。

  * `JDBC`：`Java Database Connectivity`

    **Java连接数据库**，是一种用于执行`SQL`语句的`JavaAPI`，可以为多种关系数据库提供统一访问，它由**一组用`Java`语言编写的类和接口组成**。`JDBC`提供了一种基准，据此可以构建更高级的工具和接口，使数据库开发人员能够编写数据库应用程序。`JDBC`对数据库的访问也具有平台无关性。

  * `JavaMail`

    提供给开发者**处理电子邮件相关的编程接口**，它提供了一套**邮件服务器的抽象类**。不仅支持`SMTP`服务器，也支持`IMAP`服务器和`POP`服务器。

  * `JNDI`：`Java Naming and Directory Interface`

    是一种**标准的Java命名系统接口**，JNDI提供统一的客户端API，通过不同的访问提供者接口JNDI服务供应接口(SPI)的实现，由管理者将JNDI API映射为特定的命名服务和目录系统，使得Java应用程序可以和这些命名服务和目录服务之间进行交互。

  * `EJB`：`Enterprise JavaBean`

    是**JavaEE服务器端组件模型**，设计目标与核心应用是部署分布式应用程序。简单来说就是把已经编写好的程序（即：类）打包放在服务器上执行。凭借java跨平台的优势，用EJB技术部署的分布式系统可以不限于特定的平台。EJB是javaEE的一部分，定义了一个用于开发基于组件的企业多重应用程序的标准。其特点包括网络服务支持和核心开发工具(SDK)。

  * `RMI`：`Remote Method Invocation`

    指的是**远程方法调用**。它是一种机制，能够让在某个 Java虚拟机上的对象调用另一个 Java 虚拟机中的对象上的方法。可以用此方法调用的任何对象必须实现该远程接口。

  * `Java IDL（Interface Description Language）/CORBA（CommonObject Broker Architecture）`：Java 接口定义语言/公用对象请求代理程序体系结构

    * `Java IDL`即`idltojava`编译器就是一个`ORB`，可用来在`Java`语言中定义、实现和访问`CORBA`对象。`Java IDL`支持的是一个瞬间的`CORBA`对象，即在对象服务器处理过程中有效。实际上，`Java IDL`的`ORB`是一个类库而已，并不是一个完整的平台软件，但它对`Java IDL`应用系统和其他`CORBA`应用系统之间提供了很好的底层通信支持，实现了`OMG`定义的`ORB`基本功能。
    * `CORBA`（Common Object Request Broker Architecture,公共对象请求代理体系结构，通用对象请求代理体系结构）是由`OMG`组织制订的一种标准的面向对象应用程序体系规范。或者说 `CORBA`体系结构是对象管理组织（OMG）为解决分布式处理环境(DCE)中，硬件和软件系统的互连而提出的一种解决方案；`OMG`组织是一个国际性的非盈利组织，其职责是为应用开发提供一个公共框架，制订工业指南和对象管理规范，加快对象技术的发展。

  * `JMS`：`Java Message Service`

    即**Java消息服务应用程序接口**，是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持。

  * `JTA`：`Java Transaction API`

    即**Java 事务 API**，JTA允许应用程序执行分布式事务处理——在两个或多个网络计算机资源上访问并且更新数据。JDBC驱动程序的JTA支持极大地增强了数据访问能力。**JTA事务比JDBC事务更强大**。一个JTA事务可以有多个参与者，而一个JDBC事务则被限定在一个单一的数据库连接。

  * `JTS`：`Java Transaction Service`

    是一个**组件事务监视器**。`JTS`是`CORBA OTS`事务监控的基本实现。JTS规定了事务管理器的实现方式。`JTS`事务管理器为应用服务器、资源管理器、独立的应用以及通信资源管理器提供了事务服务。

  * `JAF`：`JavaBean Activation Framework`

    是一个专用的**数据处理框架**，它用于封装数据，并为应用程序提供访问和操作数据的接口。`JAF`的主要作用是让`java`应用程序知道如何对一个数据源进行查看、编辑和打印等操作。`Mail API` 的所有版本都需要 `JAF` 来支持任意数据块的输入及相应处理。

## **2. 什么是JMS规范**

* **Java消息服务**

  Java消息服务指的是两个应用程序之间进行异步通信的API，它为标准协议和消息服务提供了一组通用接口，包括创建、发送、读取消息等，用于支持Java应用程序开发。在JavaEE中，当两个应用程序使用JMS进行通信时，它们之间不是直接相连的，而是通过一个共同的消息收发服务组件关联起来以达到解耦/异步削峰的效果。

* **JMS体系架构**

  ![](http://img.xianzilei.cn/JMS%E7%BC%96%E7%A0%81%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

# **二、JMS组成结构**

## **1. JMS Provider**

**实现JMS接口和规范的消息中间件**，也就是MQ服务器。

## **2. JMS Producer**

**消息生产者**，创建以及发送消息的客户端应用。

## **3. JMS Consumer**

**消息消费者**，接收与处理消息的客户端。

## **4. JSM Message**

### **1）消息头**

**下面是5个主要的消息头**

* **JMSDestination**

  * **消息发送的目的地**
  * 主要是指`Queue`和`Topic`
  * 自动分配

* **JMSDeliveryMode**

  * **传送模式**
  * 分为**持久**模式和**非持久**模式。
    * 一条持久性的消息应该被传送“一次仅仅一次”，这就意味着如果JMS提供者出现故障，该消息并不会丢失，它会在服务器恢复之后再次传递。
    * 一条非持久的消息最多会传送一次，这意味着服务器出现故障，该消息将永久丢失。

  * 自动分配

* **JMSExpiration**

  * **消息过期时间**，等于`Destination`的`send`方法中的`timeToLive`值加上发送时刻的GMT时间值。
  * 如果`timeToLive`值等于零，则`JMSExpiration`被设为零，表示该消息**永不过期**。
  * 如果发送后，在消息过期时间之后消息还没有被发送到目的地，则该消息**被清除**。
  * **默认是永不过期**
  * 自动分配

* **JMSPriority**

  * **消息优先级**
  * 从0-9十个级别，0-4是普通消息，5-9是加急消息。**JMS不要求JMS Provider严格按照这十个优先级发送消息，但必须保证加急消息要先于普通消息到达**。
  * **默认是4级**
  * 自动分配

* **JMSMessageId**

  * **唯一识别每个消息的标识**
  * 默认由`JMS Provider`产生，也可自己定制。
  * 自动分配

### **2）消息体**

**发送和接收的消息体类型必须一致对应**

* **TextMessage**

  **普通字符串消息**，包含一个`String`。

* **Mapmessage**

  **Map类型的消息**，`key`为`String`类型，而值为Java基本类型。

* **BytesMessage**

  **二进制数组消息**，包含一个`byte[]`。

* **StreamMessage**

  **Java 数据流消息**，用标准流操作来顺序的填充读取。

* **ObjectMessage**

  **对象消息**，包含一个可序列化的`Java`对象。

### **3） 消息属性**

* 我们可以给消息设置自定义属性，这些属性主要是提供给应用程序的。对于实现消息过滤功能，消息属性非常有用。

* JMS API定义了一些标准属性，JMS服务提供者可以选择性的提供部分标准属性。

* 如图所示：`setXXXProperty`的方法，参数类型为键值对的方式

  ![](http://img.xianzilei.cn/%E6%B6%88%E6%81%AF%E5%B1%9E%E6%80%A7API.png)

# **三、JMS的可靠性**

**JMS可靠性的一个重要方面是确保持久性消息传送至目标后，消息服务在向消费者传送它们之前不会丢失这些消息**

## **1. 持久化**

* **持久化的Queue**

  * **持久化的消息**：服务器宕机后消息**依旧存在**，只是没有入队，当服务器再次启动，消息任就会被消费。

  * **非持久化的消息**：服务器宕机后消息**永远丢失**。

  * **默认是持久化**

  * 代码示例

    ```java
    MessageProducer messageProducer = session.createProducer(queue);
    
    //1.设置通过session创建出来的生产者生产的Queue消息为持久化（默认）
    messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);
    
    //2.设置通过session创建出来的生产者生产的Queue消息为非持久化
    messageProducer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);
    ```

* **持久化的Topic**

  * **非持久化的消息**：当发送方发送消息的时候：如果接收方不在线，那么消息会被废弃，即接收方永远也收不到这些消息了。如果接收方在线，则接收方会收到这些消息。
  
  * **持久化的消息**：无论消费者是否在线，都会接收到，如果消费者不在线的话，下次连接的时候， 会把没有收过的消息都接收下来。（前提是该消费者预先订阅过该主题）
  
  * **默认是非持久化的**
  
  * 持久化的Topic类似于微信公众号的订阅发布
  
  * 代码示例（持久化）
  
    * 生产者代码
  
      ```java
      public class JMSProduce {
      
          //activeMQ的url
          public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
          //主题名
          public static final String TOPIC_NAME = "topic-01";
      
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
  
    * 消费者代码
  
      ```java
      public class JMSConsumer {
          
          //activeMQ的url
          public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
          //主题名
          public static final String TOPIC_NAME = "topic-01";
      
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
      }
      ```
  
    * 1）**先启动两个订阅（消费）者（必须先启动消费者，等于订阅了这个主题）**
  
      * IDEA控制台
  
        ```shell
        #1号订阅者日志
        我是1号订阅者
        
        #2号订阅者日志
        我是2号订阅者
        ```
  
      * ActiveMQ控制台
  
        ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-001.png)
  
        ​	**可以看到2个订阅（消费）者，此时发布（生产）者还未发布消息**
  
        ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-002.png)
  
        ​	**在线订阅数为2**
  
      
  
    * 2）**手动关闭1号订阅（消费）者**
  
      ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-003.png)
  
      ​	**1号订阅者进入离线状态**
  
    * 3）**启动生产者生产5条消息**，此时由于2号订阅者保持连接在，2号订阅者会接受到消息。
  
      * IDEA控制台
  
        ```shell
        #2号订阅者日志
        消费主题消息：持久化msg---1
        消费主题消息：持久化msg---2
        消费主题消息：持久化msg---3
        消费主题消息：持久化msg---4
        消费主题消息：持久化msg---5
        ```
  
      * ActiveMQ控制台，消息消费数量为5，即2号订阅者接受的消息数
  
        ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-004.png)
  
    * 4）**重新启动1号订阅者**，1号订阅者会重新出现在在线列表中，同时接受生产者之前发布的消息
  
      ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-005.png)
  
      ​	**1号订阅者重新回到在线列表中**
  
      ![](http://img.xianzilei.cn/%E6%8C%81%E4%B9%85%E5%8C%96%E7%9A%84Topic%E6%A1%88%E4%BE%8B%E5%9B%BE-006.png)
  
      ​	**1号订阅者会接收之前未接受的消息，此时消息总消费数为10**

## **2. 事务**

* **代码设置**

  ```java
  //创建会话session（第一个参数：事务,true表示开启，false表示关闭）
  Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
  ```

* **生产者** 

  * false：生产者执行完send()方法之后,该信息就会被**提交到队列**之中
  * true：生产者做的操作不会被立即提交,需要使用session对象**调用commit()方法**,之前的操作才会被真正提交,也可以回滚。

* **消费者**

  * false：消费者消费完一条信息之后,消息会**立马从消息队列中出列**
  * true：代表消费者消费完信息之后,需要**调用comment()方法**才能消息才能真正出列。否则的话,所消费的消息就不会出列,这样就出现消息的**重复消费**。

## **3. 签收**

**签收是针对消费端的参数**

* **代码设置**

  ```java
  //创建会话session（第二个参数：签收）
  //1) Session.AUTO_ACKNOWLEDGE：自动签收
  //2) Session.CLIENT_ACKNOWLEDGE：手动签收
  //3) Session.DUPS_OK_ACKNOWLEDGE：允许消息重复
  Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
  ```

* 参数设置

  * **非事务情况下**

    * **1）Session.AUTO_ACKNOWLEDGE**：自动签收

      当客户端从`receive()`或`onMessage()`成功返回时，Session**自动签收**客户端的这条消息。

    * **2) Session.CLIENT_ACKNOWLEDGE**：手动签收

      客户端需要通过调用消息(Message)的`acknowledge()`方法签收消息，即需要**手动签收**。

    * **3) Session.DUPS_OK_ACKNOWLEDGE**：允许消息重复（不常用）

      Session不必确保对传送消息的签收，这个模式可能会引起消息的重复，但是降低了Session的开销，所以只有客户端能容忍重复的消息，才可使用。不常用

  * **事务情况下**
    * 事务性会话中，当一个事务被成功提交则消息被**自动签收**。也就是说，在事务的开启的情况下，无论是自动签收或者是手动签收，**只要事务成功提交了，消息就会被自动签收**；**如果事务回滚，则消息会被再次传送**。
    * 在**手动签收**的情况下，不调用消息(Message)的`acknowledge`方法消息也会被签收,消息也会出列。而如果你调用消息(Message)的`acknowledge`方法进行签收，但是没有`commit`的话也是不会签收的。

## **4. 签收和事务的关系**

- **在事务性会话中**：当一个事务被成功提交则消息被自动签收。如果事务回滚，则消息会被再次传送。
- **非事务性会话中**：消息何时被确认取决于创建会话时的应答模式。
- **事务级别高于签收，开启事务时，默认自动签收**。

# **四、JMS总结**

## **1. JMS的点对点总结**

**点对点模型**是基于**队列**的，生产者**发送消息**到队列，消费者从队列**接收消息**，队列的存在使得消息的**异步传输**成为可能。

* 如果在Session关闭时有部分消息被收到但还没有被签收（acknowledge），那当消费者下次连接到相同的队列时，这些消息还会被再次接收。
* 队列可以**长久的保存消息**直到消费者收到消息。消费者不需要因为担心消息会丢失而时刻和队列保持激活的链接状态，充分体现了**异步传输**模式的优势。

## **2. JMS的发布订阅总结**

**JMS Pub/Sub 模型**定义了如何向一个内容节点发布和订阅消息，这些节点被称作topic

* 主题可以被认为是消息的**传输中介**，发布者（publisher）发布消息到主题，订阅者（subscribe）从主题订阅消息。
* 主题使得消息订阅者和消息发布者保持**互相独立**不需要解除即可保证消息的传送。
* **持久订阅**：
  * 客户端首先**向MQ注册一个自己的身份ID识别号**，当这个客户端处于离线时，生产者会为这个ID保存所有发送到主题的消息，当客户再次连接到MQ的时候，会**根据消费者的ID**得到所有当自己处于离线时发送到主题的消息
  * 
    持久订阅能够**恢复或重新派送一个未签收的消息**。
* **非持久订阅**：
  * 非持久订阅只有当客户端处于激活状态，也就是和MQ**保持连接状态**才能收发到某个主题的消息。
  * 如果消费者处于离线状态，生产者发送的主题消息将会**丢失作废**，消费者永远不会收到。
  * 
    注意：**先订阅注册才能接受到发布，只给订阅者发布消息**。