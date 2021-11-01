---
title: ActiveMQ 初探（1）——介绍与基本使用
tags: ActiveMQ
categories: 中间件
abbrlink: 7050d1f6
date: 2018-04-15 16:24:45
related_repos:
  - name: activemq01
    url: https://github.com/jitwxs/blog-sample/tree/master/ActiveMQ/activemq01
---

## 一、ActiveMQ

### 1.1 什么是 ActiveMQ

`ActiveMQ` 是Apache出品，最流行的，能力强劲的`开源消息总线`。ActiveMQ 是一个完全支持 `JMS1.1` 和 `J2EE 1.4`规 范的 `JMS Provider` 实现，尽管 `JMS` 规范出台已经是很久的事情了,但是 JMS 在当今的 J2EE 应用中间仍然扮演着特殊的地位。

主要特点：

 1. 多种语言和协议编写客户端。语言: Java, C, C++, C#, Ruby, Perl, Python, PHP。应用协议: OpenWire,Stomp REST,WS Notification,XMPP,AMQP
 
 2. 完全支持 JMS1.1 和 J2EE 1.4 规范 (持久化,XA消息,事务)
 
 3. **对 Spring 的支持**，ActiveMQ 可以很容易内嵌到使用Spring的系统里面去,而且也支持 Spring2.0 的特性
 
 4. 通过了常见 J2EE 服务器(如 Geronimo,JBoss 4, GlassFish,WebLogic)的测试,其中通过 JCA 1.5 resource adaptors 的配置,可以让 ActiveMQ 可以自动的部署到任何兼容 J2EE 1.4 商业服务器上
 
 5. 支持多种传送协议:in-VM,TCP,SSL,NIO,UDP,JGroups,JXTA
 
 6. 支持通过 JDBC 和 journal 提供高速的消息持久化
 
 7. 从设计上保证了高性能的集群,客户端-服务器，点对点
 
 8. 支持 Ajax
 
 9. 可以很容易得调用内嵌 JMS provider,进行测试

### 1.2 消息形式

- `点对点模式`，即**一个生产者和一个消费者一一对应**

- `发布/订阅模式`，即**一个生产者产生消息并进行发送后，可以由多个消费者进行接收**

JMS定义了五种不同的消息正文格式，以及调用的消息类型，允许你发送并接收以一些不同形式的数据，提供现有消息格式的一些级别的兼容性。

- `StreamMessage` -- Java原始值的数据流

- `MapMessage`--键值对

- `TextMessage`--字符串对象 （主要使用，什么都可以转JSON字符串）

- `ObjectMessage`--序列化的 Java对象

- `BytesMessage`--字节的数据流

## 二、安装 ActiveMQ

本次测试版本为当前最新的 `ActiveMQ 5.15.3 Release`，[点击下载](http://activemq.apache.org/activemq-5153-release.html)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413102919639.png)

解压并放在 `usr/local` 目录下：

```shell
root@ubuntu:/home/wxs# tar zxvf apache-activemq-5.15.3-bin.tar.gz 
root@ubuntu:/home/wxs# mv apache-activemq-5.15.3 /usr/local/activemq
root@ubuntu:/home/wxs# cd /usr/local/activemq/
root@ubuntu:/usr/local/activemq# ls
activemq-all-5.15.3.jar  conf  docs      lib      NOTICE      webapps
bin                      data  examples  LICENSE  README.txt  webapps-demo
```

启动 `ActiveMQ`（需要jdk支持）：

>启动：./activemq start
>停止：./activemq stop

```shell
root@ubuntu:/usr/local/activemq# cd bin/
root@ubuntu:/usr/local/activemq/bin# ./activemq start
INFO: Loading '/usr/local/activemq//bin/env'
INFO: Using java '/usr/local/jdk1.8.0_161/bin/java'
INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
INFO: pidfile created : '/usr/local/activemq//data/activemq.pid' (pid '6914')
```

ActiveMQ 默认运行在 `8161` 端口，点击首页的 `Manage ActiveMQ broker` 进入后台管理系统，用户名：**admin**，密码：**admin**.

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413103707326.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413104813304.png)

