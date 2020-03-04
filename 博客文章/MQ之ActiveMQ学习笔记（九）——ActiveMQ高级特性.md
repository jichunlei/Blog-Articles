# **一、异步投递**

**消息投递指的是消息生产者端将消息发送到消息服务器(即Broker)的过程**

## **1. 什么是异步投递**

**ActiveMQ支持以同步、异步两种模式将消息发送到broker**，不同的模式选择对发送延时具有很大的影响。

* **同步模式**
  * 每次发送都会**阻塞producer**，直到**broker返回一个确认**，表示消息已成功接收。
  * 只要以下两种情况才会使用同步模式（**ActiveMQ默认使用异步发送**）
    * **明确指定使用同步发送的方式**
    * **未使用事务的情况下发送持久化消息**
  * 同步模式虽然提供了消息安全的保障，但是会阻塞客户端，造成很大的延时，影响性能。
* **异步模式**
  * 消息发送出去就结束，**无需等待broker的确认**，即不管broker是否接收成功。
  * **这是ActiveMQ默认使用的消息投递模式**
  * 这种模式可以最大化producer端的发送效率。一般在发送消息量较为密集的情况下使用异步发送。
  * 但是这样会带来额外的问题
    * 需要**消耗更多的Cilent端的内存**，同时**broker端性能消耗也会增加**
    * 不能有效确保消息的发送成功，客户端需要容忍**消息丢失**的可能。

## **2. 如何配置**

官网提供了**三种**配置方式

* **1）使用连接URI配置异步发送**

  ```java
  	// 在URI中增加参数传递（jms.useAsyncSend=true）
  cf = new ActiveMQConnectionFactory("tcp://locahost:61616?jms.useAsyncSend=true");
  ```

* **2）在ConnectionFactory级别配置异步发送**

  ```java
  ((ActiveMQConnectionFactory)connectionFactory).setUseAsyncSend(true);
  ```

* **3）在Connection级别配置异步发送**

  ```java
  ((ActiveMQConnection)connection).setUseAsyncSend(true);
  ```

## **3. 异步投递情况下如何确定消息发送成功与否**

### **3.1 回调机制**

ActiveMQ为我们提供了**回调机制**，在我们向消息服务器发送消息时可以同时传递一个**异步回调方法**，无论消息是否投递成功消息服务器**都会调用这个回调方法用于通知消息生产者这个消息投递的状态**。

### **3.2 代码案例**

* 代码

  ```java
  package com.jicl.queue;
  
  import org.apache.activemq.ActiveMQConnection;
  import org.apache.activemq.ActiveMQConnectionFactory;
  import org.apache.activemq.ActiveMQMessageProducer;
  import org.apache.activemq.AsyncCallback;
  import javax.jms.*;
  
  public class JMSProduce {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.129:61616";
      //队列名
      public static final String QUEUE_NAME = "async-queue-01";
  
      public static void main(String[] args) throws JMSException {
          //1. 创建连接工厂（按照给定的url，采用默认用户名和密码）
          ActiveMQConnectionFactory activeMQConnectionFactory =
                  new ActiveMQConnectionFactory(DEFAULT_BROKER_URL);
          //设置投递方式为异步投递
          activeMQConnectionFactory.setUseAsyncSend(true);
          //2. 通过前面获得的连接工厂获取连接connection
          Connection connection = activeMQConnectionFactory.createConnection();
          connection.start();
          //3. 创建会话session（第一个参数：事务；第二个参数：签收）
          Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
          //4. 创建目的地（队列Queue或主题Topic）,此案例是队列
          Queue queue = session.createQueue(QUEUE_NAME);
          // 5. 创建消息的生产者
          ActiveMQMessageProducer activeMQMessageProducer = (ActiveMQMessageProducer)session.createProducer(queue);
          //6. 生产消息
          for (int i = 1; i <= 5; i++) {
              //7. 创建消息
              TextMessage textMessage = session.createTextMessage("async-queue-msg-" + i);
              textMessage.setJMSMessageID(String.valueOf(i));
              String messageID = textMessage.getJMSMessageID();
              //8. 通过messageProducer发送给mq，同时指定异步回调方法
              activeMQMessageProducer.send(textMessage, new AsyncCallback() {
                  @Override
                  public void onSuccess() {
                      System.out.println("编号【"+messageID+"】消息发送成功！");
                  }
  
                  @Override
                  public void onException(JMSException e) {
                      System.out.println("编号【"+messageID+"】消息发送失败！失败原因："+e.getMessage());
                  }
              });
          }
          //9. 关闭资源
          //关闭生产者
          activeMQMessageProducer.close();
          //关闭会话
          session.close();
          //关闭连接
          connection.close();
          System.out.println(">>>消息发布完成！");
      }
  }
  ```

