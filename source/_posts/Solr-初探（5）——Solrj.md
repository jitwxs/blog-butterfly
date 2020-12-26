---
title: Solr 初探（5）——Solrj
categories:
  - 搜索引擎
  - Solr
abbrlink: 46ffa9b2
date: 2018-04-10 20:27:39
copyright_author: Jitwxs
---

## 一、导入依赖

导入Solr源码包`dist`文件夹下的`solr-solrj-6.6.2.jar`以及`solrj-lib`文件夹下的所有包到项目中。除此之外，还要加上`log4j`包和`junit`测试包。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410194650625.png)

## 二、添加/更新数据

Solrj的使用十分简单，下面是一个添加数据的例子：

```java
@Test
public void testAdd() throws Exception {
	// 指定url
	String baseURL = "http://localhost:8080/solr/core1";

	// 1.创建Solr单机版客户端
	HttpSolrClient solrClient = new HttpSolrClient(baseURL);

	// 2.创建一个Document
	SolrInputDocument doc = new SolrInputDocument();
	doc.addField("id", "10086");
	doc.addField("author", "jitwxs");
	// 3.添加Document，1000ms自动提交
	solrClient.add(doc, 1000);
}
```

（1）BaseURL就是Solr的首页地址和核：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018041019503032.png)

（2）创建一个`SolrClient`，其中`HttpSolrClient`是单机版，`CloudSolrClient`是集群版。

（3）可以设置提交时间自动提交，也可以手动提交`solrClient.commit();`

（4）这种初始化`SolrClient`的方式在`Solr 7.0`中被废弃，改为：

```java
HttpSolrClient solrClient = new HttpSolrClient.Builder(BASE_URL)
				.withConnectionTimeout(10000)
				.withSocketTimeout(60000)
				.build();
```

查看Solr后台，发现数据已经被添加了：
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410195408180.png)

## 三、删除数据

删除十分简单，不再具体演示，给出相关API：

```java
// 根据id删除单个
solrClient.deleteById("10086");

// 根据id删除多个
List<String> ids = new ArrayList<>();
ids.add("1");
solrClient.deleteById(ids, 1000);

// 根据条件删
solrClient.deleteByQuery("*:*");

// 根据条件删，指定核
solrClient.deleteByQuery("core1", "id:10086", 1000);
```

## 四、查询数据

业务需求：

- 默认域为item_title

- 关键词：手机

- 价格：5k以上

- 价格降序

- 分页：第1页，每页10条

- 高亮：红色

```java
@Test
public void testQuery() throws Exception {
	// 指定Solr服务器url
	String baseURL = "http://localhost:8080/solr/core1";

	// 创建Solr单机版客户端
	SolrClient HttpSolrClient = new HttpSolrClient(baseURL);
	
	/*
	 * 查询需求
	 * 设置默认域为item_title
	 * 关键词：手机
	 * 价格：5k以上
	 * 排序：价格降序
	 * 分页：第1页，每页10条
	 * 高亮：红色
	 */
	SolrQuery solrQuery = new SolrQuery();
	// 设置默认域
	solrQuery.set("df", "item_title");
	// 设置关键词
	solrQuery.set("q", "手机");
	// 设置价格区间
	solrQuery.set("fq", "item_price:[5000 TO *]");
	// 设置排序
	solrQuery.addSort("item_price", ORDER.desc);
	// 设置分页
	solrQuery.setStart(1);
	solrQuery.setRows(5);
	// 开启高亮
	solrQuery.setHighlight(true);
	// 设置高亮字段
	solrQuery.addHighlightField("item_title");
	// 高亮前缀
	solrQuery.setHighlightSimplePre("<span style='color:red'>");
	// 高亮后缀
	solrQuery.setHighlightSimplePost("</span>");
	
	// 执行查询
	QueryResponse response = solrClient.query(solrQuery);
	// 获取文档结果集
	SolrDocumentList docs = response.getResults();
	// 获取高亮结果集
	Map<String, Map<String, List<String>>> highlighting = response.getHighlighting();
	
	// 总条数
	long numFound = docs.getNumFound();
	System.out.println("共查到：" + numFound + "条数据");
	
	for(SolrDocument doc : docs) {
		System.out.println("id:" + doc.get("id"));
		System.out.println("sell_point:" +doc.get("item_sell_point"));
		System.out.println("price:" +doc.get("item_price"));
		
		// 标题高亮
		Map<String, List<String>> map = highlighting.get(doc.get("id"));
		List<String> list = map.get("item_title");
		for(String s : list) {
			System.out.println("title:" + s);
		}
		
		System.out.println("==================");
	}
	
}
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410202635748.png)
