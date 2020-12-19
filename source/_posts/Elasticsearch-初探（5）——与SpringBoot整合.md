---
title: Elasticsearch 初探（5）——与SpringBoot整合
categories:
  - 搜索引擎
  - Elasticsearch
tags: [Elasticsearch, SpringBoot]
abbrlink: 79a2adb2
date: 2018-10-09 17:13:33
related_repos:
  - name: springboot_es
    url: https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/springboot_es
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、环境搭建

采用SpringBoot 2.0 + Elasticsearch 6.4.1。本文只列举了其中一些API，更多API请参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/master/java-rest-high-supported-apis.html)

### 1.1 导入依赖

注意`SpringBoot 2.0.5.RELEASE` 默认依赖的Elasticsearch版本是5.6.11，因此不要使用`springboot-starter-data-elasticsearch`，需要手动导入相关依赖。

```xml
<!-- elasticSearch版本 -->
<elasticSearch.version>6.4.1</elasticSearch.version>

<!-- elasticsearch依赖 -->
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>${elasticSearch.version}</version>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>transport</artifactId>
    <version>${elasticSearch.version}</version>
    <!-- 排除Springboot 2.0默认的elasticsearch依赖，默认依赖的是5.6.11 -->
    <exclusions>
        <exclusion>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 不使用Springboot 2.0默认的transport-netty4-client，默认版本是5.6.11 -->
<dependency>
    <groupId>org.elasticsearch.plugin</groupId>
    <artifactId>transport-netty4-client</artifactId>
    <version>${elasticSearch.version}</version>
</dependency>
<!-- elasticsearch高级别 API -->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>${elasticSearch.version}</version>
</dependency>
```

以上为需要用到的包，一共4个，需要解释下：

1. `org.elasticsearch.client.transport`默认会依赖自带的`org.elasticsearch.elasticsearch`，之前说过版本是5.x，因此必须要排除掉，使用我们手动导入的来替代。

2. 实际用于操纵的`TransportClient`默认依赖的仍然是5.x版本的transport-netty4-client，因此要手动导入。