* 执行结果

  ```tex
  15:08:25.780 [main] DEBUG org.apache.activemq.transport.WireFormatNegotiator - Sending: WireFormatInfo { version=12, properties={StackTraceEnabled=true, PlatformDetails=Java, CacheEnabled=true, Host=192.168.245.129, TcpNoDelayEnabled=true, SizePrefixDisabled=false, CacheSize=1024, ProviderName=ActiveMQ, TightEncodingEnabled=true, MaxFrameSize=9223372036854775807, MaxInactivityDuration=30000, MaxInactivityDurationInitalDelay=10000, ProviderVersion=5.15.9}, magic=[A,c,t,i,v,e,M,Q]}
  15:08:25.836 [ActiveMQ Transport: tcp:///192.168.245.129:61616@59885] DEBUG org.apache.activemq.transport.InactivityMonitor - Using min of local: WireFormatInfo { version=12, properties={StackTraceEnabled=true, PlatformDetails=Java, CacheEnabled=true, Host=192.168.245.129, TcpNoDelayEnabled=true, SizePrefixDisabled=false, CacheSize=1024, ProviderName=ActiveMQ, TightEncodingEnabled=true, MaxFrameSize=9223372036854775807, MaxInactivityDuration=30000, MaxInactivityDurationInitalDelay=10000, ProviderVersion=5.15.9}, magic=[A,c,t,i,v,e,M,Q]} and remote: WireFormatInfo { version=12, properties={TcpNoDelayEnabled=true, SizePrefixDisabled=false, CacheSize=1024, ProviderName=ActiveMQ, StackTraceEnabled=true, PlatformDetails=Java, CacheEnabled=true, TightEncodingEnabled=true, MaxFrameSize=104857600, MaxInactivityDuration=30000, MaxInactivityDurationInitalDelay=10000, ProviderVersion=5.15.9}, magic=[A,c,t,i,v,e,M,Q]}
  15:08:25.836 [ActiveMQ Transport: tcp:///192.168.245.129:61616@59885] DEBUG org.apache.activemq.transport.WireFormatNegotiator - Received WireFormat: WireFormatInfo { version=12, properties={TcpNoDelayEnabled=true, SizePrefixDisabled=false, CacheSize=1024, ProviderName=ActiveMQ, StackTraceEnabled=true, PlatformDetails=Java, CacheEnabled=true, TightEncodingEnabled=true, MaxFrameSize=104857600, MaxInactivityDuration=30000, MaxInactivityDurationInitalDelay=10000, ProviderVersion=5.15.9}, magic=[A,c,t,i,v,e,M,Q]}
  15:08:25.836 [ActiveMQ Transport: tcp:///192.168.245.129:61616@59885] DEBUG org.apache.activemq.transport.WireFormatNegotiator - tcp:///192.168.245.129:61616@59885 before negotiation: OpenWireFormat{version=12, cacheEnabled=false, stackTraceEnabled=false, tightEncodingEnabled=false, sizePrefixDisabled=false, maxFrameSize=9223372036854775807}
  15:08:25.836 [ActiveMQ Transport: tcp:///192.168.245.129:61616@59885] DEBUG org.apache.activemq.transport.WireFormatNegotiator - tcp:///192.168.245.129:61616@59885 after negotiation: OpenWireFormat{version=12, cacheEnabled=true, stackTraceEnabled=true, tightEncodingEnabled=true, sizePrefixDisabled=false, maxFrameSize=104857600}
  编号【ID:1】消息发送成功！
  编号【ID:2】消息发送成功！
  编号【ID:3】消息发送成功！
  编号【ID:4】消息发送成功！
  编号【ID:5】消息发送成功！
  15:08:26.084 [main] DEBUG org.apache.activemq.util.ThreadPoolUtils - Shutdown of ExecutorService: java.util.concurrent.ThreadPoolExecutor@9629756[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0] is shutdown: true and terminated: true took: 0.000 seconds.
  15:08:26.086 [main] DEBUG org.apache.activemq.util.ThreadPoolUtils - Shutdown of ExecutorService: java.util.concurrent.ThreadPoolExecutor@58651fd0[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0] is shutdown: true and terminated: true took: 0.000 seconds.
  15:08:26.087 [main] DEBUG org.apache.activemq.transport.tcp.TcpTransport - Stopping transport tcp:///192.168.245.129:61616@59885
  15:08:26.092 [main] DEBUG org.apache.activemq.thread.TaskRunnerFactory - Initialized TaskRunnerFactory[ActiveMQ Task] using ExecutorService: java.util.concurrent.ThreadPoolExecutor@20e2cbe0[Running, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0]
  15:08:26.094 [ActiveMQ Task-1] DEBUG org.apache.activemq.transport.tcp.TcpTransport - Closed socket Socket[addr=/192.168.245.129,port=61616,localport=59885]
  15:08:26.095 [main] DEBUG org.apache.activemq.util.ThreadPoolUtils - Forcing shutdown of ExecutorService: java.util.concurrent.ThreadPoolExecutor@20e2cbe0[Running, pool size = 1, active threads = 0, queued tasks = 0, completed tasks = 1]
  >>>消息发布完成！
  ```

