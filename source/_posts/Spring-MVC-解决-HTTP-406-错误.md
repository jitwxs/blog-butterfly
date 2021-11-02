---
title: Spring MVC 解决 HTTP 406 错误
categories: 
  - Java Web
  - Spring Framework
abbrlink: 6da44730
date: 2018-04-17 00:45:50
---

在一般Java Web项目中，406错误，是比较少见的错误，导致原因一般是以下两种：

**原因一：** 缺少`jackJson`包

导入依赖即可：

```xml
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	<version>2.9.4</version>
</dependency>
```

**原因二：** 请求路径为`*.html`

**当SpringMVC的拦截请求为`*.html`时，不允许返回json格式数据！**

```xml
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.html</url-pattern>
</servlet-mapping>
```

解决办法，要么修改为其他的拦截请求，要么添加一个其他的拦截请求，以添加一个`*.action`为例：

```xml
...
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.html</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>*.action</url-pattern>
</servlet-mapping>
```
