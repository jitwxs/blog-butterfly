---
title: ActiveMQ 初探（2）——与 Spring 整合
tags: ActiveMQ
categories: Middleware
abbrlink: 5f77c427
date: 2018-05-08 22:50:53
related_repos:
  - name: activemq02
    url: https://github.com/jitwxs/blog-sample/tree/master/ActiveMQ/activemq02
---

## 一、环境准备

与 Spring 整合，除了原本的 `activemq-all` 外，还需导入 `spring-context-support` 和 `spring-jms` 包，如果 Spring 为5.0+，需要 `javax.jms-api` 依赖：

```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-all</artifactId>
    <version>5.15.3</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>5.0.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>5.0.4.RELEASE</version>
</dependency>
<!-- Spring 5.0及以上需要添加 -->
<dependency>
    <groupId>javax.jms</groupId>
    <artifactId>javax.jms-api</artifactId>
    <version>2.0.1</version>
</dependency>
```

新建配置文件 `cfg.properties`，并在其中添加ActiveMQ的相关信息：

```ini cfg.properties
activemq.broker_url=tcp://192.168.30.188:61616
activemq.queue_name=test-queue
activemq.topic_name=test-topic
```

编写 ActiveMQ 的配置文件 `applicationContext-activemq.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 加载配置文件 -->
    <context:property-placeholder location="classpath:cfg.properties"/>

    <!-- 真正可以产生Connection的ConnectionFactory，由对应的 JMS服务厂商提供 -->
    <bean id="targetConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="${activemq.broker_url}" />
    </bean>

    <!-- Spring用于管理真正的ConnectionFactory的ConnectionFactory -->
    <bean id="connectionFactory"
          class="org.springframework.jms.connection.SingleConnectionFactory">
        <!-- 目标ConnectionFactory对应真实的可以产生JMS Connection的ConnectionFactory -->
        <property name="targetConnectionFactory" ref="targetConnectionFactory" />
    </bean>

    <!-- Queue模式 -->
    <bean id="queueDestination" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg name="name" value="${activemq.queue_name}"/>
    </bean>
    <!--Topic模式 -->
    <bean id="topicDestination" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg name="name" value="${activemq.topic_name}"/>
    </bean>

    <!-- Spring提供的JMS工具类，它可以进行消息发送、接收等 -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <!-- 这个connectionFactory对应的是我们定义的Spring提供的那个ConnectionFactory对象 -->
        <property name="connectionFactory" ref="connectionFactory" />
    </bean>
</beans>
```

配置文件中核心就是 `jmsTemplate`，因为MQ中间件不止 ActiveMQ 一种，因此只要符合 JMS 规范，则都可以使用 `jmsTemplate` 来管理，只需要修改底层的实现就可以了（底层实现即 `targetConnectionFactory`）。

在配置文件中还准备了两个 `Destination` 对象，即 `queueDestination` 和 `topicDestination`，分别是对应了 Queue 模式和 Topic 模式。

## 二、消息发送

编写一个生产者的示例代码，步骤大概如下：

 1. 加载容器
 2. 从容器中获取 **jmsTemplate** 对象
 3. 从容器中获取 **Destination** 对象
 4. 发送消息

```java
@Test
public void testProducer() {
    // 1、加载Spring容器
    ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:spring/applicationContext-activemq.xml");
    // 2、获取jmsTemplate对象
    JmsTemplate jmsTemplate = ac.getBean(JmsTemplate.class);
    // 3、获取Destination对象（以Queue模式为例，如果要测试Topic更换下面注释即可）
    Destination destination = (Destination)ac.getBean("queueDestination");
    //Destination destination = (Destination)ac.getBean("topicDestination");
    // 4、发送消息
    jmsTemplate.send(destination, session -> session.createTextMessage("这是Spring与ActiveMQ整合的测试消息"));
}
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180415151538184.png)

## 三、消息接收

接收消息需要自定义一个实现 `MessageListener` 的类，在其中打印一句话：

```java
import javax.jms.Message;
import javax.jms.MessageListener;
import javax.jms.TextMessage;

public class MyMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        try {
            TextMessage textMessage = (TextMessage)message;
            System.out.println("收到消息：" + textMessage.getText());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

在Spring 容器中首先要注入 `MyMessageListener`，然后配置一个 `DefaultMessageListenerContainer`，指定 `connectionFactory`、`destination` 和 `messageListener`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 加载配置文件 -->
    <context:property-placeholder location="classpath:cfg.properties"/>

    <!-- 真正可以产生Connection的ConnectionFactory，由对应的 JMS服务厂商提供 -->
    <bean id="targetConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="${activemq.broker_url}" />
    </bean>

    <!-- Spring用于管理真正的ConnectionFactory的ConnectionFactory -->
    <bean id="connectionFactory"
          class="org.springframework.jms.connection.SingleConnectionFactory">
        <!-- 目标ConnectionFactory对应真实的可以产生JMS Connection的ConnectionFactory -->
        <property name="targetConnectionFactory" ref="targetConnectionFactory" />
    </bean>

    <!-- Queue模式 -->
    <bean id="queueDestination" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg name="name" value="${activemq.queue_name}"/>
    </bean>
    <!--Topic模式 -->
    <bean id="topicDestination" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg name="name" value="${activemq.topic_name}"/>
    </bean>

    <!-- Spring提供的JMS工具类，它可以进行消息发送、接收等 -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <!-- 这个connectionFactory对应的是我们定义的Spring提供的那个ConnectionFactory对象 -->
        <property name="connectionFactory" ref="connectionFactory" />
    </bean>

    <!-- 配置消费者 -->
    <bean id="myMessageListener" class="jit.wxs.activemq.MyMessageListener"/>
    <!-- 消息监听容器 -->
    <bean class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="connectionFactory" />
        <!-- 指定Destination，ref为上面定义的两个模式 -->
        <property name="destination" ref="queueDestination" />
        <!-- 指定Listener -->
        <property name="messageListener" ref="myMessageListener" />
    </bean>
</beans>
```

运行 `testProducer()` 方法，MessageListener 中接收到消息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180508224144573.png)
