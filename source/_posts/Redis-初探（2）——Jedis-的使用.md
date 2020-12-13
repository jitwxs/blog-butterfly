---
title: Redis 初探（2）——Jedis 的使用
tags: Jedis
categories: 
  - 数据库
  - Redis
typora-root-url: ..
abbrlink: baef2507
date: 2018-02-28 00:13:53
copyright_author: Jitwxs
---

### 2.1 准备工作

首先我们在Linux中开启redis服务：

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-server ../redis.conf 
wxs@ubuntu:/usr/local/redis/src$ ps auxc | grep redis
wxs        5278  0.0  0.4  51828  8408 ?        Ssl  20:12   0:00 redis-server
```

上一节中我们只是在Linux主机中简单的测试了下 Redis 的使用，但是那样并没有什么实际用处，我们可以在程序中去远程的访问 Redis。

我们以 Java 为例，想要在 Java 程序中使用 Redis，首先我们需要借助第三方库 `jedis`，导入需要的包：

- jedis [点我下载](http://www.mvnjar.com/redis.clients/jedis/2.9.0/detail.html)
- commons-pool [点我下载](http://commons.apache.org/proper/commons-pool/download_pool.cgi)

为了方便测试，还需导包：

-  junit [点我下载](http://www.mvnjar.com/junit/junit/4.12/detail.html)
- hamcrest-all（可能用到） [点我下载](http://repo2.maven.org/maven2/org/hamcrest/hamcrest-all/1.3/)

### 2.2 第一个Jedis程序

导完包后，我们编写第一个 Jedis 程序，代码如下：

```java
import org.junit.Test;

import redis.clients.jedis.Jedis;

/**
 * @className JedisTest.java
 * @author jitwxs
 * @version 创建时间：2018年2月27日 下午8:21:10   
*/
public class JedisTest {
    @Test
    public void test1() {
        // 1.获得连接对象。参数为redis所在的服务器地址及端口号
        Jedis jedis = new Jedis("192.168.111.152", 6379);

        // 2.获得数据
        String userName = jedis.get("userName");
        System.out.println(userName);
        
        jedis.close();
    }
}
```

是不是十分简单，首先我们获取到了 Jedis 的实例，需要传入的参数也是十分易懂，即运行 Redis 的主机的 IP 地址和 Redis 的端口号。然后直接使用 get 方法就获取到了 userName 的 value 值。

运行程序，如果你没有导入上面的 `hamcrest-all` 包的话，可能会出现这个错误：

![](/images/posts/20180227204212520.png)

导入后，运行结果如下：

![](/images/posts/20180227204332240.png)

很不幸，再次出现了错误，错误原因是说连接被拒绝，这是因为我们的 Redis 的配置文件中**默认只允许本机访问**，停 止Redis 服务，修改配置文件：

```shell
wxs@ubuntu:/usr/local/redis/src$ ./redis-cli shutdown
wxs@ubuntu:/usr/local/redis/src$ sudo vim ../redis.conf 
```

将配置文件中的 `bind` 参数修改为0.0.0.0，即所有主机都能访问：

![](/images/posts/20180227211443295.png)

重新启动 Redis，并运行 Java 程序：

![](/images/posts/20180227214326760.png)

我这里已经能够成功取出值了，如果这个时候你还是出现了错误，且错误信息是连接超时（connention timeout...）,可能是因为Linux主机**没有开放 Redis 的端口**，导致我们远程连接被拒绝。以 iptables 为例：

```shell
wxs@ubuntu:~$sudo iptables -A INPUT -p tcp –dport 6379 -j ACCEPT #开放本机6379端口
```

### 2.3 Jedis连接池

每使用一次就 new 一个 Jedis 实例感觉不是很靠谱，Jedis 和数据库连接库一样，都有连接池，即 `JedisPool` ，它的配置很简单。配置好连接池后以后直接从连接池中取 Jedis 对象就可以了。测试代码如下：

```java
@Test
public void test2() {
    //1. 创建Jedis连接池配置（包含许多配置，这里只配置了3个）
    JedisPoolConfig poolConfig = new JedisPoolConfig();
    // 设置最小和最大闲置个数
    poolConfig.setMinIdle(5);
    poolConfig.setMaxIdle(10);
    // 设置连接池最大个数
    poolConfig.setMaxTotal(30);
    
    //2. 创建Jedis连接池
    JedisPool pool = new JedisPool(poolConfig,"192.168.111.152", 6379);
    
    //3. 从连接池中获取Jedis对象
    Jedis jedis = pool.getResource();
    
    //4.操纵数据
    jedis.set("age", "20");
    System.out.println(jedis.get("age"));
    
    //5.关闭资源
    jedis.close();
    pool.close();
}
```

![](/images/posts/20180227234859583.png)

### 2.4 封装成工具类

上面完成了 Redis 的基本使用，我们可以把上面的 Redis 连接池封装成一个工具类，其他地方的代码直接调用工具类来执行即可。

首先在 src 下面新建一个配置文件 redis.properties，把一些配置信息存储在配置文件中：

```ini redis.properties
#Redis连接信息
redis.ip = 192.168.111.152
redis.port = 6379

#Redis配置信息
redis.minIdle = 5
redis.maxIdle = 10
redis.maxTotal = 30
```

新建 JedisUtils.java：

```java
import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

/**
 * @className JedisUtils.java
 * @author jitwxs
 * @version 创建时间：2018年2月27日 下午11:56:55   
*/
public class JedisUtils {
    private static JedisPool jedisPool = null;
    
    /**
     * 获取连接
     * @author jitwxs
     * @version 创建时间：2018年2月28日 上午12:06:00 
     * @return
     */
    public static Jedis getJedis() {
        if(jedisPool == null) {
            initJedisPool();
        }
        return jedisPool.getResource();
    }
    
    /**
     * 关闭连接
     * @author jitwxs
     * @version 创建时间：2018年2月28日 上午12:06:59 
     * @param jedis
     */
    public static void closeJedis(Jedis jedis) {  
        jedis.close();  
    }
    
    private static void initJedisPool() {
        InputStream in = JedisUtils.class.getClassLoader().getResourceAsStream("redis.properties");
        Properties properties = new Properties();
        try {
            properties.load(in);
            JedisPoolConfig poolConfig = new JedisPoolConfig();
            poolConfig.setMinIdle(Integer.parseInt(properties.getProperty("redis.minIdle")));
            poolConfig.setMaxIdle(Integer.parseInt(properties.getProperty("redis.maxIdle")));
            poolConfig.setMaxTotal(Integer.parseInt(properties.getProperty("redis.maxTotal")));

            jedisPool = new JedisPool(poolConfig, properties.getProperty("redis.ip"),
                    Integer.parseInt(properties.getProperty("redis.port")));
        } catch (IOException e) {
            System.out.println("载入配置文件错误");
            e.printStackTrace();
        }
    }
}

```

编写测试用例，运行程序：

```java
@Test
public void test3() {
    Jedis jedis = JedisUtils.getJedis();
    System.out.println(jedis.get("age"));
    JedisUtils.closeJedis(jedis);
}
```

![](/images/posts/20180228001213330.png)