## 三、Queue

前面说过 `ActiveMQ` 有两种消息模式，即：

- `点对点模式`，即**一个生产者和一个消费者一一对应**

- `发布/订阅模式`，即**一个生产者产生消息并进行发送后，可以由多个消费者进行接收**

首先介绍下`点对点模式`，即 `Queue`。

首先导入 ActiveMQ 依赖：

```xml
<dependency>
  <groupId>org.apache.activemq</groupId>
  <artifactId>activemq-all</artifactId>
  <version>5.15.3</version>
</dependency>
```

### 3.1 Producer

首先我们编写下消息的发布者（`Producer`），因为 `ActiveMQ` 遵循 `JMS` 规范，所以步骤比较繁琐，步骤如下：

 1. **创建一个连接工厂对象**，指定服务 IP 和端口
 2. 使用工厂对象**创建 Collection 对象**
 3. **开启连接**
 4. **创建 Session 对象**
 5. 使用 Session 对象**创建 Destination 对象**
 6. 使用 Session 对象**创建 Producer 对象**
 7. **创建 Message 对象**
 8. **发送消息**
 9. 关闭资源

编写一个测试方法：

```java
@Test
public void testQueueProducer() throws Exception {
    //1、创建一个连接工厂对象，指定服务IP和端口
    // 这里的端口不是8161，而是ActiveMQ服务端口，默认为61616
    String brokerURL = "tcp://192.168.30.155:61616";
    ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(brokerURL);
    //2、使用工厂对象创建Collection对象
    Connection connection = connectionFactory.createConnection();
    //3、开启连接，调用Collection.start()
    connection.start();
    //4、创建Session对象
    // 参数1：是否开启事务，如果为true，参数2无效
    // 参数2：应答模式，自动应答/手动应答，一般自动应答即可
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
    //5、使用Session对象创建Destination对象（实现类：queue或topic）
    Queue queue = session.createQueue("test-queue");
    //6、使用Session对象创建Producer对象
    MessageProducer producer = session.createProducer(queue);
    //7、创建Message对象
    TextMessage message = session.createTextMessage("It just a test queue...");
    //8、发送消息
    producer.send(message);
    //9、关闭资源
    producer.close();
    session.close();
    connection.close();
}
```

执行后查看 ActiveMQ 后台，选择 `Queue` 选项卡可以看到我们创建的 Queue 对象 test-queue：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413133733442.png)

点进去可以看到发送的消息内容：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413133723384.png)

### 3.2 Consumer

现在我们已经把消息发送出去了，但是还没有消费者去接收这个消息，下面开始编写消费者，步骤如下：

 1. **创建一个连接工厂对象**，指定服务IP和端口
 2. 使用工厂对象**创建 Collection 对象**
 3. **开启连接**
 4. **创建 Session 对象**
 5. 使用 Session 对象**创建 Destination 对象**
 6. 使用 Session 对象**创建 Consumer 对象**
 7. **接收消息**
 8. 关闭资源

可以看到1 ~ 5步和之前都是一样的，编写测试方法：

```java
@Test
public void testQueueConsumer() throws Exception {
    //1、创建一个连接工厂对象，指定服务IP和端口
    // 这里的端口不是8161，而是ActiveMQ服务端口，默认为61616
    String brokerURL = "tcp://192.168.30.155:61616";
    ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(brokerURL);
    //2、使用工厂对象创建Collection对象
    Connection connection = connectionFactory.createConnection();
    //3、开启连接，调用Collection.start()
    connection.start();
    //4、创建Session对象
    // 参数1：是否开启事务，如果为true，参数2无效
    // 参数2：应答模式，自动应答/手动应答，自动应答即可
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
    //5、使用Session对象创建Destination对象（queue或topic）
    Queue queue = session.createQueue("test-queue");
    //6、使用Session对象创建一个Consumer对象
    MessageConsumer consumer = session.createConsumer(queue);
    //7、接收消息
    consumer.setMessageListener(message -> {
        try {
            TextMessage msg = (TextMessage) message;
            System.out.println("接收到消息：" + msg.getText());
        } catch (JMSException e) {
            e.printStackTrace();
        }
    });
    //阻塞程序，避免结束
    System.in.read();
    //8、关闭资源
    consumer.close();
    session.close();
    connection.close();
}
```

