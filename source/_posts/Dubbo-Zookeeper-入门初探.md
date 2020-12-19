---
title: Dubbo + Zookeeper 入门初探
tags:
  - Dubbo
  - Zookeeper
categories: 中间件
abbrlink: 5def036a
date: 2018-03-22 14:55:13
copyright_author: Jitwxs
---

>2018年2月15日，阿里巴巴的dubbo进入了Apache孵化器，社区的加入，希望dubbo能变得更好...

最近在学习一个分布式项目，使用到了`dubbo`，之前没有使用过，体验一下，分布式项目地址：[点击这里](https://github.com/jitwxs/e3mall)。

下面我使用[dubbo官网](http://dubbo.apache.org/)的一张图来介绍下`dubbo`（本人才开始学习，如有错误，欢迎指正）：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322102544516.png)

- Registry：**注册中心**，相当于**房产中介**，服务提供者和使用者都需要在这里注册/使用服务，我使用 `zookeeper` 实现。

- Monitor：**监控中心**，相当于**房产局**，它可以统计服务提供者和服务使用者的一些信息，及他们之间的关系，我使用 `dubbo admin` 实现。

- Provider：**服务提供者**，相当于**房东**，提供服务。

- Consumer：**服务消费者**，想当于**租户**，使用服务。

下面我通俗的解释下 dubbo 的整个流程，我**将服务比喻成房子**：

 **start**：dubbo 一启动，房东想好自己准备要租出去的房子

**register**：房东将房子拿到房产中介那边进行登记，并留下自己的联系方式

**subscribe**：租户告诉房产中介自己想租一个什么样的房子

**notify**：房产中介回复给租户符合条件的房子的房东的联系方式

 **invoke**：租户拿着联系方式去找房东租房子

 **count**：房产局全程监控着房东和租户之间的交易

其中：

- start、register、subscribe 在 dubbo 服务一启动就完成了

- notify、count 是异步执行的

- invoke 是同步执行的

## 一、搭建 Java 和 Tomcat 环境

这一步比较简单，直接跳过，不会的可以看下这篇文章：[Linux搭建JavaWeb开发环境（Java、Tomcat、MySQL）](http://blog.csdn.net/yuanlaijike/article/details/78877830)

## 二、搭建 Zookeeper 环境

我使用的是 `zookeeper-3.5.2-alpha`，[点我下载](http://mirrors.hust.edu.cn/apache/zookeeper/zookeeper-3.5.2-alpha/zookeeper-3.5.2-alpha.tar.gz)。

下载后将其解压：

```shell
wxs@ubuntu:~$ sudo tar zxf zookeeper-3.5.2-alpha.tar.gz
wxs@ubuntu:~$ sudo mv zookeeper-3.5.2-alpha /usr
wxs@ubuntu:~$ cd /usr/zookeeper-3.5.2-alpha
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha$ ls
bin          ivysettings.xml       recipes
build.xml    ivy.xml               src
CHANGES.txt  lib                   zookeeper-3.5.2-alpha.jar
conf         LICENSE.txt           zookeeper-3.5.2-alpha.jar.asc
contrib      NOTICE.txt            zookeeper-3.5.2-alpha.jar.md5
dist-maven   README_packaging.txt  zookeeper-3.5.2-alpha.jar.sha1
docs
```

在 zookper 文件夹下建立 `logs` 文件夹和 `data` 文件夹用于存放日志和数据：

```shell
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha$ sudo mkdir data
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha$ sudo mkdir logs
```

进入 `conf` 目录，复制一份 `zoo_sample.cfg` 为 `zoo.cfg`，对其进行修改：

```shell
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha$ cd conf/
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/conf$ cp zoo_sample.cfg zoo.cfg
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/conf$ vim zoo.cfg 
```

配置下 `dataDir` 和 `dataLogDir` 的路径，为之前创建的两个文件夹的路径，`clientPort` 使用默认的 2181 端口即可：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018032209423299.png)

我使用的是单机模式，没有配集群，这样就可以了。进入到`bin`目录，启动服务即可：

```shell
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/bin$ ./zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/zookeeper-3.5.2-alpha/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/bin$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/zookeeper-3.5.2-alpha/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: standalone
```

**小心踩坑：**

执行 `./zkServer.sh start` 时不要加 `sudo`，如果 root 用户配置文件没有配 `JAVA_HOME` 会出现找不到 `JAVA_HOME`！

```shell
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/bin$ sudo ./zkServer.sh start
Error: JAVA_HOME is not set and java could not be found in PATH.
```

**相关命令：**

>启动服务：start    停止服务： stop       重启服务； restart          查看状态：status

## 三、搭建 Dubbo 监控中心

**版本要求：**

请使用 `dubbo-admin-2.5.6.war` 及以上版本，否则会不支持JDK1.8！下载链接：[点击这里](https://download.csdn.net/download/yuanlaijike/10302261)

**小心踩坑：**

如果你的 `zookeeper` 和 `dubbo-admin` 在一台服务器上，`dubbo-admin` 不用修改任何内容！

如果不在一台服务器上，将 war 包解压后，修改项目 `/WEF-INF/dubbo.properties`文件，将 zookeeper 地址改为其所在服务器的地址（这里同时能修改 root 用户和 guest 用户的密码）。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322143242449.png)

## 四、配置项目

这里牵扯到项目代码，如果看不懂，可以下载文章开头的项目源码，或者直接使用官方提供的`dubbo-demo`，更为简单。

首先给服务提供方和服务使用方导入依赖包：

```xml
<properties>
    <dubbo.version>2.6.1</dubbo.version>
    <zookeeper.version>3.5.2-alpha</zookeeper.version>
    <curator.version>4.0.1</curator.version>
</properties>
    
<!-- dubbo包 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <!-- 排除dubbo自带的spring和netty，使用项目的，如果本身项目没有，无需排除 -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.jboss.netty</groupId>
            <artifactId>netty</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- zookeeper包 -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <type>pom</type>
</dependency>
<!-- curator(zookeeper的客户端)包 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-client</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
</dependency>
```

还需要在相关配置文件加上 `dubbo` 的 `bean` 的头部约束，将下面的添加到 bean 头部即可：

```xml
xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"

http://code.alibabatech.com/schema/dubbo
http://code.alibabatech.com/schema/dubbo/dubbo.xsd
```

### 4.1 服务提供方代码

对于服务提供方，如果我们想要 `TbItemService` 对外提供服务：

```java
package jit.wxs.service.impl;

import jit.wxs.pojo.TbItem;
import jit.wxs.mapper.TbItemMapper;
import jit.wxs.service.TbItemService;
import com.baomidou.mybatisplus.service.impl.ServiceImpl;
import org.springframework.stereotype.Service;

/**
 * <p>
 * 商品表 服务实现类
 * </p>
 *
 * @author jitwxs
 * @since 2018-03-21
 */
@Service
public class TbItemServiceImpl extends ServiceImpl<TbItemMapper, TbItem> implements TbItemService {

}
```

需要修改 spring 关于 service 的配置文件，加入 `dubbo` 的配置信息：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://code.alibabatech.com/schema/dubbo
	http://code.alibabatech.com/schema/dubbo/dubbo.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 扫描service层注解 -->
    <context:component-scan base-package="jit.wxs.service"/>

    <!-- dubbo发布服务 -->
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="e3-manager" />
    <!-- 配置zookeeper的地址，集群地址用逗号隔开 -->
    <dubbo:registry protocol="zookeeper" address="192.168.30.145:2181" />
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <!-- 声明需要暴露的服务接口
        ref：为注入的对应接口的bean
        timneout：超时时间，单位ms，开发模式可以设长一点方便debug
    -->
    <dubbo:service interface="jit.wxs.service.TbItemService" ref="tbItemServiceImpl" timeout="600000"/>
</beans>
```

- `dubbo:application`：提供方的应用名

- `dubbo:registry`：注册中心的类型和地址

- `dubbo:protocol`：这个服务要暴露在哪个端口上（使用方根据这个端口使用服务）

- `dubbo:service`：设置暴露的服务的接口，ref 为该接口的 bean，timeout 为超时时间

### 4.2 服务使用方代码

服务使用方，我使用 `Spring MVC` 来实现，修改 Spring MVC 的配置文件，加入 dubbo 的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
        http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 扫描组件 -->
    <context:component-scan base-package="jit.wxs.web"/>

    <!-- 注解驱动 -->
    <mvc:annotation-driven />

    <!-- 全局异常类 -->
    <!--<bean class="cn.edu.jit.exception.GlobalExceptionResolver"/>-->

    <!-- 视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/" />
        <property name="suffix" value=".jsp" />
    </bean>

    <!-- 引用dubbo服务 -->
    <!-- 使用方应用信息，用于计算依赖关系 -->
    <dubbo:application name="e3-manager-web"/>
    <!-- 指定zookeeper的地址，集群用逗号分隔 -->
    <dubbo:registry protocol="zookeeper" address="192.168.30.145:2181"/>
    <!-- 申明要访问的接口，并创建代理对象，注入bean，名为id的值 -->
    <dubbo:reference interface="jit.wxs.service.TbItemService" id="tbItemService" />
</beans>
```

- `dubbo:application`： 使用方的应用名

- `dubbo:registry`：注册中心的类型和地址

- `dubbo:reference`：要使用的服务的接口，并将返回的注入 bean，名称为id设的值

如果配置没有问题的话，现在使用方已经能够使用提供方提供的服务了，直接将 `tbItemService` 注入进来即可：

```java
package jit.wxs.web;

import jit.wxs.pojo.TbItem;
import jit.wxs.service.TbItemService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * <p>
 * 商品表 前端控制器
 * </p>
 *
 * @author jitwxs
 * @since 2018-03-21
 */
@RestController
@RequestMapping("/items")
public class TbItemController {
    @Autowired
    private TbItemService tbItemService;

    @GetMapping("/{id}")
    public TbItem getItemById(@PathVariable Long id) {
        TbItem item = null;
        if(id != null) {
            item = tbItemService.selectById(id);
        }

        return item;
    }
}
```

## 五、测试

服务器上分别启动 `zookper` 和 `tomcat`：

```shell
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/bin$ ./zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /usr/zookeeper-3.5.2-alpha/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
wxs@ubuntu:/usr/zookeeper-3.5.2-alpha/bin$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/zookeeper-3.5.2-alpha/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: standalone
```

```shell
wxs@ubuntu:/usr/apache-tomcat-8.5.28/bin$ ./startup.sh 
Using CATALINA_BASE:   /usr/apache-tomcat-8.5.28
Using CATALINA_HOME:   /usr/apache-tomcat-8.5.28
Using CATALINA_TMPDIR: /usr/apache-tomcat-8.5.28/temp
Using JRE_HOME:        /usr/jdk1.8.0_161/jre
Using CLASSPATH:       /usr/apache-tomcat-8.5.28/bin/bootstrap.jar:/usr/apache-tomcat-8.5.28/bin/tomcat-juli.jar
Tomcat started.
```

使用下面命令可以持续显示 `tomcat` 的输出：

>wxs@ubuntu:/usr/apache-tomcat-8.5.28/bin$ tail -f ../logs/catalina.out 

分别启动服务提供方的项目和服务使用方的项目：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018032211294940.png)

测试下 `/items/{id}` 这个API：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322113130841.png)

成功！下面再访问下监控中心，因为监控中心和 `zookeeper` 在一台服务器上，我的 tomcat 部署在 8888 端口，即访问 `192.168.30.145:8080/dubbo-admin` 即可，用户名密码默认为 root：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322142406319.png)

查看所有注册的服务：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322142709390.png)

查看包括消费者和提供者的所有应用名：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322142715245.png)

消费者、提供者详细信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180322142720610.png)
