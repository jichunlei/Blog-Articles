# **一、ActiveMQ简介**

## **1. 什么是ActiveMQ**

* **Apache ActiveMQ**是`Apache`软件基金会所研发的开放源代码消息中间件。
* 由于`ActiveMQ`是一个纯Java程序，因此只需要操作系统支持Java虚拟机，`ActiveMQ`便可执行

## **2. 主要特性**

* **服从JMS规范**：JMS 规范提供了良好的标准和保证，包括：同步 或 异步 的消息分发，一次和仅一次的消息分发，消息接收和订阅等等。遵从JMS规范的好处在于，不论使用什么JMS实现提供者，这些基础特性都是可用的；
* **连接灵活性**：`ActiveMQ`提供了广泛的连接协议，支持的协议有：HTTP/S，IP多播，SSL，TCP，UDP等等。对众多协议的支持让`ActiveMQ`拥有了很好的灵活性；
* **支持的协议种类多**：`OpenWire`、`STOMP`、`REST`、`XMPP`、`AMQP`；
* **持久化插件和安全插件**：`ActiveMQ`提供了多种持久化选择。而且，`ActiveMQ`的安全性也可以完全依据用户需求进行自定义鉴权和授权；
* **支持的客户端语言种类多**：除了Java之外，还有：C/C++，.NET，Perl，PHP，Python，Ruby；
* **代理集群**：多个`ActiveMQ`代理可以组成一个集群来提供服务；
* **异常简单的管理**：`ActiveMQ`是以开发者思维被设计的。所以，它并不需要专门的管理员，因为它提供了简单又使用的管理特性。有很多中方法可以监控`ActiveMQ`不同层面的数据，包括使用在`JConsole`或者在`ActiveMQ`的`WebConsole`中使用JMX。通过处理JMX的告警消息，通过使用命令行脚本，甚至可以通过监控各种类型的日志。

# **二、ActiveMQ下载及安装**

## **1. 下载**

