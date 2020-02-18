# **一、什么是Broker**

* 相当于一个`ActiveMQ`的**服务器实例**
* 实质就是用代码的形式把**MQ嵌入到Java代码**，以便可以随时随地启动。
* 可以**节省资源**，同时保证**可靠性**。

# **二、Broker代码示例**

* **1）引入必要依赖**

  ```xml
  <dependency>
      <groupId>org.apache.activemq</groupId>
      <artifactId>activemq-all</artifactId>
      <version>5.15.9</version>
  </dependency>
  
  <dependency>
      <groupId>org.apache.xbean</groupId>
      <artifactId>xbean-spring</artifactId>
      <version>3.16</version>
  </dependency>
  
  <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.9.5</version>
  </dependency>
  ```

* **2）Broker代码**

  ```java
  public class EmbedBroker {
      public static void main(String[] args) throws Exception {
          BrokerService brokerService=new BrokerService();
          brokerService.setUseJmx(true);
          brokerService.addConnector("tcp://localhost:61616");
          brokerService.start();
      }
  }
  ```

* **3）生产者代码**

  ```java
  public class JMSProduce {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //队列名
      public static final String QUEUE_NAME = "queue-01";
  
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
              TextMessage textMessage = session.createTextMessage("msg---" + i);
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

* **4）消费者代码**

  ```java
  public class JMSConsumer {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://localhost:61616";
      //队列名
      public static final String QUEUE_NAME = "queue-01";
  
      public static void main(String[] args) throws JMSException {
          //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
          ActiveMQConnectionFactory activeMQConnectionFactory =
                  new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
          //2. 通过前面获得的连接工厂获取连接connection
          Connection connection = activeMQConnectionFactory.createConnection();
          connection.start();
          //3. 创建会话session（第一个参数：事务；第二个参数：签收）
          Session session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE);
          //4. 创建目的地（队列Queue或主题Topic）,此案例是队列
          Queue queue = session.createQueue(QUEUE_NAME);
          // 5. 创建消息的消费者
          MessageConsumer messageConsumer = session.createConsumer(queue);
          //6. 消费消息
          Message message = messageConsumer.receive(4000l);
          while (message != null) {
              TextMessage textMessage = (TextMessage) message;
              System.out.println(">>>消费者接受到消息：" + textMessage.getText());
              textMessage.acknowledge();
              message = messageConsumer.receive(2000l);
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

* **5）启动Broker**

  ```shell
  INFO | Using Persistence Adapter: KahaDBPersistenceAdapter[D:\GitRepositories\SpringBoot-Learning\activemq-data\localhost\KahaDB]
   INFO | JMX consoles can connect to service:jmx:rmi:///jndi/rmi://localhost:1099/jmxrmi
   INFO | Page File: activemq-data\localhost\KahaDB\db.data. Recovering pageFile free list due to prior unclean shutdown..
   INFO | KahaDB is version 6
   INFO | Page File: activemq-data\localhost\KahaDB\db.data. Recovered pageFile free list of size: 0
   INFO | PListStore:[D:\GitRepositories\SpringBoot-Learning\activemq-data\localhost\tmp_storage] started
   INFO | Apache ActiveMQ 5.15.9 (localhost, ID:LAPTOP-TUPQA4KG-53018-1582032874935-0:1) is starting
   INFO | Listening for connections at: tcp://127.0.0.1:61616
   INFO | Connector tcp://127.0.0.1:61616 started
   INFO | Apache ActiveMQ 5.15.9 (localhost, ID:LAPTOP-TUPQA4KG-53018-1582032874935-0:1) started
   INFO | For help or more information please see: http://activemq.apache.org
  ```

  根据日志`Apache ActiveMQ 5.15.9 (localhost, ID:LAPTOP-TUPQA4KG-53018-1582032874935-0:1) is starting`可以看到启动成功。

* **6）启动生产者生产消息**

  ```shell
  >>>消息发布成功！
  
  Process finished with exit code 0
  ```

  生产消息成功，说明启动的Broker没有问题

* **7）启动消费者消费消息**

  ```shell
  >>>消费者接受到消息：msg---1
  >>>消费者接受到消息：msg---2
  >>>消费者接受到消息：msg---3
  >>>消费者接受到消息：msg---4
  >>>消费者接受到消息：msg---5
  
  Process finished with exit code 0
  ```

  消费消息成功。