3. 我们通过`elasticsearch-rest-high-level-client`提供的API来操纵Elasticsearch，关于该包，可以参考文章：[使用Java High Level REST Client操作elasticsearch](https://www.cnblogs.com/ginb/p/8716485.html)。

### 1.2 配置Elasticsearch

首先在配置文件中配置集群的相关信息：

```ini
elasticsearch.clusterName=my-es-cluster
elasticsearch.node1.ip=127.0.0.1
elasticsearch.node1.port=9300
elasticsearch.node2.ip=127.0.0.1
elasticsearch.node2.port=9301
elasticsearch.node3.ip=127.0.0.1
elasticsearch.node3.port=9302
```

然后编写ElasticSearchConfig 类进行相关配置：

```java
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.TransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.net.InetAddress;

@Configuration
public class ElasticSearchConfig {
    private Logger logger  = LoggerFactory.getLogger(this.getClass());

    @Value("${elasticsearch.node1.ip}")
    private String firstIp;
    @Value("${elasticsearch.node2.ip}")
    private String secondIp;
    @Value("${elasticsearch.node3.ip}")
    private String thirdIp;
    @Value("${elasticsearch.node1.port}")
    private String firstPort;
    @Value("${elasticsearch.node2.port}")
    private String secondPort;
    @Value("${elasticsearch.node3.port}")
    private String thirdPort;
    @Value("${elasticsearch.clusterName}")
    private String clusterName;

    @Bean
    public TransportClient getTransportClient() {
        logger.info("ElasticSearch初始化开始。。");
        logger.info("要连接的节点1的ip是{}，端口是{}，集群名为{}" , firstIp , firstPort , clusterName);
        logger.info("要连接的节点2的ip是{}，端口是{}，集群名为{}" , secondIp , secondPort , clusterName);
        logger.info("要连接的节点3的ip是{}，端口是{}，集群名为{}" , thirdIp , thirdPort , clusterName);
        TransportClient transportClient = null;
        try {
            Settings settings = Settings.builder()
                    //集群名称
                    .put("cluster.name",clusterName)
                    //目的是为了可以找到集群，嗅探机制开启
                    .put("client.transport.sniff",true)
                    .build();
            transportClient = new PreBuiltTransportClient(settings);

            TransportAddress firstAddress = new TransportAddress(InetAddress.getByName(firstIp),Integer.parseInt(firstPort));
            TransportAddress secondAddress = new TransportAddress(InetAddress.getByName(secondIp),Integer.parseInt(secondPort));
            TransportAddress thirdAddress = new TransportAddress(InetAddress.getByName(thirdIp),Integer.parseInt(thirdPort));
            transportClient.addTransportAddress(firstAddress);
            transportClient.addTransportAddress(secondAddress);
            transportClient.addTransportAddress(thirdAddress);
            logger.info("ElasticSearch初始化完成。。");
        }catch (Exception e){
            e.printStackTrace();
            logger.error("ElasticSearch初始化失败：" +  e.getMessage(),e);
        }
        return transportClient;
    }
}
```

我们所有的操纵都是依赖于`TransportClient`来进行，因此当我们在某个类中要使用Elasticsearch时，都需要引入`TransportClient`：

```java
@Resource
private TransportClient transportClient;
```

## 二、操纵索引

假设我想创建一个索引名为book，类型为 it，包含书名name、作者author、价格price、出版时间publish_date四个属性。

### 2.1 创建索引

首先规范一下创建请求的数据格式，假设如下：

```json
{
	"index": "book",
	"type":"it",
	"properties": {
		"name": "text",
		"author": "text",
		"price": "integer",
		"publish_date": "date"
	}
}
```

**（1）设置索引的setting部分**

```java
XContentBuilder settings = XContentFactory.jsonBuilder()
	.startObject()
	.field("number_of_shards",6)
	.field("number_of_replicas",1)
	.startObject("analysis").startObject("analyzer").startObject("ik")
	.field("tokenizer","ik_max_word")
	.endObject().endObject().endObject()
	.endObject();
```

以上代码就相当于配置创建索引的setting部分如下：

```json
"settings":{
  "number_of_shards": "6",
  "number_of_replicas": "1",
  "analysis":{
    "analyzer":{
      "ik":{
        "tokenizer":"ik_max_word"
      }
    }
  }
}
```

即指定了分6片，1份备份，使用IK分词器，分词模式为ik_max_word。

**（2）设置索引的mapping部分**

```java
XContentBuilder mapping = XContentFactory.jsonBuilder();
mapping.startObject().field("dynamic","strict").startObject("properties");

for(Map.Entry<String, String> entry : properties.entrySet()) {
    String field = entry.getKey();
    String fieldType = entry.getValue();
    mapping.startObject(field).field("type",fieldType);
    // 对于日期属性增加format
    if("date".equals(fieldType.trim())){
        mapping.field("format","yyyy-MM-dd HH:mm:ss || yyyy-MM-dd ");
    }
    mapping.endObject();
}

mapping.endObject().endObject();
```

代码中的properties就是请求中的properties，以上代码就相当于配置创建索引的mapping部分如下：

```json
"mappings":{
    "dynamic":"strict",
    "properties":{
        "name":{
          "type":"text"
        },
        "author":{
          "type":"text"
        },
        "price":{
          "type":"integer"
        },
        "publishDate":{
          "type":"date",
          "format":"yyyy-MM-dd HH:mm:ss || yyyy-MM-dd"
        }
    }
}
```

**（3）创建索引**

```java
CreateIndexRequest createIndexRequest = Requests.createIndexRequest(index).settings(settings).mapping(type,mapping);
CreateIndexResponse response = transportClient.admin().indices().create(createIndexRequest).get();
```

我们可以通过`response.isAcknowledged()`返回值判断是否成功，或者通过捕获异常也可以判断是否成功。

**（4）完整代码**

```java
private final String INDEX = "index";
private final String TYPE = "type";
private final String PROPERTIES = "properties";
private Logger logger = LoggerFactory.getLogger(this.getClass());

@Resource
private TransportClient transportClient;

/**
 * 创建索引
 * map包含：index、type、properties。其中properties为map，key：属性名；value：类型
 * @author jitwxs
 * @since 2018/10/9 15:07
 */
@PostMapping("/index/create")
public ResultBean createIndex(@RequestBody Map param) {
    logger.info("接收的创建索引的参数：" + param);

    String index = (String)param.get(INDEX);
    String type = (String)param.get(TYPE);
    Map<String, String> properties = (Map<String, String>) param.get(PROPERTIES);

    if(StringUtils.isBlank(index) || StringUtils.isBlank(type) || properties == null || properties.size() == 0){
        return ResultBean.error("参数错误！");
    }

    try {
        XContentBuilder settings = XContentFactory.jsonBuilder()
                .startObject()
                .field("number_of_shards",6)
                .field("number_of_replicas",1)
                .startObject("analysis").startObject("analyzer").startObject("ik")
                .field("tokenizer","ik_max_word")
                .endObject().endObject().endObject()
                .endObject();

        XContentBuilder mapping = XContentFactory.jsonBuilder();
        mapping.startObject().field("dynamic","strict").startObject("properties");

        for(Map.Entry<String, String> entry : properties.entrySet()) {
            String field = entry.getKey();
            String fieldType = entry.getValue();
            mapping.startObject(field).field("type",fieldType);
            // 对于日期属性增加format
            if("date".equals(fieldType.trim())){
                mapping.field("format","yyyy-MM-dd HH:mm:ss || yyyy-MM-dd ");
            }
            mapping.endObject();
        }
        mapping.endObject().endObject();

        CreateIndexRequest createIndexRequest = Requests.createIndexRequest(index).settings(settings).mapping(type,mapping);
        CreateIndexResponse response = transportClient.admin().indices().create(createIndexRequest).get();

        logger.info("建立索引映射成功：" + response.isAcknowledged());
        return ResultBean.success("创建索引成功！");
    } catch (Exception e) {
        logger.error("创建索引失败！要创建的索引为{}，文档类型为{}，异常为：",index,type,e.getMessage(),e);
        return ResultBean.error("创建索引失败！");
    }
}
```

```json
POST /index/create
{
	"index": "book",
	"type":"it",
	"properties": {
		"name": "text",
		"author": "text",
		"price": "integer",
		"publish_date": "date"
	}
}

{
    "status": true,
    "msg": "创建索引成功！",
    "data": null
}
```

### 2.2 删除索引

删除索引只需要提供索引名即可，较为简单，代码如下：

```java
@DeleteMapping("/index/delete/{index}")
public ResultBean deleteIndex(@PathVariable String index) {
    if (StringUtils.isBlank(index)) {
        return ResultBean.error("参数错误，索引为空！");
    }
    try {
        // 删除索引
        DeleteIndexRequest deleteIndexRequest = Requests.deleteIndexRequest(index);
        DeleteIndexResponse response = transportClient.admin().indices().delete(deleteIndexRequest).get();

        logger.info("删除索引结果:{}",response.isAcknowledged());
        return ResultBean.success("删除索引成功！");
    } catch (Exception e) {
        logger.error("删除索引失败！要删除的索引为{}，异常为：",index,e.getMessage(),e);
        return ResultBean.error("删除索引失败！");
    }
}
```

```JSON
DELETE /index/delete/book

{
    "status": true,
    "msg": "删除索引成功！",
    "data": null
}
```

### 2.3 判断索引是否存在

```java
IndicesExistsRequest inExistsRequest = new IndicesExistsRequest(index);
IndicesExistsResponse inExistsResponse = transportClient.admin().indices().exists(inExistsRequest).actionGet();

boolean hasExist = inExistsResponse.isExists();
```

### 2.4 清空索引内容

```java
// 查询所有记录
BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery().must(QueryBuilders.matchAllQuery());
// 删除查询结果
BulkByScrollResponse response = DeleteByQueryAction.INSTANCE.newRequestBuilder(transportClient)
         .filter(queryBuilder)
         .source(index)
         .get();
// 得到删除数
long deleteCount = response.getDeleted();
```

## 三、数据操作

### 3.1 插入、更新数据

- 若未指定ID或ID不存在，插入数据

- 若指定ID且ID存在，更新数据

```java
@PutMapping("/{index}/{type}/{id}")
public ResultBean addDocument(@PathVariable String index, @PathVariable String type, @PathVariable String id
        , @RequestBody Map map) {
    logger.info("接收到数据的参数。索引名：{}，类别：{}，id：{}，参数：{}", index, type, id, map);

    IndexResponse response = transportClient.prepareIndex(index, type, id)
            .setSource(map)
            .get();

   return response.status().getStatus() == 200 ?
   			ResultBean.success("插入/更新数据成功！") : ResultBean.error("插入/更新数据失败！");
}
```

```JSON
PUT /book/it/1
{
	"name": "Java从入门到放弃",
	"author": "高德纳",
	"price": "135",
	"publish_date": "2018-10-09"
}

{
    "status": true,
    "msg": "插入/更新数据成功！",
    "data": null
}
```

### 3.2 查看数据

```java
@GetMapping("/{index}/{type}/{id}")
public ResultBean getDocument(@PathVariable String index, @PathVariable String type, @PathVariable String id) {
    GetResponse response = transportClient.prepareGet(index, type, id).get();

    Map<String, Object> source = response.getSource();

    return ResultBean.success("查询成功！", source);
}
```

```JSON
GET /book/it/1

{
    "status": true,
    "msg": "查询成功！",
    "data": {
        "author": "高德纳",
        "price": "135",
        "name": "Java从入门到放弃",
        "publish_date": "2018-10-09"
    }
}
```

### 3.3 删除数据

```java
@DeleteMapping("/{index}/{type}/{id}")
public ResultBean deleteDocument(@PathVariable String index, @PathVariable String type, @PathVariable String id) {
    DeleteResponse response= transportClient.prepareDelete(index, type, id).get();

    return response.status().getStatus() == 200 ?
            ResultBean.success("删除数据成功！") : ResultBean.error("删除数据失败！");
}
```

```JSON
DELETE /book/it/1

{
    "status": true,
    "msg": "删除数据成功！",
    "data": null
}
```

### 3.4 搜索数据

Elasticsearch重头戏就是搜索，这里限于篇幅只能简单演示下，详细的搜索功能请参考官方API：https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/java-search.html

需求：查询书名中包含关键字的记录，且50 < 价格 <= 200，按照价格降序排序，从0开始取最多10条记录，进行关键字高亮处理。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181009164522410.png)

