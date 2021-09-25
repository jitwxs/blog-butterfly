---
title: Solr 初探（4）——后台数据导入及 Document 的使用
categories:
  - 搜索引擎
  - Solr
abbrlink: 7aa79ad1
date: 2018-04-10 19:14:28
---

进入Solr后台页面，选择一个核，点击 `Documents`，进入 `Document` 管理标签：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410164320462.png)

## 一、添加 Document

在[《Solr初探（2）——域管理》](/ea6efdc6.html)中我们已经说过了，id 是一个 Document 必须要包含的 field，让我们新建一个 `Document`，类型为 `JSON`：

```json
{
"id" : 1,
"name" : "jitwxs"
}
```

在查询页中点击 `Execute Query` 进行查询，就能看见我们添加的这个 `Doucment` 了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410164729758.png)

## 二、更新 Document

修改一个 `Document` 十分简单，只需要指定 `id`，Solr 就会先删除 id 为指定 id 的 `Document`，然后将我们输入的 `Document` 保存起来，达到更新的目的（内部实现是**先删后添**）。

当我们输入：

```json
{
"id" : 1,
"author":"jitwxs",
"content" : "哇哈哈哈"
}
```

重新查询：

```json
{
  "responseHeader":{
    "status":0,
    "QTime":0,
    "params":{
      "q":"*:*",
      "indent":"on",
      "wt":"json",
      "_":"1523349939964"}},
  "response":{"numFound":1,"start":0,"docs":[
      {
        "id":"1",
        "author":"jitwxs",
        "author_s":"jitwxs",
        "content":["哇哈哈哈"],
        "_version_":1597348563073892352}]
  }}
```

## 三、删除 Document

删除一个 `Document` 比较特殊，使用 JSON 无法实现，我们将 `Document` 类型修改为 `XML`，将 id 为 1 的 Document 删除掉：

```xml
<delete>
    <id>1</id>
</delete>
```

重新查询：

```json
{
  "responseHeader":{
    "status":0,
    "QTime":1,
    "params":{
      "q":"*:*",
      "indent":"on",
      "wt":"json",
      "_":"1523349939964"}},
  "response":{"numFound":0,"start":0,"docs":[]
  }
```

除了上面哪种方法，还可以指定删除条件：

```xml
<delete>
	<query>id:1</query>
</delete>
```

因为可以始用正则表达式，那么举一反三，删除所有 Document，就是：

```xml
<delete>
	<query>*:* </query>
</delete>
```

## 四、导入数据

>注：数据库文件使用宜立方商城的数据库，[点击这里](https://github.com/jitwxs/e3mall/blob/master/sql/e3mall.sql)下载 sql 文件。

**Step1：** 导入依赖包

在 `core1` 的目录下新建 `lib` 文件夹（如果有，无需建立），从 Solr 源码包的 dist 文件夹中导入两个 `solr-dataimporthandler` 包，以及一个 `mysql` 驱动包。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410175217215.png)

**Step2：** 创建导入数据的 Handler

编辑 `core1/conf` 目录下的 `solrconfig.xml` 文件，在文件末尾添加上对 `requestHander` 的配置：

```xml
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
	<lst name="defaults">
		<str name="config">data-config.xml</str>
	</lst>
</requestHandler>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410182810703.png)

**Step3：** 配置 data-config.xml

我们刚刚在 `requestHander` 中指定了我们数据导入的配置文件，因此我们在 `solrconfig.xml` 的同级目录下，即 `core1/conf` 目录下新建 `data-config.xml` 文件：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410183100909.png)

编辑文件内容：

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<dataConfig>
	<!-- 数据库信息 -->
	<dataSource type="JdbcDataSource" 
		driver="com.mysql.jdbc.Driver" 
		url="jdbc:mysql://localhost:3306/e3mall" 
		user="root" password="root"/>
	<document>
		<!-- document实体 -->
		<entity name="item" query="SELECT id,title,sell_point,price FROM tb_item">
			<!-- 数据库字段映射solr字段 -->
			<field column="id" name="id"/>
			<field column="title" name="item_title"/>
			<field column="sell_point" name="item_sell_point"/>
			<field column="price" name="item_price"/>
		</entity>
	</document>
</dataConfig>
```

配置文件总共两个部分，第一个部分是数据库连接信息，第二个部分是在`document`中建立了一个实体，参数一个是name，一个是sql查询语句，其内部对查询结果的**每一项映射Solr的`field`**。

**Step4：** 配置field

在上一步中，我们将查询结果的每一项映射到了 Solr 的 `filed` 的上，但是除了 `id` 是存在的以外，其他都不存在，因此我们需要手动添加。

编辑 `managed-schema` 文件，在末尾新增相关的 field，如果你理解了[《Solr初探（2）——域管理》](/ea6efdc6,html)，这里的配置就很简单了。

解释：

（1）创建了 `item_title`、`item_sell_point`、`item_price` 三个域，其中前两个类型为 IK 分词器的类型，第三个为 int 类型。

（2）创建了 `item_keywords` 域，将 `item_title`、`item_sell_point` 拷贝到 `item_keywords` 中，便于搜索。（因为需求是从商品标题或商品卖点中查找）

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410183707457.png)

**Step5：** 重启服务

重启服务，选择 `core1` 的 `Dataimport` 标签页，此时右边就已经有内容了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410184041506.png)

解释：

（1）`/dataimport`

第二步 `requestHandler` 中起的名字。

（2）`clean`

每次导入前，清空 Document。

（3）`Entity`

第三步 `data-config.xml` 中配置的 `entity`。

当我们点击 Execute 按钮后，过一会右边会提示索引创建成功，如下所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410184054353.png)

点击 `Query` 标签页，查询后可以看到数据已经被导入了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410184106342.png)

## 五、查询 Document

之前一直是用 `*.*` 查询所有，下面详细介绍下查询：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410191304255.png)