## **4. 总结**

* **同步发送和异步发送的区别**

  * 同步发送需要broker的确认回复，所以只要**send方法不阻塞**就表示一定发送成功。

  * 异步发送不需要broker确认，可以由客户端的**回调方法**判断是否发送成功。

* **同步发送**适用于**对消息安全有很高要求的系统，不允许消息丢失**。
* **异步发送**适用于能够**容忍少量消息丢失**，**慢消费者、快生产者**的情况下。

# **二、延迟投递和定时投递**

**ActiveMQ从5.4版本开始提供了延迟和定时投递的机制。**

## **1. 四个属性**

| **属性名**             | **属性类型** | **描述**                                    |
| ---------------------- | ------------ | ------------------------------------------- |
| `AMQ_SCHEDULED_DELAY`  | `long`       | 延迟投递时间，单位毫秒                      |
| `AMQ_SCHEDULED_PERIOD` | `long`       | 重复投递的时间间隔，单位毫秒                |
| `AMQ_SCHEDULED_REPEAT` | `int`        | 重复投递次数（总共投递次数=重复投递次数+1） |
| `AMQ_SCHEDULED_CRON`   | `String`     | `Cron`表达式（Unix系统中任务调度器）        |



## **2. 如何配置**

在`conf/activemq.xml`修改如下配置

```xml
<!--添加属性：schedulerSupport="true"-->
<broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}" schedulerSupport="true">
```

## **3. 代码案例**

### **3.1 使用属性设置**

* **1）生产者代码**

  ```java
  package com.jicl.queue;
  
  import org.apache.activemq.*;
  import javax.jms.*;
  
  public class JMSProduce {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.130:61616";
      //队列名
      public static final String QUEUE_NAME = "delay-schedule-queue-01";
  
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
          long delay = 3 * 1000 ; //延迟投递的时间
          long period = 4 * 1000 ; //重复投递的时间间隔
          int repeat = 5 ; //重复投递次数
          //7. 创建消息
          TextMessage textMessage = session.createTextMessage("async-queue-msg");
          //设置消息每过3秒投递，每4秒重复投递一次 ，一共重复投递5+1=6次
          textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY,delay);
          textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD,period);
          textMessage.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT,repeat);
          //8. 通过messageProducer发送给mq，同时指定异步回调方法
          messageProducer.send(textMessage);
          //9. 关闭资源
          //关闭生产者
          messageProducer.close();
          //关闭会话
          session.close();
          //关闭连接
          connection.close();
          System.out.println(">>>消息发布完成！");
      }
  }
  ```

