---
title: Redis 初探（7）——Jedis 操纵集群
tags: Jedis
categories: 
  - 数据库
  - Redis
typora-root-url: ..
abbrlink: 5108d6b6
date: 2018-04-09 15:57:55
copyright_author: Jitwxs
---

在[《Redis 初探（2）——Jedis 的使用》](/baef2507.html)中，我们已经学会了Jedis操纵单机Redis的简单使用，本章将继续深入，介绍Jedis对集群的操纵。

## 一、Jedis 连接单机

在开始介绍 Jedis 连接集群之前，先简单回顾下连接单机的使用。

### 1.1 简单使用

```java
@Test
public void testJedis() {
    // 1.获得连接对象。参数为redis所在的服务器地址及端口号
    Jedis jedis = new Jedis("192.168.30.155", 6379);

    // 2.获得数据
    String age = jedis.get("age");
    System.out.println(age);

    jedis.close();

}
```

### 1.2 使用连接池

```java
@Test
public void testJedisPool() {
    //1. 创建Jedis连接池配置（包含许多配置，这里只配置了3个）
    JedisPoolConfig poolConfig = new JedisPoolConfig();
    // 设置最小和最大闲置个数
    poolConfig.setMinIdle(5);
    poolConfig.setMaxIdle(10);
    // 设置连接池最大个数
    poolConfig.setMaxTotal(30);

    //2. 创建Jedis连接池
    JedisPool pool = new JedisPool(poolConfig,"192.168.30.155", 6379);

    //3. 从连接池中获取Jedis对象
    Jedis jedis = pool.getResource();

    //4.操纵数据
    jedis.set("sex", "male");
    System.out.println(jedis.get("sex"));

    //5.关闭资源
    jedis.close();
    pool.close();
}
```

## 二、Jedis 连接集群

Jedis 连接集群也是十分简单，首先创建一个 `HostAndPort` 的集合，里面存放着集群中的每一个节点，然后创建 `JedisCluster` 对象，直接操纵该对象即可。

`JedisCluster` 在项目中**单例存在**即可。第三步的关闭可省略（因为单例存在，如果关掉了，即整个项目要结束了）。

```java
@Test
public void testJedisCluster() throws IOException {
    // 1.创建一个JedisCluster对象。第一个参数nodes是一个set类型，set中包含若干个HostAndPort对象
    Set<HostAndPort> nodes = new HashSet<>();
    nodes.add(new HostAndPort("192.168.30.155", 7001));
    nodes.add(new HostAndPort("192.168.30.155", 7002));
    nodes.add(new HostAndPort("192.168.30.155", 7003));
    nodes.add(new HostAndPort("192.168.30.155", 7004));
    nodes.add(new HostAndPort("192.168.30.155", 7005));
    nodes.add(new HostAndPort("192.168.30.155", 7006));
    JedisCluster cluster = new JedisCluster(nodes);
    // 2.直接使用JedisCluster对象操作redis，单例存在即可。
    cluster.set("test", "123");
    System.out.println(cluster.get("test"));
    // 3.关闭JedisCluster对象。
    cluster.close();
}
```

## 三、Jedis 的实际应用

在实际项目中，测试环境一般使用单机版，生产环境使用集群版。为了避免代码的不必要改动，我们使用`策略模式`，`面向接口开发`。

### 3.1 JedisClient 接口

定义一个 `JedisClient` 的接口，里面封装了常用的一些方法，主要是对 `String` 类型和 `Hash` 类型的操作，可以根据实际情况进行修改。

```java
import java.util.List;

public interface JedisClient {
    String set(String key, String value);
    String get(String key);
    Boolean exists(String key);
    Long expire(String key, int seconds);
    Long ttl(String key);
    Long incr(String key);
    Long hset(String key, String field, String value);
    String hget(String key, String field);
    Long hdel(String key, String... field);
    Boolean hexists(String key, String field);
    List<String> hvals(String key);
    Long del(String key);
}
```

### 3.2 单机实现类

```java
import java.util.List;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;

public class JedisClientPool implements JedisClient {
    
    private JedisPool jedisPool;

    public JedisPool getJedisPool() {
        return jedisPool;
    }

    public void setJedisPool(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
    }

    @Override
    public String set(String key, String value) {
        Jedis jedis = jedisPool.getResource();
        String result = jedis.set(key, value);
        jedis.close();
        return result;
    }

    @Override
    public String get(String key) {
        Jedis jedis = jedisPool.getResource();
        String result = jedis.get(key);
        jedis.close();
        return result;
    }

    @Override
    public Boolean exists(String key) {
        Jedis jedis = jedisPool.getResource();
        Boolean result = jedis.exists(key);
        jedis.close();
        return result;
    }

    @Override
    public Long expire(String key, int seconds) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.expire(key, seconds);
        jedis.close();
        return result;
    }

    @Override
    public Long ttl(String key) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.ttl(key);
        jedis.close();
        return result;
    }

    @Override
    public Long incr(String key) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.incr(key);
        jedis.close();
        return result;
    }

    @Override
    public Long hset(String key, String field, String value) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.hset(key, field, value);
        jedis.close();
        return result;
    }

    @Override
    public String hget(String key, String field) {
        Jedis jedis = jedisPool.getResource();
        String result = jedis.hget(key, field);
        jedis.close();
        return result;
    }

    @Override
    public Long hdel(String key, String... field) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.hdel(key, field);
        jedis.close();
        return result;
    }

    @Override
    public Boolean hexists(String key, String field) {
        Jedis jedis = jedisPool.getResource();
        Boolean result = jedis.hexists(key, field);
        jedis.close();
        return result;
    }

    @Override
    public List<String> hvals(String key) {
        Jedis jedis = jedisPool.getResource();
        List<String> result = jedis.hvals(key);
        jedis.close();
        return result;
    }

    @Override
    public Long del(String key) {
        Jedis jedis = jedisPool.getResource();
        Long result = jedis.del(key);
        jedis.close();
        return result;
    }

}
```