```java
@GetMapping("/{index}/{type}/search")
public ResultBean search(@PathVariable String index, @PathVariable String type, String name) {
    Map<String, Object> map = new HashMap<>();
    try {
        SearchResponse response = transportClient.prepareSearch(index)
                .setTypes(type)
                .setQuery(QueryBuilders.boolQuery()
                        .must(QueryBuilders.matchQuery( "name", name))
                        .must(QueryBuilders.rangeQuery("price").gt(50).lte(200))
                )
                .highlighter(new HighlightBuilder()
                        .field("name")
                        .preTags("<span style='color:red;'>")
                        .postTags("</span>")
                )
                .addSort("price", SortOrder.DESC)
                .setFrom(0)
                .setSize(10)
                .execute()
                .get();

        SearchHits hits = response.getHits();
        map.put("total", hits.getTotalHits());

        List<String> result = new ArrayList<>();
        List<String> highLight = new ArrayList<>();
        for(SearchHit hit:hits){
            result.add(hit.getSourceAsString());
            highLight.add(hit.getHighlightFields().toString());
        }

        map.put("result", result);
        map.put("highLight", highLight);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }

    return ResultBean.success("搜索完成！", map);
}
```

```json
GET /book/it/search?name=入门

{
    "status": true,
    "msg": "搜索完成！",
    "data": {
        "result": [
            "{\"name\":\"MySQL从入门到删库\",\"author\":\"滑稽脸\",\"price\":\"200\",\"publish_date\":\"2018-08-18\"}",
            "{\"name\":\"Java从入门到放弃\",\"author\":\"高德纳\",\"price\":\"100\",\"publish_date\":\"2018-10-09\"}"
        ],
        "total": 2,
        "highLight": [
            "{name=[name], fragments[[MySQL从<span style='color:red;'>入</span><span style='color:red;'>门</span>到删库]]}",
            "{name=[name], fragments[[Java从<span style='color:red;'>入</span><span style='color:red;'>门</span>到放弃]]}"
        ]
    }
}
```

