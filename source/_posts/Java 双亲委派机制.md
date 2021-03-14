---
title: Java 双亲委派机制
copyright_author: Jitwxs
tags: 双亲委派
categories: Java
abbrlink: 9f878221
date: 2021-03-14 11:26:34
---

## 一、ClassLoader

在介绍双亲委派机制之前，必须先要了解类加载器 `ClassLoader`，当 `.java` 文件经过编译生成 `.class` 文件后，需要通过 ClassLoader 将其加载入 JVM 中。

Java 中包括以下四类 ClassLoader：

- Bootstrap ClassLoader 启动类加载器【主要负责加载核心类库（`java,lang.*`）等】
- Extension ClassLoader 扩展类加载器【主要负责加载 `jre/lib/ext` 目录下的扩展类库】
- Application ClassLoader 应用程序类加载器【主要负责加载当前应用程序的类】
- Custom ClassLoader 自定义类加载器

### 1.1 获取 ClassLoader

通过 `java.lang.Class#getClassLoader` 方法就能够知道加载某个类的具体 ClassLoader，如下面代码所示：

```java
import sun.net.spi.nameservice.dns.DNSNameService;

public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println(Object.class.getClassLoader());

        System.out.println(DNSNameService.class.getClassLoader());

        System.out.println(Foo.class.getClassLoader());
    }

    static class Foo {}
}
```

输出结果如下：

```java
null
sun.misc.Launcher$ExtClassLoader@7ef20235
sun.misc.Launcher$AppClassLoader@18b4aac2
```

- `Object.class` 是 `java.lang` 类，它的 ClassLoader 是 Bootstrap ClassLoader，但是由于 Bootstrap ClassLoader 采用 C++ 实现，因此其的输出结果为 null。
- `DNSNameService.class` 是 JRE 扩展包类，它的 ClassLoader 是 Extension ClassLoader。
- `Foo.class` 是我自定义的类，它的 ClassLoader 是 Application ClassLoader。

### 1.2 篡改 ClassLoader

上一节提到 `Object.class` 是 `java.lang` 包提供的类。如果我自己也定义一个 `java.lang.Object` 类，能够编译通过吗？

```java
package java.lang;

public class Object {
    public static void main(String[] args) {
        System.out.println("Custom Object.class");
    }
}
```

运行如上代码后会报错，原因就是因为 Java 的双亲委托机制。

```java
错误: 在类 java.lang.Object 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

## 二、双亲委派机制

### 2.1 概念

当 JVM 接收到需要加载类的请求时

（1）首先**自下向上**判断该类是否已经加载：

- 首先判断 Custom ClassLoader 是否已经加载该类，如果没有加载，委派给自己的 parent classLoader --> Extension ClassLoader。

- 判断 Application ClassLoader 是否已经加载该类，如果没有加载，委派给自己的 parent classLoader --> Extension ClassLoader。
- 判断 Extension ClassLoader 是否已经加载该类，如果没有加载，委派给自己的 parent classLoader --> Bootstrap ClassLoader。
- 判断 Bootstrap ClassLoader 是否已经加载该类。

（2）如果所有 ClassLoader 都未加载，则**自上向下**加载该类：

- 首先判断 Bootstrap ClassLoader 能否加载该类，如果不能加载，委派给自己的 child classLoader -> Extension ClassLoader。
- 判断 Extension ClassLoader 能否加载该类，如果不能加载，委派给自己的 child classLoader -> Extension Application 。
- 判断 Application ClassLoader 能否加载该类，如果不能加载，委派给自己的 child classLoader -> Custom Application 。
- 判断 Custom Application 能否加载该类，如果不能加载，抛出 ClassNotFondException。

下面两张流程图形象的描述了这种关系：

![双亲委派机制-1](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts//202103/20210314104028.png)

![双亲委派机制-2](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts//202103/20210314105732.png)

这也就解释了为什么 1.2 节的代码运行会报错的原因。当运行 main 方法后，加载 `java.lang.Object.class` 的不是 Application ClassLoader 而是 Bootstrap ClassLoader，即实际被加载的是源码包的 Object.class，在那个类中是不存在 main 方法的。

### 2.2 作用

**（1）为什么要自下向上判断类是否已经加载？**

确保类只被加载一次，如果 parent ClassLoader 已经加载，那么 child ClassLoader 就无需加载。

**（2）为什么要自上向下加载类？**

确保 Java 核心类库不被 Application ClassLoader 或 Custom ClassLoader 所加载，保证安全。

### 2.3 源码实现

#### 2.3.1 加载 Parent

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        ClassLoader classLoader = ClassLoader.getSystemClassLoader();

        while (true) {
            System.out.println("ClassLoader: " + classLoader);

            classLoader = classLoader.getParent();

            if(classLoader == null) {
                System.out.println("ClassLoader: Bootstrap ClassLoader");
                break;
            }
        }
    }
}
```

- `ClassLoader.getSystemClassLoader()` 获取默认委托的 ClassLoader，即最底层的。
- 通过 `getParent()` 获取上层 ClassLoader，当为 null 时表示当前已经是 Bootstrap ClassLoader。

运行结果如下：

```java
ClassLoader: sun.misc.Launcher$AppClassLoader@18b4aac2
ClassLoader: sun.misc.Launcher$ExtClassLoader@2b193f2d
ClassLoader: Bootstrap ClassLoader
```

#### 2.3.2 ClassLoader#loadClass

![ClassLoader#loadClass](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts//202103/20210314112545.png)