### 3.3 集群实现类

```java
import java.util.List;

import redis.clients.jedis.JedisCluster;

public class JedisClientCluster implements JedisClient {
    
    private JedisCluster jedisCluster;
    
    public JedisCluster getJedisCluster() {
        return jedisCluster;
    }

    public void setJedisCluster(JedisCluster jedisCluster) {
        this.jedisCluster = jedisCluster;
    }

    @Override
    public String set(String key, String value) {
        return jedisCluster.set(key, value);
    }

    @Override
    public String get(String key) {
        return jedisCluster.get(key);
    }

    @Override
    public Boolean exists(String key) {
        return jedisCluster.exists(key);
    }

    @Override
    public Long expire(String key, int seconds) {
        return jedisCluster.expire(key, seconds);
    }

    @Override
    public Long ttl(String key) {
        return jedisCluster.ttl(key);
    }

    @Override
    public Long incr(String key) {
        return jedisCluster.incr(key);
    }

    @Override
    public Long hset(String key, String field, String value) {
        return jedisCluster.hset(key, field, value);
    }

    @Override
    public String hget(String key, String field) {
        return jedisCluster.hget(key, field);
    }

    @Override
    public Long hdel(String key, String... field) {
        return jedisCluster.hdel(key, field);
    }

    @Override
    public Boolean hexists(String key, String field) {
        return jedisCluster.hexists(key, field);
    }

    @Override
    public List<String> hvals(String key) {
        return jedisCluster.hvals(key);
    }

    @Override
    public Long del(String key) {
        return jedisCluster.del(key);
    }

}
```

### 3.4 实战演示

（1）首先准备一个配置文件 `cfg.properties`，在配置文件中加入关于 redis 的信息：

```ini cfg.properties
# redis单机
redis.standalone.host=192.168.30.155
redis.standalone.port=6379
#redis集群
redis.cluster.01.host=192.168.30.155
redis.cluster.01.port=7001
redis.cluster.02.host=192.168.30.155
redis.cluster.02.port=7002
redis.cluster.03.host=192.168.30.155
redis.cluster.03.port=7003
redis.cluster.04.host=192.168.30.155
redis.cluster.04.port=7004
redis.cluster.05.host=192.168.30.155
redis.cluster.05.port=7005
redis.cluster.06.host=192.168.30.155
redis.cluster.06.port=7006
```

（2）准备 Spring 关于 redis 的配置文件 `applicationContext-redis.xml`：

需要注意的是，单机和集群同时**只能放开一个，另一个必须注释掉**。因为我们取Bean是取接口，即 `JedisClient`，这两个都是 `JedisClient` 的实现类。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 注意：单机和集群同时只能放开一个 -->

    <!-- 加载配置文件 -->
    <context:property-placeholder location="classpath:cfg.properties"/>

    <!-- 配置Redis单机 -->
    <bean id="jedisClientPool" class="jit.wxs.common.jedis.JedisClientPool">
        <property name="jedisPool" ref="jedisPool"/>
    </bean>
    <bean id="jedisPool" class="redis.clients.jedis.JedisPool">
        <constructor-arg name="host" value="${redis.standalone.host}"/>
        <constructor-arg name="port" value="${redis.standalone.port}"/>
    </bean>

    <!-- 配置Redis集群 -->
    <bean id="jedisClientCluster" class="jit.wxs.common.jedis.JedisClientCluster">
        <property name="jedisCluster" ref="jedisCluster"/>
    </bean>
    <bean id="jedisCluster" class="redis.clients.jedis.JedisCluster">
        <constructor-arg>
            <set>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.01.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.01.port}"/>
                </bean>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.02.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.02.port}"/>
                </bean>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.03.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.03.port}"/>
                </bean>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.04.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.04.port}"/>
                </bean>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.05.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.05.port}"/>
                </bean>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.06.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.06.port}"/>
                </bean>
            </set>
        </constructor-arg>
    </bean>
</beans>
```

（3）编写测试方法

```java
@Test
public void test() {
    ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:applicationContext-redis.xml");
    JedisClient jedisClient = ac.getBean(JedisClient.class);

    jedisClient.set("author", "jitwxs");
    String result = jedisClient.get("author");
    System.out.println(result);
}
```

因为我们是面向接口开发，因此当我们切换单机和集群时，这段代码不需要任何修改，只需要修改配置文件即可。