说明一下：

（1）不是使用了IK分词吗，为什么查询“入门”，高亮代码为：`MySQL从<span style='color:red;'>入</span><span style='color:red;'>门</span>到删库`？

答：创建索引时制定的是`ik_max_word`模式，即最大匹配，更改为`ik_smart`即可。

（2）当使用完全匹配查询时，查不出结果？例如增加一个作者为"高德纳"的条件：

```java
.setTypes(type)
	.setQuery(QueryBuilders.boolQuery()
	        .must(QueryBuilders.matchQuery( "name", name))
	        .must(QueryBuilders.rangeQuery("price").gt(50).lte(200))
	        .must(QueryBuilders.termQuery("author,", author))
	)
```

这是因为Elasticsearch默认会对text类型进行分词，又因为使用了must（必须匹配） + term（完全匹配）条件，所以要想能搜出来应该修改为：

```java
.setTypes(type)
	.setQuery(QueryBuilders.boolQuery()
	        .must(QueryBuilders.matchQuery( "name", name))
	        .must(QueryBuilders.rangeQuery("price").gt(50).lte(200))
	        .must(QueryBuilders.termQuery("author,", "高"))
	        .must(QueryBuilders.termQuery("author,", "德"))
	        .must(QueryBuilders.termQuery("author,", "纳"))
	)
```

当然这样很蠢，所以应该创建索引时设置类型为keyword：

```json
"author":{
   "type":"keyword"
}
```

请不要设置成如下形式，在5.X版本下index只能设置为true/false，如果不需要索引将type设置为keyword即可：

```json
"author":{
   "type":"text",
   "index":"not_analyzed"
}
```