[官网下载地址](http://activemq.apache.org/activemq-5159-release)（版本5.15.9）

## **2. 安装**

解压到指定目录下（默认为`/usr/local/java`下）

```shell
tar -zxvf apache-activemq-5.15.9-bin.tar.gz -C /usr/local/java
```

## **3. 启动**

* **进入bin目录**：`/usr/local/java/apache-activemq-5.15.9/bin`

* **执行启动命令**：`./activemq start`

  ```shell
  [root@localhost bin]# ./activemq start
  INFO: Loading '/usr/local/java/apache-activemq-5.15.9//bin/env'
  INFO: Using java '/usr/local/java/jdk1.8.0_144/bin/java'
  INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
  INFO: pidfile created : '/usr/local/java/apache-activemq-5.15.9//data/activemq.pid' (pid '1393')
  ```

* **查看启动结果**：

  ```shell
  [root@localhost bin]# ps -ef|grep activemq|grep -v grep
  root       1393      1  0 18:33 pts/0    00:00:40 /usr/local/java/jdk1.8.0_144/bin/java -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dcom.sun.management.jmxremote -Djava.awt.headless=true -Djava.io.tmpdir=/usr/local/java/apache-activemq-5.15.9//tmp -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data -jar /usr/local/java/apache-activemq-5.15.9//bin/activemq.jar start
  [root@localhost bin]# netstat -an|grep 61616
  tcp6       0      0 :::61616                :::*                    LISTEN 
  ```

  可以看到启动成功！（**activemq默认端口61616**）

## **4. 关闭**

**执行关闭命令**：`./activemq stop`

```shell
[root@localhost bin]# ./activemq stop
INFO: Loading '/usr/local/java/apache-activemq-5.15.9//bin/env'
INFO: Using java '/usr/local/java/jdk1.8.0_144/bin/java'
INFO: Waiting at least 30 seconds for regular process termination of pid '1393' : 
Java Runtime: Oracle Corporation 1.8.0_144 /usr/local/java/jdk1.8.0_144/jre
  Heap sizes: current=63360k  free=62311k  max=1013632k
    JVM args: -Xms64M -Xmx1G -Djava.util.logging.config.file=logging.properties -Djava.security.auth.login.config=/usr/local/java/apache-activemq-5.15.9//conf/login.config -Dactivemq.classpath=/usr/local/java/apache-activemq-5.15.9//conf:/usr/local/java/apache-activemq-5.15.9//../lib/: -Dactivemq.home=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.base=/usr/local/java/apache-activemq-5.15.9/ -Dactivemq.conf=/usr/local/java/apache-activemq-5.15.9//conf -Dactivemq.data=/usr/local/java/apache-activemq-5.15.9//data
Extensions classpath:
  [/usr/local/java/apache-activemq-5.15.9/lib,/usr/local/java/apache-activemq-5.15.9/lib/camel,/usr/local/java/apache-activemq-5.15.9/lib/optional,/usr/local/java/apache-activemq-5.15.9/lib/web,/usr/local/java/apache-activemq-5.15.9/lib/extra]
ACTIVEMQ_HOME: /usr/local/java/apache-activemq-5.15.9
ACTIVEMQ_BASE: /usr/local/java/apache-activemq-5.15.9
ACTIVEMQ_CONF: /usr/local/java/apache-activemq-5.15.9/conf
ACTIVEMQ_DATA: /usr/local/java/apache-activemq-5.15.9/data
Connecting to pid: 1393
.Stopping broker: localhost
. TERMINATED
```

# **三、ActiveMQ控制台**

* **ActiveMQ控制台访问地址**：

  * ```tex
    http://你的mq所在的服务器ip:8161/admin
    ```

  * 初始用户名和密码都是`admin`

  * 注意：

    * **端口`61616`：提供JMS服务**
    * **端口`8161`：提供管理控制台服务**

* **界面**如图所示：

  ![](http://img.xianzilei.cn/ActiveMQ%E6%8E%A7%E5%88%B6%E5%8F%B0.png)

# **四、Java编码实现ActiveMQ通讯**

## **1. JMS开发的基本步骤**

* 1）创建一个`connection factory`

* 2）通过`connection factory`来创建`JMS connection`

* 3）启动`JMS connection`

* 4）通过`connection`创建`JMS session`

* 5）创建`JMS destination`

* 6）创建`JMS producer`或者创建`JMS message`并设置`destination`

* 7）创建`JMS consumer`或者是注册一个`JMS message listener`

* 8）发送或者接受`JMS message(s)`

* 9）关闭所有的JMS资源(`connection`, `session`, `producer`, `consumer`等)

  ![](http://img.xianzilei.cn/JMS%E7%BC%96%E7%A0%81%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

## **2. 代码实现（队列模式）**

* **运行环境**

  * IDEA 2018
  * JDK 1.8
  * Maven 3.5

* **导入依赖**

  ```xml
  <!--activemq所需jar包-->
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
  ```

* **生产者代码实现（队列）**

  * **代码实现**

    ```java
    package com.jicl;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    
    /**
     * JMS（队列实现）--消息生产者
     *
     * @author : xianzilei
     * @date : 2020/2/16 17:08
     */
    public class JMSProduce {
    
        //activeMQ的url
        public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
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

  * **执行结果**

    * **IDEA控制台**

      ```tex
      >>>消息发布成功！
      
      Process finished with exit code 0
      ```

    * **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-001.png)

      * `Number Of Pending Messages`：等待消费的消息，即当前未出列的消息数量（总接受数 - 总出队列数）。
      * `Number Of Consumers `：消费者数量，即消费者端的消费者数量。
      * `Messages Enqueued`：进队消息数，即进入队列的总数量，包括出队列的。（这个数量只增不减）。
      * `Messages Dequeued`：出队消息数，即消费者消费掉的数量。

* **消费者代码实现（队列）**

  * **代码实现一（同步阻塞方式）**

    ```java
    package com.jicl;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    
    /**
     * JMS（队列实现）--消息消费者
     *
     * @author : xianzilei
     * @date : 2020/2/16 19:14
     */
    public class JMSConsumer {
    
        //activeMQ的url
        public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
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

    * `receive`方法说明

      ```java
      // 收到消息前一直阻塞进程
      javax.jms.MessageConsumer#receive()
      // 超时后不再阻塞进程
      javax.jms.MessageConsumer#receive(long timeout)
      ```

  * **代码实现二（异步非阻塞方式）**

    ```java
    package com.jicl;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    import java.io.IOException;
    
    /**
     * JMS（队列实现）--消息消费者
     *
     * @author : xianzilei
     * @date : 2020/2/16 19:14
     */
    public class JMSConsumer {
    
            //activeMQ的url
            public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
            //队列名
            public static final String QUEUE_NAME = "queue-01";
    
            public static void main(String[] args) throws JMSException, IOException {
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
                //6. 消费消息：消费方式二--异步非阻塞方式（监听器onMessage()）
                messageConsumer.setMessageListener(message -> {
                    if (message instanceof TextMessage) {
                        try {
                            String text = ((TextMessage) message).getText();
                            System.out.println(">>>消费者接受到消息：msg---" + text);
                        } catch (JMSException e) {
                            e.printStackTrace();
                        }
                    }
                });
                //手动阻塞进程，防止消费者还未消费就被关闭
                System.in.read();
                //9. 关闭资源
                //关闭生产者
                messageConsumer.close();
                //关闭会话
                session.close();
                //关闭连接
                connection.close();
            }
    }
    ```

  * **执行结果**

    * 1）**同步阻塞方式**

      * receive()

        * IDEA控制台

          ```shell
          >>>消费者接受到消息：msg---1
          >>>消费者接受到消息：msg---2
          >>>消费者接受到消息：msg---3
          >>>消费者接受到消息：msg---4
          >>>消费者接受到消息：msg---5
          ```

          注意：此处程序并没有结束，始终阻塞在receive()方法处

        * ActiveMQ控制台：此时消费者数量为1，因为消费端程序还未结束。

          ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-002.png)

      * receive(timeout)

        * IDEA控制台

          ```shell
          >>>消费者接受到消息：msg---1
          >>>消费者接受到消息：msg---2
          >>>消费者接受到消息：msg---3
          >>>消费者接受到消息：msg---4
          >>>消费者接受到消息：msg---5
          
          Process finished with exit code 0
          ```

        * ActiveMQ控制台：此时消费者数量为0，因为消费端程序已经结束。

          ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-003.png)

    * 2）**异步非阻塞方式**

      * IDEA控制台，因为手动阻塞（防止消费者还未消费就被关闭），所以程序未结束

        ```shell
        >>>消费者接受到消息：msg---msg---1
        >>>消费者接受到消息：msg---msg---2
        >>>消费者接受到消息：msg---msg---3
        >>>消费者接受到消息：msg---msg---4
        >>>消费者接受到消息：msg---msg---5
        ```

      * ActiveMQ控制台：同上

## **3. 代码实现（主题模式）**

**主题模式需要先启动消费者再启动生产者生产数据，否则生产者生产的数据就会被废弃。**

* **消费者代码实现（主题）**

  * **代码实现**：与队列不同处主要在**创建消息的消费者**处

    ```java
    package com.jicl;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    import java.io.IOException;
    
    /**
     * JMS（主题实现）--消息消费者
     *
     * @author : xianzilei
     * @date : 2020/2/16 19:14
     */
    public class JMSConsumer {
    
            //activeMQ的url
            public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
            //队列名
            public static final String TOPIC_NAME = "topic-01";
    
            public static void main(String[] args) throws JMSException, IOException {
                System.out.println("我是1号消费者");
                //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
                ActiveMQConnectionFactory activeMQConnectionFactory =
                        new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
                //2. 通过前面获得的连接工厂获取连接connection
                Connection connection = activeMQConnectionFactory.createConnection();
                connection.start();
                //3. 创建会话session（第一个参数：事务；第二个参数：签收）
                Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
                //4. 创建目的地（队列Queue或主题Topic）,此案例是主题
                Topic topic = session.createTopic(TOPIC_NAME);
                // 5. 创建消息的消费者
                MessageConsumer messageConsumer = session.createConsumer(topic);
                //6. 消费消息
                messageConsumer.setMessageListener(message -> {
                    if (message instanceof TextMessage) {
                        try {
                            String text = ((TextMessage) message).getText();
                            System.out.println(">>>消费者接受到消息：msg---" + text);
                        } catch (JMSException e) {
                            e.printStackTrace();
                        }
                    }
                });
                //手动阻塞进程，防止消费者还未消费就被关闭
                System.in.read();
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

  * **执行结果（这里启动三个消费者）**

    ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-004.png)

    * **IDEA控制台**

      ```shell
      #1号消费者日志
      我是1号消费者
      
      #2号消费者日志
      我是2号消费者
      
      #3号消费者日志
      我是3号消费者
      ```

    * **ActiveMQ控制台**

      

      ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-005.png)

      其中消费者数量为3

* **生产者代码实现（主题）**

  * **代码实现**：与队列不同处主要在**创建消息的生产者**处

    ```java
    package com.jicl;
    
    import org.apache.activemq.ActiveMQConnectionFactory;
    import javax.jms.*;
    
    /**
     * JMS（主题实现）--消息生产者
     *
     * @author : xianzilei
     * @date : 2020/2/16 17:08
     */
    public class JMSProduce {
    
        //activeMQ的url
        public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.128:61616";
        //队列名
        public static final String TOPIC_NAME = "topic-01";
    
        public static void main(String[] args) throws JMSException {
            //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
            ActiveMQConnectionFactory activeMQConnectionFactory =
                    new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
            //2. 通过前面获得的连接工厂获取连接connection
            Connection connection = activeMQConnectionFactory.createConnection();
            connection.start();
            //3. 创建会话session（第一个参数：事务；第二个参数：签收）
            Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            //4. 创建目的地（队列Queue或主题Topic）,此案例是主题
            Topic topic = session.createTopic(TOPIC_NAME);
            // 5. 创建消息的生产者
            MessageProducer messageProducer = session.createProducer(topic);
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

  * **执行结果**

    * **IDEA控制台**：每个消费者日志**均一样**

      ```shell
      >>>消费者接受到消息：msg---msg---1
      >>>消费者接受到消息：msg---msg---2
      >>>消费者接受到消息：msg---msg---3
      >>>消费者接受到消息：msg---msg---4
      >>>消费者接受到消息：msg---msg---5
      ```

    * **ActiveMQ控制台**

      ![](http://img.xianzilei.cn/activeMQ%E6%A1%88%E4%BE%8B%E5%9B%BE-006.png)

      其中出队消息数，即消费者消费掉的数量为`3*5=15`

## **4. 两种消息传递模式对比**

| **模式**       | **Queue（队列）模式**                                        | **Topic（主题）模式**                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **工作模式**   | **“负载均衡”模式，没有消费者，消息也不会丢失；有多个消费者，一条消息也只会发送给其中一个消费者，要求消费者ack信息** | **"订阅-发布"模式，无订阅者，消息会被丢弃；多个订阅者，则都会收到消息** |
| **状态**       | 默认会在mq服务器中以文件的形式储存，一般保存在`$AMQ_HOME\data\kr-store\data`，可配置`DB`存储 | 无状态                                                       |
| **传递完成性** | 消息不会丢失                                                 | 无订阅者，消息会被丢弃                                       |
| **处理效率**   | 一条消息只发送给一个消费者，所以消费者再多，性能不会明显降低 | 消息按照订阅者数量进行复制，处理性能随其增加会明显降低       |

