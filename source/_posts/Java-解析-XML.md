---
title: Java 解析 XML
tags: XML
categories: Java Basic
toc_number: true
abbrlink: aa870a3e
date: 2017-12-11 17:00:07
---

## XML解析方式

**（1）DOM解析：**解析器把**整个**XML文件加载到内存，并生成一个`Document`对象。

优点：**元素与元素之前保持依赖管理**，可以对其进行CRUD操作。

缺点：当XML文件过大时，可能会出现**内存溢出**问题。

**（2）SAX解析：**一种速度更快、更有效的方法。它逐行扫描文档，一边扫描一边解析。基于**事件驱动**进行具体解析，每执行一行，都将触发对应的事件。

优点：**处理速度很快**，可以处理大文件。

缺点：**只能读，逐行后会释放资源**。

**（3）PULL解析：** Android内置的XML解析方式，类似于SAX。

## XML解析器

根据不同的解析方式提供了具体的实现，称为`解析器`。为了简化解析器的操作，各大公司提供了易于操作的`解析开发包`：

- JAXP: SUN公司提供支持DOM和SAX的开发包

- jsoup: 一种处理HTML特定解析开发包

- **dom4j**: 较为常用的解析开发包，Hibernate底层所使用的

- JDom: dom4j兄弟

## DOM解析原理

XML DOM将整个XML文档加载到内存中，生成一颗**DOM树**，并获得一个Document对象，通过Document对象就可以对DOM进行操作。若XML文件如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<web-app version="2.5">
    <servlet>
        <servlet-name>helloServlet</servlet-name>
        <servlet-class>jit.wxs.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>helloServlet</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```

经过DOM解析后，生成了一棵树：

![XML树](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171211161640833.png)

## dom4j

这里以dom4j为例，讲解代码实现。

（1）导入jar包

![导入jar包](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201712/20171211162043978.png)

（2）API

dom4j必须使用核心类`SaxReader`加载xml文档获得`Document`，通过Document对象获得文档的根元素，然后就可以操作了。

**SaxReader对象：**

- read(...) 加载执行xml文档

**Document对象：**

- getRootElement() 获得根元素

**Element对象：**

- elements(...) 获得指定名称的所有子元素，可以不指定名称

- element(...) 获得指定名称的第一个子元素，可以不指定名称

- getName() 获得当前元素的元素名

- attributeValue() 获得指定属性名的属性值

- elementText(...) 获得指定名称子元素的文本值

- getText() 获得当前元素的文本内容

（3）示例代码

```java
import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import java.io.File;
import java.util.List;

public class Main {

    public static void main(String[] args) throws DocumentException {
        // 1. 获得Document对象
        SAXReader saxReader = new SAXReader();
        Document doc = saxReader.read(new File("src/web.xml"));
        
        // 2. 获得根元素
        Element rootElement = doc.getRootElement();
        
        // 获取version属性值
        String version = rootElement.attributeValue("version");
        System.out.println("version: " + version);

        printElement(rootElement);
    }

    // 递归遍历DOM树
    private static void printElement(Element root) {
        // 3. 获得所有子元素
        List<Element> lists = root.elements();

        // 4. 遍历所有
        for (Element element: lists ) {
            String name = element.getName();
            System.out.println("进入结点：" + name);

            if(element.elements().size() == 0) {
                String text = element.getText();
                System.out.println(name + "内容为：" + text);
            } else {
                printElement(element);
            }
        }
    }
}
```

```
version: 2.5
进入结点：servlet
进入结点：servlet-name
servlet-name内容为：helloServlet
进入结点：servlet-class
servlet-class内容为：jit.wxs.HelloServlet
进入结点：servlet-mapping
进入结点：servlet-name
servlet-name内容为：helloServlet
进入结点：url-pattern
url-pattern内容为：/hello
```