* **2）消费者代码**

  ```java
  package com.jicl.queue;
  
  import org.apache.activemq.ActiveMQConnectionFactory;
  import javax.jms.*;
  import java.io.IOException;
  
  public class JMSConsumer {
  
      //activeMQ的url
      public static final String DEFAULT_BROKER_URL = "tcp://192.168.245.130:61616";
      //队列名
      public static final String QUEUE_NAME = "delay-schedule-queue-01";
  
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

* **3）分别启动消费者和生产者（注意先启动消费者），查看消费者执行结果**

  ```tex
  >>>消费者接受到消息：async-queue-msg
  >>>消费者接受到消息：async-queue-msg
  >>>消费者接受到消息：async-queue-msg
  >>>消费者接受到消息：async-queue-msg
  >>>消费者接受到消息：async-queue-msg
  >>>消费者接受到消息：async-queue-msg
  ```

  **共消费6条消息**

### **3.2 使用Cron表达式**

如下设置，具体案例不再演示。

```java
textMessage.setStringProperty(ScheduledMessage.AMQ_SCHEDULED_CRON,"0 * * * *");
```

**Cron表达式的优先级高于另外三个参数**，如果在设置了Cron的同时，也有repeat和period参数，则会在每次Cron执行的时候，重复投递repeat次，每次间隔为period，即**叠加的效果**。例如**每小时都会发生消息被投递10次，延迟1秒开始，每次间隔1秒**

```java
textMessage.setStringProperty(ScheduledMessage.AMQ_SCHEDULED_CRON,"0 * * * *");
textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, 1000);
textMessage.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD, 1000);
textMessage.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT, 9);
```

如果不熟悉`Cron`表达式可以参考如下链接学习：

[Cron表达式维基百科说明](https://zh.wikipedia.org/wiki/Cron)

# **三、消息重试机制和死信队列**

## **1. 消息重试机制**

在消息的消费过程中，如果消息未被签收或者签收失败，是会导致消息重复消费的。这里的消息重发，指的是**消息可以被broker重新分派给消费者（不一定是之前的消费者）**。

* **消息重发的几种情况**
  * 事务会话中，当还未进行`session.commit()`时，进行`session.rollback()`，那么所有还没`commit`的消息都会进行重发。
  * 使用客户端手动确认的方式时，还未进行确认并且执行`Session.recover()`，那么所有还没`acknowledge`的消息都会进行重发。
  * 所有未`ack`的消息，当进行`session.closed()`关闭事务，那么所有还没`ack`的消息`broker`端都会进行重发，而且是马上重发。
  * 消息被消费者拉取之后，超时没有响应`ack`，消息会被`broker`重发。
* 消息并不会被broker无限次重发，而是有**一定的时间间隔和重发次数**，**默认的时间间隔为1s，默认的重发次数为6次**，即当消息签收失败时`ActiveMQ`消息服务器会继续每隔1s向消费者端发送一次这个签收失败的消息，默认会尝试6次（加上正常的1次共7次）。
* 当一个消息被重发**超过默认的最大重发次数**（默认6次）时，会给MQ发一个“poison ack”表示这个消息有毒，告诉broker不要再发了。这个时候broker会把这个消息放到**DLQ（私信队列）**。
* **消费重试机制的默认相关配置如下**：
  * `collisionAvoidanceFactor`：设置**防止冲突范围的正负百分比**，只有启用`useCollisionAvoidance`参数时才生效，也就是在延迟时间上再加一个时间波动范围，默认值为0.15
  * **maximumRedeliveries**：**最大重传次数**，达到最大重传次数后抛出异常。值为-1时不限制次数，为0时不重传，**默认值为6**
  * `maximumRedeliveryDelay`：**最大重连时间间隔**，只在useExponentialBackOff为true是有效。假设首次重连间隔为10ms，倍数为2，那么第2次重连的时间间隔为20ms，第3次重连的时间间隔为40ms，当重连时间间隔大于最大重连时间间隔时，以后每次重连的时间间隔都是设置的最大重连时间间隔。默认值为-1，表示没有最大重连时间间隔
  * **initialRedeliveryDelay**：初始的**重发时间间隔**，即正常发送签收失败后间隔多长时间进行重发，**默认值为1000L，单位毫秒**
  * `redeliveryDelay`：**重发延迟时间**，当`initialRedeliveryDelay`=0是有效，默认1000L
  * `useCollisionAvoidance`：**启用防止冲突功能**，默认false
  * `useExponentialBackOff`：**启用指数倍数递增的方式增加重发延迟时间**，默认false
  * `backOffMultiplier`：**重连时间间隔的递增倍数**，只有值大于1和启用`useExponentialBackOff`参数时生效，默认为5

## **2. 死信队列**

在消息重试机制说到，如果当一个消息被重发**超过默认的最大重发次数**时，broker会不再进行重发，而是把这个消息放到**DLQ（私信队列）**。开发人员可以在这个队列中查看处理出错的消息，进行人工干预。

- 死信队列的配置主要有两种：

  - **SharedDeadLetterStrategy**

    **共享的死信队列配置策略**，将所有的`DeadLetter`保存在一个共享的队列中，这是`ActiveMQ Broker`端的默认策略。共享队列的名称默认为“`ActiveMQ.DLQ`”，可以通过“`deadLetterQueue`”属性来设定：在`activemq.xml`中的`<policyEntries>`节点中配置

    ```xml
    <deadLetterStrategy>
    	<sharedDeadLetterStrategy deadLetterQueue="DLQ-QUEUE"/>
    </deadLetterStrategy>
    ```

  - **IndividualDeadLetterStrategy**

    单独的死信队列配置策略，把`DeadLetter`放入各自的死信通道中。对于`Queue`而言，死信通道的前缀默认为“`ActiveMQ.DLQ.Queue`”；对于`Topic`而言，死信通道的前缀默认为“`ActiveMQ.DLQ.Topic`”。比如队列`Order`，那么它对应的死信通道为“`ActiveMQ.DLQ.Queue.Order`”。我们可以使用`queuePrefix`和`topicPrefix`来指定上述前缀：

    ```xml
    <!-- 仅对与order队列起作用 -->
    <policyEntry queue="order">
    	<deadLetterStrategy>
    		<!-- useQueueForQueueMessage属性的作用：是否将名为order的Topic中的DeadLetter也保存在该队列中，默认为true -->
    		<individualDeadLetterStrategy queuePrefix="DLQ." useQueueForQueueMessage="false"/>
    	</deadLetterStrategy>
    </policyEntry>
    ```

- 配置案例

  - **案例一：自动删除过期消息**

    **对于过期的消息将不会被放入到死信队列，而是自动删除，>表示对所有队列起作用，processExpired表示是否将过期消息放入死信队列，默认为true**

    ```xml
    <!-- >表示对所有队列起作用，如果只想指定某个队列直接写队列名即可 -->
    <policyEntry queue=">">
    	<deadLetterStrategy>
    		<sharedDeadLetterStrategy processExpired="false"/>
    	</deadLetterStrategy>
    </policyEntry>
    ```

  - **案例二：将签收失败的非持久消息也放入到死信队列**

    **默认情况下，ActiveMQ不会把非持久化的死消息放入死信队列，processNonPersistent表示是否将非持久化消息放入死信队列，默认为false**

    ```xml
    <!-- >表示对所有队列起作用，如果只想指定某个队列直接写队列名即可 -->
    <policyEntry queue=">">
    	<deadLetterStrategy>
    		<sharedDeadLetterStrategy processNonPersistent="true"/>
    	</deadLetterStrategy>
    </policyEntry>
    ```

## **3. 如何保证消息不被重复消费**

ActiveMQ中的消息有时是会被重复消费的，这是可能会导致一些问题的，因此我们需要保证消息不被重复消费，一般根据不同的业务有以下措施：

* 如果消息是做数据库的插入操作，给这个消息一个**唯一的主键**，那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据 。
* 在分布式系统中，可以用`redis` 等的第三方服务，给消息一个**全局 id** ，只要消费过的消息，将 id 和message 以 K-V 形式写入 `redis` ，那消费者开始消费前，先去 `redis` 中查询有没消费的记录即可。