---
title: Solr 初探（2）——域管理
categories:
  - 搜索引擎
  - Solr
abbrlink: ea6efdc6
date: 2018-04-10 15:31:06
---

在上一节中，我们已经成功搭建了 Solr 后台，并且在后台中新建了一个`核（core）`，本节将介绍Solr配置域。

我们在后台系统中选择 `core1`，点击 `Documents`，在里面添加一个 `Document（文档）`，内容如下：

```json
{"id":"1","name":"jitwxs"}
```

点击 Submit 按钮执行成功：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410145146922.png)

然后选择 `Query`，查询条件设为 `*.*`，即查询所有，就可以看见我们刚刚插入的 `Doucment`：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018041014515683.png)

在[《Lucene 初探——基于 Lucene 6.6.2》](/44bf5506.html)这篇文章中，我们已经知道了一个 `Document` 中可以有多个`域（field）`，这里的 `id` 和 `name` 就是这个文档的域。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180304215920616.png)

我们打开 core1 的 `conf` 文件夹，其中有两个重要的配置文件需要我们掌握，一个是 `managed-schema`，一个是 `solrconfig.xml`，其中是 `managed-schema` 就是 Solr 对域的配置文件。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410145804812.png)

## 一、field 和 fieldType

打开 `managed-schema` 文件，首先介绍下 `field` 标签，这里是 Solr 帮我们预设好的域，如图示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410150124206.png)

| 属性名 | 含义 |
|:-------------|:-------------|
| name | 域的名字 |
| type | 域的类型 |
| indexed | 是否索引 |
| stored | 是否存储 |
| required | 是否必须 |
| multiValued | 是否允许多值（多个数值，数组一样） |

域的类型 Solr 也为我们定义好了，部分如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410150532177.png)

我们发现，之前我们添加的域 `filed` 和域 `name` 都是 Solr 定义过的，且 `id` 是 `require` 的，因此我们后面每次添加 `Document`，**都必须包含 id 域**。

值得注意的是，**域必须得在该配置文件中配置过后，才能够使用，否则无法使用！！！**

## 二、dynamicField

如果想添加一个域叫 `abcdefg`，那么需要在配置文件中添加一行：

```xml
<field name="abcdefg" ... />
```

但是每次修改配置文件都需要重启 Solr 服务，有没有更方便的方法呢？其实有的，Solr贴心的帮我们想好了自定义 Filed 问题，其中有一个`dynamicField`标签，列举其中一部分如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410151132809.png)

我们发现域名是`正则表达式`的形式，因此只要我们的域匹配了正则表达式，就无需自己定义，比如将 `abcdefg` 修改为 `abcdefg_s` 就匹配了 `*_s` 的域，这样就可以不用自己定义了。

## 三、copyField

假设我们有一堆文章，我们**想要搜索作者包含jitwxs或者描述中包含jitwxs的文章**，但是作者信息存在 `author` 域中，描述信息存在 `description`域中。难道我们要在两个域中分别查一次吗，当然不是，这就牵扯到了 `copyField`。

如图所示，配置文件中定义了一堆 `copyField`，其中就有 `author` 和 `description`，我们发现这两个域的 `dest` 属性均为 `text`。

当我们修改 `author` 域或者 `description` 域的时候，因为配置了 `copyField`，Solr 会**自动将内容拷贝**到 `dest` 的目标域，即 `text` 域。当我们查询的时候，就不会去查询 `author` 域或者 `description` 域，而是直接查询 `text` 域。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180410152402449.png)
