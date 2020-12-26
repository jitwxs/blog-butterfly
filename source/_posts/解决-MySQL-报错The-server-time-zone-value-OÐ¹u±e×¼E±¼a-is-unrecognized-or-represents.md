---
title: >-
  解决 MySQL 报错The server time zone value 'ÖÐ¹ú±ê×¼Ê±¼ä' is unrecognized or
  represents ....
categories: 
  - 数据库
  - MySQL
abbrlink: 8a950d9f
date: 2018-11-15 16:47:15
copyright_author: Jitwxs
---

## 问题描述

今天在使用SpringBoot 2.1 + MyBatis时，报了一个很奇怪的错误，如下所示：

```java
2018-11-15 15:22:42.424 ERROR 14132 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Exception during pool initialization.

java.sql.SQLException: The server time zone value 'ÖÐ¹ú±ê×¼Ê±¼ä' is unrecognized or represents more than one time zone. You must configure either the server or JDBC driver (via the serverTimezone configuration property) to use a more specifc time zone value if you want to utilize time zone support.
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:129) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:97) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:89) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:63) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:73) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:76) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:835) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:455) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:240) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.mysql.cj.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:207) ~[mysql-connector-java-8.0.13.jar:8.0.13]
	at com.zaxxer.hikari.util.DriverDataSource.getConnection(DriverDataSource.java:136) ~[HikariCP-3.2.0.jar:na]
	at com.zaxxer.hikari.pool.PoolBase.newConnection(PoolBase.java:369) ~[HikariCP-3.2.0.jar:na]
	at com.zaxxer.hikari.pool.PoolBase.newPoolEntry(PoolBase.java:198) ~[HikariCP-3.2.0.jar:na]
...
```

看了下包依赖，SpringBoot 2.1 默认使用的是`mysql-connector-java 8`的依赖，而不是之前使用的 5.x。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181115152954963.jpg)

出错的原因是从`mysql-connector-java 6`开始，连接时需要指定时区`serverTimezone`。

## 解决办法

解决办法大致有以下三种：

（1）将包替换为 5.X 的版本，当然这等于没说。。

（2）修改Url，显示的指定时区，对于标准时区是`GMT`，对于我们来说，使用北京时间，即GMT+8，在尾部添加`serverTimezone=GMT%2B8`：

```
spring.datasource.url=jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=GMT%2B8
```

（3）修改MySQL的 my.ini 配置文件，在[mysqld]下添加：

```
default-time-zone='+08:00'
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181115153837884.jpg)

记得重启下mysql服务，在命令行中执行命令`SHOW VARIABLES LIKE '%time_zone%'`，得到下图结果就OK了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181115154134709.jpg)

## 其他

`mysql-connector-java 5.X` 之后数据库驱动从

```
com.mysql.jdbc.Driver
```

更改到了

```
com.mysql.cj.jdbc.Driver
```

别忘了更替噢。
