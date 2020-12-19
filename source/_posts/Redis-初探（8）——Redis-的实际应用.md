---
title: Redis 初探（8）——Redis 的实际应用
categories: 
  - 数据库
  - Redis
abbrlink: 5907c9b5
date: 2018-04-10 01:32:07
copyright_author: Jitwxs
---

在[《Redis初探（7）——Jedis操纵集群》](/5108d6b6.html)中，我们已经学会了搭建 Redis 集群，以及使用策略模式，在xml文件中灵活切换单机版和集群版。

本章将演示在宜立方商城项目中使用 Redis，项目地址：[e3mall](https://github.com/jitwxs/e3mall)。

## 一、功能需求

商城首页访问量巨大，因为首页的大轮播图是从数据库查询获取的，**每次访问都要查询一次数据库**，数据库压力巨大，亟需缓存。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410005313843.png)

## 二、功能实现

实现之前首先思考 Redis 是要加在 Service 层还是 Web 层。理论上来说都可以，但是加在 Web 层的话，其他 Web 去调用 Service 还是得去查数据库，因此我们加在 Service 层。

其次思考使用什么数据类型，我们使用**哈希类型**，field 为类别的 id，value 为对应查询的查询内容。

### 2.1 配置文件 cfg.properties

首先在配置文件中加入 Redis 相关的信息，最后一项 `redis.CONTENT_KEY` 为我们首页轮播图缓存的 key 值：

```ini cfg.properties
#Redis单机
redis.standalone.host=192.168.30.155
redis.standalone.port=6379

#Redis集群
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

#Redis key相关
#用于存放tb_content表的缓存（哈希类型）
redis.CONTENT_KEY=CONTENT_KEY
```

### 2.2 Spring 中 Redis 配置

这里的代码在上一节已经说过了，因为我们是开发环境，使用单机版即可。

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
    <!--<bean id="jedisClientCluster" class="jit.wxs.common.jedis.JedisClientCluster">-->
        <!--<property name="jedisCluster" ref="jedisCluster"/>-->
    <!--</bean>-->
    <!--<bean id="jedisCluster" class="redis.clients.jedis.JedisCluster">-->
        <!--<constructor-arg>-->
            <!--<set>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.01.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.01.port}"/>-->
                <!--</bean>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.02.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.02.port}"/>-->
                <!--</bean>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.03.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.03.port}"/>-->
                <!--</bean>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.04.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.04.port}"/>-->
                <!--</bean>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.05.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.05.port}"/>-->
                <!--</bean>-->
                <!--<bean class="redis.clients.jedis.HostAndPort">-->
                    <!--<constructor-arg name="host" value="${redis.cluster.06.host}"/>-->
                    <!--<constructor-arg name="port" value="${redis.cluster.06.port}"/>-->
                <!--</bean>-->
            <!--</set>-->
        <!--</constructor-arg>-->
    <!--</bean>-->
</beans>
```

### 2.3 Service 层代码

首先我们注入了 `JedisClient`，然后从配置文件取到了 key 的名字 `CONTENT_KEY`。

在 `listByCategoryId()` 方法中，我们先查询 Redis 中是否有存在的 field，如果有，直接返回；如果没有，先查询数据库，然后存入缓存。

为了**保证缓存的同步**，在添加和删除方法中，我直接删除掉了相应 field 的缓存，这样当执行查询方法时，会重新保存缓存。

需要注意的是，**Redis 的正常/异常与否，不应当影响程序的正常运行**。因为即使没有 Redis 程序也是可以正常运行的，因此我们在 Redis 操作的地方，需要 `try-catch`，在 catch 中可以打印日志信息等操作，我这里只是简单的输出在控制台。

> 注：JedisClient 接口和其单机/集群实现类代码省略，需要请看上一节。

```java
@Service
public class TbContentServiceImpl extends ServiceImpl<TbContentMapper, TbContent> implements TbContentService {
    @Autowired
    private TbContentMapper contentMapper;

    @Autowired
    private JedisClient jedisClient;

    @Value("${redis.CONTENT_KEY}")
    private String CONTENT_KEY;

    /**
     * 删除CONTENT_KEY中指定field
     */
    private void deleteContentKeyFromRedis(Long cid) {
        try {
            jedisClient.hdel(CONTENT_KEY, cid + "");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public List<TbContent> listByCategoryId(Long cid) {
        try {
            // 如果缓存存在的话，直接从缓存中取
            String json = jedisClient.hget(CONTENT_KEY, cid + "");
            if(StringUtils.isNotBlank(json)) {
                return JsonUtils.jsonToList(json, TbContent.class);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        List<TbContent> contents = contentMapper.selectList(new EntityWrapper<TbContent>() .eq("category_id", cid));

        try {
            // 加入缓存
            jedisClient.hset(CONTENT_KEY, cid+"", JsonUtils.objectToJson(contents));
        } catch (Exception e) {
            e.printStackTrace();
        }

        return contents;
    }

    @Override
    public void addContent(TbContent tbContent) {
        // 更新缓存
        deleteContentKeyFromRedis(tbContent.getCategoryId());

        tbContent.setCreated(new Date());
        tbContent.setUpdated(new Date());
        
        contentMapper.insert(tbContent);
    }

    @Override
    public void deleteById(Long id) {
        if(id == null) {
            return;
        }
        // 更新缓存
        TbContent tbContent = contentMapper.selectById(id);
        deleteContentKeyFromRedis(tbContent.getCategoryId());

        contentMapper.deleteById(id);
    }
}
```

### 2.4 Web 层代码

我们设首页轮播图的 id 为 `ad1Id`，直接调用 `tbContentService.listByCategoryId(ad1Id)` 即可。

```java
@Controller
public class PageController {
    @Value("${ad1.id}")
    private Long ad1Id;

    @Autowired
    private TbContentService tbContentService;

    @RequestMapping("/index")
    public String showIndex(Model model) {
        // 得到首页大轮播图的List
        List<TbContent> ad1List = tbContentService.listByCategoryId(ad1Id);

        model.addAttribute("ad1List", ad1List);
        return "index";
    }
}
```

## 三、验证

服务器启动单机版 Redis，当我们刷新首页的时候，就会将缓存保存到了 Redis 中。

Key 为 `CONTENT_KEY`，field 目前只有一个，即首页轮播图，其值为89，value 为转换为 json 的数据：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410012837641.png)

当我在后台为首页添加一个轮播图后，该 Field 被删除掉了（这里之所以连 key 也被删掉了，是因为该 key 中只有一个field，因此唯一的 field 被删掉了，key 也就删掉了）：

```shell
127.0.0.1:6379> keys *
(empty list or set)
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410012308433.png)

重新刷新首页，正确显示三张：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410012539258.png)

再次查看 Redis：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180410012749749.png)