运行程序，接收到了之前的消息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413135654960.png)

此时重新查看后台，发现 `Number Of Pending Messages` 的值已经变成 0，说明刚刚那条消息已经被消费掉了，`Number Of Consumers` 值变为了 1。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413135757928.png)

## 四、Topic

下面介绍下`发布/订阅模式`，即 `Topic`，当一个生产者发送了消息后。多个订阅了该生产者的消费者都会收到消息。

使用 `Queue` 时，因为是一对一，所以消费者没有接收的话消息会被保存到 ActiveMQ 后台，我们可以查看得到，如上面的图示。

但是 `Topic` 不一样，ActiveMQ 后台是不会保存消息的，因此**如果消费者没有接收的话，这个消息就丢失了**。

因此，我们必须**先启动消费者，生产者再发送消息**，消费者我才能收到。

### 4.1 Producer

Topic 的生产者和 Queue 生产者的步骤几乎一样，只是第 5 步创建 `Destination` 对象的实现类修改为 Topic 即可：

```java
@Test
public void testTopicProducer() throws Exception {
    //1、创建一个连接工厂对象，指定服务IP和端口
    // 这里的端口不是8161，而是ActiveMQ服务端口，默认为61616
    String brokerURL = "tcp://192.168.30.155:61616";
    ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(brokerURL);
    //2、使用工厂对象创建Collection对象
    Connection connection = connectionFactory.createConnection();
    //3、开启连接，调用Collection.start()
    connection.start();
    //4、创建Session对象
    // 参数1：是否开启事务，如果为true，参数2无效
    // 参数2：应答模式，自动应答/手动应答，自动应答即可
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
    //5、使用Session对象创建Destination对象（queue或topic）
    Topic topic = session.createTopic("test-topic");
    //6、使用Session对象创建一个Producer对象
    MessageProducer producer = session.createProducer(topic);
    //7、创建一个Message对象
    TextMessage message = session.createTextMessage("It just a test topic...");
    //8、发送消息
    producer.send(message);
    //9、关闭资源
    producer.close();
    session.close();
    connection.close();
}
```

进入后台查看，test-topic 拥有 0 个消费者，1 个消息入队，0 个消息入队。前面说过 ActiveMQ 后台是不会保存 Topic 的消息的，所以我们刚刚发送的消息因为没有消费者就丢失了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180413141016830.png)

### 4.2 Consumer

和 Queue 的 Consumer 差不多，只是修改第5步改为创建 Topic 即可：

```java
@Test
public void testTopicConsumer() throws Exception {
    //1、创建一个连接工厂对象，指定服务IP和端口
    // 这里的端口不是8161，而是ActiveMQ服务端口，默认为61616
    String brokerURL = "tcp://192.168.30.155:61616";
    ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(brokerURL);
    //2、使用工厂对象创建Collection对象
    Connection connection = connectionFactory.createConnection();
    //3、开启连接，调用Collection.start()
    connection.start();
    //4、创建Session对象
    // 参数1：是否开启事务，如果为true，参数2无效
    // 参数2：应答模式，自动应答/手动应答，自动应答即可
    Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
    //5、使用Session对象创建Destination对象（queue或topic）
    Topic topic = session.createTopic("test-topic");
    //6、使用Session对象创建一个Consumer对象
    MessageConsumer consumer = session.createConsumer(topic);
    //7、接收消息
    consumer.setMessageListener(message -> {
        try {
            TextMessage msg = (TextMessage) message;
            System.out.println("接收到消息：" + msg.getText());
        } catch (JMSException e) {
            e.printStackTrace();
        }
    });
    //阻塞程序，避免结束
    System.in.read();
    //8、关闭资源
    consumer.close();
    session.close();
    connection.close();
}
```
