# **一、ActiveMQ支持的协议**

## **1. 主要协议总览**

| 协议  | 描述                                                         |
| ----- | ------------------------------------------------------------ |
| TCP   | 默认的Broker配置                                             |
| NIO   | 和TCP协议类似，但NIO更侧重于底层的访问操作                   |
| AMQP  | Advanced Message Queuing Protocol，一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同开发语言等条件限制。 |
| STOMP | STOMP，Streaming Text Orientation Message Protocol，是流文本定向消息协议，是一种为MOM(Message Oriented Middleware，面向消息中间件)设计的简单文本协议。 |
| SSL   | 安全链接                                                     |
| MQTT  | MQTT(Message Queuing Telemetry Transport，消息队列遥测传输)是IBM开发的一个即时通讯协议，有可能成为物联网的重要组成部分。该协议支持所有平台，几乎可以把所有联网物品和外部连接起来，被用来当作传感器和致动器(比如通过Twitter让房屋联网)的通信协议。 |
| WS    | 使用新的HTML5 WebSockets与代理交换消息                       |

## **2. 常用的两个协议详解**

* **1）TCP(Transmission Control Protocal)**
  * **默认**的Broker配置，监听端口是**61616**
  * 在网络传输数据前，必须要序列化数据，消息是通过一个叫`wire protocol`的来序列化成字节流。默认情况下，`ActiveMQ`把`wire protocol`叫做`OpenWire`，它的目的是促使网络上的效率和数据快速交互。
  * **优点**：
    * TCP协议传输**可靠性高**，**稳定性强**
    * **高效性**：字节流方式传递，效率很高
    * **有效性**、可用性：应用广泛，支持任何平台
* **2）NIO(New I/O Protocal)**
  * 与TCP协议类似，但是更侧重于**底层的访问操作**
  * 它允许开发人员对同一资源可有**更多的client调用**和**服务器端有更多的负载**。
  * **适合使用NIO的场景**
    * **可能有大量的client连接到Broker上**，一般情况下，大量的连接是被操作系统所限制的。因此，NIO的实现比TCP需要更少的线程，所以建议使用NIO协议。
    * **可能对于Broker有一个很迟钝的网络传输**，NIO比TCP提供更好的性能。

# **二、配置ActiveMQ的协议和端口**

## **1. 默认配置**

ActiveMQ默认的协议配置在安装位置conf目录下的activemq.xml中，其中标签transportConnectors用来配置协议信息。以5.15.9版本为例，初始配置如下：

```xml
<transportConnectors>
    <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
    <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
</transportConnectors>
```

可以看到初始配置**默认支持TCP、AMQP、STOMP、MQTT、WS协议**。ActiveMQ并不是只能同时支持一种协议，而是可以**同时支持多种**，也就是说，连接到ActiveMQ服务器的**生产者和消费者可以是不同的协议**。

## **2. 添加NIO协议**

生产中为了提高MQ服务器的性能，我们大多会采用NIO的方式。

* ActiveMQ支持将NIO的协议添加到配置中，配置如下：

  ```xml
  <transportConnectors>
      <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
      <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <!-- 新加的NIO协议，端口设置为61618 -->
  	<transportConnector name="nio" uri="nio://0.0.0.0:61618?trace=true"/>
  </transportConnectors>
  ```

* 打开ActiveMQ控制台可以看到配置成功的NIO协议

  ![](http://img.xianzilei.cn/ActiveMQ%E5%8D%8F%E8%AE%AE%E7%94%A8%E4%BE%8B%E5%9B%BE-01.png)

* 代码配置修改：配置了NIO后，在使用时我们只需要将协议的前缀改为nio即可（并非其他协议也是，部分协议需要修改整套代码），如下所示

  ```yaml
  spring:
    activemq:
      broker-url: nio://192.168.245.128:61616 #activemq服务器地址
  ```

## **3. 配置ActiveMQ同时支持NIO和多种协议**

* URI格式以"nio"开头，代表这个端口使用TCP协议为基础的NIO网络模型。但是这样的设置方式，只能使这个端口支持`Openwire`协议。

* 也就是说如果仅仅配置`<transportConnector name="nio" uri="nio://0.0.0.0:61618?trace=true"/>`这一行，**仅支持以TCP协议为基础的NIO网络IO模型，其他协议（如AMQP、STOMP等）还是只能使用BIO网络IO模型**。

* 因此ActiveMQ对所有协议进行了优化，提供了**增强型NIO**。这个模式，让这个端口既支持NIO网络模型，又支持多个协议。其实就是使用`auto`关键字，相当于把所有协议**封装起来**，只**暴露一个端口**，使得使用起来更加灵活，用起来也很简单。配置如下：

  ```xml
  <transportConnectors>
      <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
      <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
      <!-- 新加的NIO协议，端口设置为61618 -->
  	<transportConnector name="nio" uri="nio://0.0.0.0:61618?trace=true"/>
      <!-- 增强型NIO，端口设置为61608 -->
      <transportConnector name="auto+nio" uri="auto+nio://0.0.0.0:61608"/>
  </transportConnectors>
  ```

* ActiveMQ控制台

  ![](http://img.xianzilei.cn/ActiveMQ%E5%8D%8F%E8%AE%AE%E7%94%A8%E4%BE%8B%E5%9B%BE-02.png)

* 代码配置

  ```yaml
  spring:
    activemq:
      broker-url: tcp://192.168.245.128:61608 #协议可以是tcp、nio、STOMP、AMQP、MQTT，只要端口是61608即可
  ```

  