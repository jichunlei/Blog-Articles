# **一、项目准备**

* **运行环境**
  * `IDEA 2018`
  * `JDK 1.8`
  * `Maven 3.5`
  * `SpringBoot 2.0.4`
  * `ActiveMQ 5.15.9`

* **分别新建两个微服务项目**，一个**生产（发布）者服务**，一个**消费（订阅）者服务**（细节略）

* **导入依赖**

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <!--Springboot整合activemq包-->
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-activemq</artifactId>
  </dependency>
  <!--activemq连接池-->
  <dependency>
      <groupId>org.apache.activemq</groupId>
      <artifactId>activemq-pool</artifactId>
  </dependency>
  ```

# **二、SpringBoot整合ActiveMQ之点对点（队列）**

## **1. 生产端服务实现**

* **1）`application.yml`主配置文件**

  ```yaml
  server:
    port: 8888 #微服务端口号
  
  spring:
    activemq:
      broker-url: tcp://192.168.245.128:61616 #activemq服务器地址
      user: admin #用户名
      password: admin #密码
      pool:
        enabled: true #是否开启连接池
        max-connections: 100 #连接池最大连接数
    jms:
      pub-sub-domain: false #false(默认)=Queue true=Topic
  
  #自定义队列名
  myQueueName: xianzilei-queue-01
  ```

* **2）`ActiveMQ`配置类**

  ```java
  package com.jicl.config;
  
  import org.apache.activemq.command.ActiveMQQueue;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.jms.annotation.EnableJms;
  import org.springframework.stereotype.Component;
  import javax.jms.Queue;
  
  /**
   * ActiveMQ配置类
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:24
   */
  @Component
  @EnableJms //开启JMS注解功能
  public class ActiveMQConfig {
      //自定义的队列名
      @Value("${myQueueName}")
      private String myQueueName;
  
      //注入队列实例
      @Bean
      public Queue queue(){
          return new ActiveMQQueue(myQueueName);
      }
  }
  ```

* **3）队列生产者代码**

  ```java
  package com.jicl.produce;
  
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.jms.core.JmsMessagingTemplate;
  import org.springframework.jms.core.JmsTemplate;
  import org.springframework.stereotype.Component;
  import javax.jms.Queue;
  import java.util.UUID;
  
  /**
   * 生产者（队列）
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:25
   */
  @Component
  public class QueueProducer {
      //JmsMessagingTemplate比JmsTemplate功能更加强大
      @Autowired
      private JmsMessagingTemplate jmsMessagingTemplate;
      //自定义的队列对象
      @Autowired
      private Queue queue;
  
      //生产消息
      public void prodeceMsg() {
          //生产消息到队列中
          jmsMessagingTemplate.convertAndSend(queue,
                  "queue---"+UUID.randomUUID().toString().substring(0, 6));
          System.out.println("生产消息成功！");
      }
  }
  ```

  

* **4）测试类代码**

  ```java
  package com.jicl;
  
  import com.jicl.produce.QueueProducer;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
  import org.springframework.test.context.web.WebAppConfiguration;
  
  /**
   * ActiveMQ测试类
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:33
   */
  @SpringBootTest(classes = SpringBootActivemqProducerApplication.class)
  @RunWith(SpringJUnit4ClassRunner.class)
  @WebAppConfiguration
  public class TestActiveMQ {
      @Autowired
      private QueueProducer queueProducer;
  
      @Test
      public void testProdeceMsg(){
          queueProducer.prodeceMsg();
      }
  }
  ```

* **5）执行测试代码**

  * **IDEA控制台**

    ```tex
    生产消息成功！
    ```

  * **ActiveMQ控制台**

    ![](http://bg.xianzilei.cn/SpringBoot%E6%95%B4%E5%90%88ActiveMQ%E4%B9%8B%E7%82%B9%E5%AF%B9%E7%82%B9%E7%94%A8%E4%BE%8B%E5%9B%BE-001.png)

    ​	**消息发送队列成功！**

## **2. 消费端服务实现**

消费端的代码非常简单

* **1）`application.yml`主配置文件（与生产端配置基本一致，唯一不同的是微服务端口号）**

  ```yaml
  server:
    port: 9999 #微服务端口号
  
  spring:
    activemq:
      broker-url: tcp://192.168.245.128:61616 #activemq服务器地址
      user: admin #用户名
      password: admin #密码
      pool:
        enabled: true #是否开启连接池
        max-connections: 100 #连接池最大连接数
    jms:
      pub-sub-domain: false #false(默认)=Queue true=Topic
  
  #自定义队列名
  myQueueName: xianzilei-queue-01
  ```

* **2）队列消费者代码**

  ```java
  package com.jicl.consumer;
  
  import org.springframework.jms.annotation.JmsListener;
  import org.springframework.stereotype.Component;
  import javax.jms.JMSException;
  import javax.jms.TextMessage;
  
  /**
   * 消费者（队列）
   *
   * @author : xianzilei
   * @date : 2020/2/19 19:15
   */
  @Component
  public class QueueConsumer {
  
      //消费消息
      @JmsListener(destination = "${myQueueName}")
      public void receiveMsg(TextMessage textMessage) throws JMSException {
          System.out.println("消费者接受到消息："+textMessage.getText());
      }
  }
  ```

* **3）启动消费端服务（运行启动类）**

  * **IDEA控制台**

    ```te
    消费者接受到消息：queue---f967d8
    ```

  * **ActiveMQ控制台**

    ![](http://bg.xianzilei.cn/SpringBoot%E6%95%B4%E5%90%88ActiveMQ%E4%B9%8B%E7%82%B9%E5%AF%B9%E7%82%B9%E7%94%A8%E4%BE%8B%E5%9B%BE-002.png)

    ​	**消费者消费成功！**

# **三、SpringBoot整合ActiveMQ之发布订阅（主题）**

## **1. 订阅端服务实现**

- **1）`application.yml`主配置文件（与队列不同是pub-sub-domain需要配置成true，表示主题）**

  ```yaml
  server:
    port: 7777 #微服务端口号
  
  spring:
    activemq:
      broker-url: tcp://192.168.245.128:61616 #activemq服务器地址
      user: admin #用户名
      password: admin #密码
      pool:
        enabled: true #是否开启连接池
        max-connections: 100 #连接池最大连接数
    jms:
      pub-sub-domain: true #false(默认)=Queue true=Topic
  
  #自定义队列名
  myTopicName: xianzilei-topic-01
  ```

- **2）主题订阅者代码**

  ```java
  package com.jicl.subscriber;
  
  import org.springframework.jms.annotation.JmsListener;
  import org.springframework.stereotype.Component;
  import javax.jms.JMSException;
  import javax.jms.TextMessage;
  
  /**
   * 消费者（主题）
   *
   * @author : xianzilei
   * @date : 2020/2/19 19:15
   */
  @Component
  public class TopicSubscriber {
  
      //消费消息
      @JmsListener(destination = "${myTopicName}")
      public void receiveMsg(TextMessage textMessage) throws JMSException {
          System.out.println("订阅者接受到消息："+textMessage.getText());
      }
  }
  
  ```

- **3）启动订阅端服务（运行启动类）**

  * **启动两个订阅端**

    ![](http://bg.xianzilei.cn/SpringBoot%E6%95%B4%E5%90%88ActiveMQ%E4%B9%8B%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E7%94%A8%E4%BE%8B%E5%9B%BE-001.png)

    **编写两个启动类，每启动一个换一个端口号**

  - **ActiveMQ控制台**

    ![](http://bg.xianzilei.cn/SpringBoot%E6%95%B4%E5%90%88ActiveMQ%E4%B9%8B%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E7%94%A8%E4%BE%8B%E5%9B%BE-002.png)

    ​	**可以看到两个订阅者！**

## **2. 发布端服务实现**

- **1）`application.yml`主配置文件（与队列不同是pub-sub-domain需要配置成true，表示主题）**

  ```yaml
  server:
    port: 6666 #微服务端口号
  
  spring:
    activemq:
      broker-url: tcp://192.168.245.128:61616 #activemq服务器地址
      user: admin #用户名
      password: admin #密码
      pool:
        enabled: true #是否开启连接池
        max-connections: 100 #连接池最大连接数
    jms:
      pub-sub-domain: true #false(默认)=Queue true=Topic
  
  #自定义队列名
  myTopicName: xianzilei-topic-01
  ```

- **2）`ActiveMQ`配置类**

  ```java
  package com.jicl.config;
  
  import org.apache.activemq.command.ActiveMQTopic;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.jms.annotation.EnableJms;
  import org.springframework.stereotype.Component;
  import javax.jms.Topic;
  
  /**
   * ActiveMQ配置类
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:24
   */
  @Component
  public class ActiveMQConfig {
      //自定义的主题名
      @Value("${myTopicName}")
      private String myTopicName;
  
      //自定义主题
      @Bean
      public Topic topic(){
          return new ActiveMQTopic(myTopicName);
      }
  }
  ```

- **3）主题发布者代码**

  ```java
  package com.jicl.publisher;
  
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.jms.core.JmsMessagingTemplate;
  import org.springframework.stereotype.Component;
  import javax.jms.Topic;
  import java.util.UUID;
  
  /**
   * 发布者（主题）
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:25
   */
  @Component
  public class TopicPublisher {
      //JmsMessagingTemplate比JmsTemplate功能更加强大
      @Autowired
      private JmsMessagingTemplate jmsMessagingTemplate;
      //自定义的主题对象
      @Autowired
      private Topic topic;
  
      public void prodeceMsg() {
          //生产消息到主题中
          jmsMessagingTemplate.convertAndSend(topic,
                  "topic---"+UUID.randomUUID().toString().substring(0, 6));
          System.out.println("发布消息成功！");
      }
  }
  ```

- **4）测试类代码**

  ```java
  package com.jicl;
  
  import com.jicl.publisher.TopicPublisher;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
  import org.springframework.test.context.web.WebAppConfiguration;
  
  /**
   * ActiveMQ测试类
   *
   * @author : xianzilei
   * @date : 2020/2/19 10:33
   */
  @SpringBootTest(classes = SpringBootActivemqProducerApplication.class)
  @RunWith(SpringJUnit4ClassRunner.class)
  @WebAppConfiguration
  public class TestActiveMQ {
      @Autowired
      private TopicPublisher topicPublisher;
  
      @Test
      public void testSendMsg(){
          //发布4条消息
          for (int i = 1; i < 5; i++) {
              topicPublisher.prodeceMsg();
          }
      }
  }
  ```

- **5）执行测试代码**

  - **IDEA控制台**

    ```shell
    #发布端日志
    发布消息成功！
    发布消息成功！
    发布消息成功！
    发布消息成功！
    
    #1号订阅端日志
    订阅者接受到消息：topic---5c4f16
    订阅者接受到消息：topic---e793ba
    订阅者接受到消息：topic---050ae9
    订阅者接受到消息：topic---4d7d21
    
    #2号订阅端日志
    订阅者接受到消息：topic---5c4f16
    订阅者接受到消息：topic---e793ba
    订阅者接受到消息：topic---050ae9
    订阅者接受到消息：topic---4d7d21
    ```

  - **ActiveMQ控制台**

    ![](http://bg.xianzilei.cn/SpringBoot%E6%95%B4%E5%90%88ActiveMQ%E4%B9%8B%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E7%94%A8%E4%BE%8B%E5%9B%BE-003.png)

    ​	**消息接收成功！**