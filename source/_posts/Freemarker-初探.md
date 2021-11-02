---
title: Freemarker 初探
tags: Freemarker
categories: 
  - Java Web
  - Template Engine
abbrlink: e843d977
date: 2018-04-16 11:07:35
---

## 一、什么是Freemarker

### 1.1 介绍

`FreeMarker` 是一个用Java语言编写的模板引擎，它**基于模板来生成文本输出**。`FreeMarker` **与Web容器无关**，即在 Web 运行时，它并不知道 Servlet 或 HTTP。它**不仅可以用作表现层的实现技术，而且还可以用于生成 XML，JSP 或 Java 等**。

主要用 `Freemarker` **做静态页面或是页面展示**。

![Freemarker](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416095513273.png)

### 1.2 入门程序

首先导入依赖：

```xml
<dependency>
	<groupId>org.freemarker</groupId>
	<artifactId>freemarker</artifactId>
	<version>2.3.27-incubating</version>
</dependency>
```

在我的D盘下，存在文件夹 `flt` ，用于存放模板文件；存在文件夹 `flt_out`，用于存放输出文件。

在 flt 文件夹下新建文件 `test.ftl`，内容为：

```
${test}
```

![test](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416100108850.png)

是不是有点像 `EL表达式`，在 EL表达式中使用 `model.addAttr..()` 就可以替换掉JSP 中的 EL表达式了，Freemarker 和它类似，其步骤大致如下：

 1. 创建`Configuration`对象，指定编码集和模板文件夹
 2. 创建`Template`对象，指定模板文件
 3. 准备数据
 4. 创建`Writer`对象，指定输出文件
 5. 生成模板

```java
@Test
public void firstDemo() throws Exception {
    //1、创建Configuration对象，指定编码集和模板文件夹
    Configuration configuration = new Configuration(Configuration.getVersion());
    configuration.setDefaultEncoding("utf-8");
    configuration.setDirectoryForTemplateLoading(new File("D:/ftl"));
    //2、创建Template对象，指定模板文件
    Template template = configuration.getTemplate("test.ftl");
    //3、准备数据，Map或POJO类型，推荐Map
    Map<String,Object> data = new HashMap<>();
    data.put("test", "第一个Freemarker例子");
    //4、创建Writer对象，指定输出文件
    Writer writer = new FileWriter("D:/ftl_out/test.txt");
    //5、生成模板
    template.process(data,writer);
}
```

运行程序，在 ftl_out 文件夹中生成了 test.txt 文件，查看文件内容：

![test结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416101139775.png)

可以看到，成功将`${test}`替换为输入的数据。

## 二、基本语法

### 2.1 取 Map 中 Key

上面的入门程序其实就是这种。

如果模板中内容为：

```
${test}
```

直接往 Map 中存 key 即可：

```java
Map<String,Object> data = new HashMap<>();
data.put("test", "第一个Freemarker例子");
```

### 2.2 取 POJO 中属性

其实取 POJO 和 EL表达式还是一样，如果存在模板内容为：

```
学生信息：
姓名：${student.name}
年龄：${student.age}
```

有一个 Student 对象，内容为：

```java
public class Student {
	private String name;
	private Integer age;
	// 省略getter/setter...
}
```

则创建一个 Map，放入一个 Student 对象即可：

```java
Map<String,Object> data = new HashMap<>();
data.put("student", student对象);
```

### 2.3 取集合中元素

如果存在一个学生列表，模板内容：

```
<#list studentList as student>
学生信息：
姓名：${student.name}
年龄：${student.age}
</#list>
```

传递一个 list 即可：

```java
List<Student> list = new ArrayList<>();
list.add(new Student("jitwxs", 20));
list.add(new Student("zhangsan", 25));
list.add(new Student("lisi", 12));

Map<String,Object> data = new HashMap<>();
data.put("studentList", list);
```

![集合元素](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416103159941.png)

### 2.4 取循环中下标

如果要取循环中的坐标，修改模板如下：

```
<#list studentList as student>
学生信息，当前index：${student_index}
姓名：${student.name}
年龄：${student.age}
</#list>
```

![循环下标](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416103359147.png)

### 2.5 判断

修改模板如下：

```
<#list studentList as student>
<#if student_index % 2 == 0>
学生信息（偶数）：
<#else>
学生信息（单数）：
</#if>
姓名：${student.name}
年龄：${student.age}
</#list>
```

![判断](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416103645519.png)

### 2.6 日期处理

传入模板一个Date()对象：

```java
Map<String,Object> data = new HashMap<>();
data.put("date", new Date());
```

模板内容：

```
当前日期：${date?date}
当前时间：${date?time}
当前日期和时间：${date?datetime}
自定义日期格式：${date?string("yyyyMMdd-HH:mm:ss")}
```

![日期处理](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416104211750.png)

### 2.7 NULL处理

放入一个 null 值：

```java
Map<String,Object> data = new HashMap<>();
data.put("value",null);
```

模板内容如下：

```
方法1：使用默认值
${value!"value值为null"}

方法2：使用if语句
<#if value??>
${value}
<#else>
value值为null
</#if>
```

![NULL处理](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180416105101636.png)

### 2.8 include

一个模板还可以包含另一个模板，使用 include 标签即可：

```
学生信息：
姓名：${student.name}
年龄：${student.age}

<#include "test.ftl"/>
```

## 三、与Spring整合

在配置文件中将 `FreeMarkerConfigurer` 注入 Bean：

```xml
<!-- FreeMarker -->
<bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
    <property name="templateLoaderPath" value="D:/ftl" />
    <property name="defaultEncoding" value="UTF-8" />
</bean>
```

首先用 `@Autowired` 将 `freeMarkerConfig` 注入进来

```java
@Autowired
private FreeMarkerConfig freeMarkerConfig;
```

然后调用 `getConfiguration()` 得到 `Configuration` 对象，剩下的和之前就一样了。

```java
Configuration configuration = freeMarkerConfig.getConfiguration();
```
