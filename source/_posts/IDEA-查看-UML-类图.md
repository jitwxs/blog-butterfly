---
title: IDEA 查看 UML 类图
categories:
  - 开发工具
  - IDEA
tags: UML
abbrlink: 13a03d86
date: 2019-07-17 00:00:19
copyright_author: Jitwxs
---

## 一、基础使用

查看类图功能特别是对于刚接手一个新系统时，对于系统的熟悉起到辅助作用，本文介绍下 IDEA 的 Diagrams 功能，希望对您能有所帮助。

### 1.1 查看类图

IDEA 的 Diagrams 功能使用起来非常简单，在你想要生成类图的类中右击选择 Diagrams 即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716225007941.png)

如上图所示，该功能有两个子选项，这两个选项的区别就是前者将类图渲染在一个新标签页中，而后者则是在当前页以浮窗的形式展示。除了在类中右键点击以外，在目录树中右键点击也是同样的效果，然后你就会得到一幅类图。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716225504423.png)

### 1.2 类图含义

在上图中，类名前面的符号含义，比较明显。C 表示该类是一个**普通类**，I 则表示该类是一个**接口**，@ 表示该类是一个**注解**。符号后面的绿色的解锁状态表明该类是 public 修饰。

上图中几种线的含义和 UML 规范基本相同，**三角箭头的实线**表示二者的**泛化（Generalization）关系**，在 Java 中的体现就是 extend，箭头所指的类就是父类。

```java
public class WebSecurityConfigurerAdapter {
}

public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
}
```

**三角箭头的虚线**表示二者的**实现（Realization）关系**，在 Java 中的体现就是 implements，箭头所指的类就是父类。

```java
public interface WebSecurityConfigurer {
}

public class WebSecurityConfigurerAdapter implements WebSecurityConfigurer {
}
```

**普通箭头的虚线**表示二者的**依赖（Dependency）关系**，在 Java 中往往体现在对变量、方法等的调用关系。

```java
public interface WebSecurityConfigurer<T extends SecurityBuilder<Filter>> extends
        SecurityConfigurer<Filter, T> {
}
```

上图中最后一种**单纯的虚线**，应该不属于 UML 的规范，在这里的含义是**注解**的依赖。

## 二、深入使用

### 2.1 移除类

有时候我们需要移除类图中不相关的类，达到突出重点的目的。实现起来十分简单，选中想要删除的类，按下 DELETE 键即可。

如果你发现 DELETE 键无效，比如我再 MAC 上就不能用，此时选中该类后右击 DELETE 也是一样的效果。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716233026438.png)

### 2.2 展示类图的附加属性

单纯的类图可能不能满足我们的需要，比如我们想要显示每个类的方法要怎么办呢。有两种方法，第一种是点类图标签页左上角的一排按钮，第二种是右击选择 Show Categories 即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716233228749.png)

- Fields：属性
- Constructors：构造方法
- Methods：方法
- Properties：属性的 getter、setter 方法
- 内部类

同时我们也可以对这些附加属性筛选显示级别，右击后选择 Change Visability Level 即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/2019071623380141.png)

###  2.3 加入其它类图

当我们还需要查看其他类和当前类图中的类是否有关系的时候，可以在当前类图中加入其他类。右键选择 Add Class to Diagram... ，然后在弹出框中输入类名即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716234537483.png)

### 2.4 查看具体代码

想要查看某个类的代码，在类图中选中某个类后，右击选择 Jump to Source 即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716234831123.png)

而如果要跳转到某个附加属性上，则需要先双击附加属性框，然后用鼠标选择具体项后，右击选择 Jump to Source 即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716235046599.png)

### 2.5 类图布局

如果你不小心将类图弄乱了也没关系，IDEA 内置了多种布局，你可以任意选择。右键选择 Layout 即可，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190717001008819.png)

## 三、Structure

最后在补充下 IDEA 的 Structure 功能，也是在看代码中一个十分常用的功能。最通用的方法，就是点击 IDEA 左下角的 Structure 标签页即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716235425617.png)

更常用的则是以悬浮窗的形式弹出，方便我们随时跳转。这个快捷键如果我没记错的话，Windwos 平台是 `Ctrl + O`，Mac 平台是 `Command + F12`。

如果快捷键无效的话，可以在 IDEA 设置中查看下 file structure 的键位设置。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190716235813229.png)
