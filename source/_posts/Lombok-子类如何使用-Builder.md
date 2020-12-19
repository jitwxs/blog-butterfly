---
title: Lombok 子类如何使用 @Builder
categories: 开发工具
tags: Lombok
icons:
  - fas fa-fire red
abbrlink: 54621f54
date: 2020-08-26 22:15:18
copyright_author: Jitwxs
---

## 一、前言

业务开发中，子类父类还算是经常用到，Lombok 的 `@builder` 提供的链式调用帮助我们更轻松的创建对象。但是实验后却发现子类的 `@Builder` 是不会包含父类的属性。

假设存在父类 `A`：

```java
@Data
@Builder
public class A {
    private String aName;

    private String aAge;
}
```

存在子类 `B`：

```java
@Builder
@Data
@EqualsAndHashCode(callSuper = true)
public class B extends A {
    private String bName;

    private String bAge;
}
```

使用 `builder` 进行初始化时，类 A 可以正常创建，类 B 仅可以初始化自己的属性，父类属性无法初始化。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200829214450.png)

## 二、解决：构造方法

查阅网络后，一种解决方法是利用构造方法：

1. 父类生成全参构造方法
2. 子类手动声明全参构造方法
3. 将子类 `@builder` 注解移动全参构造方法上，并设置 `builderMethodName`

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class A {
    private String aName;

    private String aAge;
}
```

```java
@Data
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class B extends A {
    private String bName;

    private String bAge;

    @Builder(builderMethodName = "childBuilder")
    public B(String aName, String aAge, String bName, String bAge) {
        super(aName, aAge);
        this.bName = bName;
        this.bAge = bAge;
    }
}
```

修改 Main 方法如下：

```java
public class BuilderMain {
    public static void main(String[] args) {
        A xxx = A.builder()
                .aName("xxx")
                .aAge("111")
                .build();
        B yyy = B.childBuilder()
                .aName("xxx")
                .aAge("111")
                .bName("yyy")
                .bAge("222")
                .build();

        System.out.println(xxx);
        System.out.println(yyy);
    }
}
```

代码运行后，能得到正确结果：

```java
A(aName=xxx, aAge=111)
B(super=A(aName=xxx, aAge=111), bName=yyy, bAge=222)
```

但是这种方式弊端也很明显：

1. 子类调用父类的全参构造，当父类参数数量、顺序调整时，子类也需要同步调整。
2. 如果父类参数过多，构造方法十分不优雅。

## 三、解决：SuperBuilder

Lombok 自 `v1.18.2` 开始，为了解决这个问题，引入了 `@SuperBuilder` 注解，使用该注解，就可以很容易解决这个问题。

修改代码如下：

```java
@Data
@SuperBuilder
public class A {
    private String aName;

    private String aAge;
}
```

```java
@SuperBuilder
@Data
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class B extends A {
    private String bName;

    private String bAge;
}
```

Main 方法保持不变，代码运行后，能得到正确结果：

```java
A(aName=xxx, aAge=111)
B(super=A(aName=xxx, aAge=111), bName=yyy, bAge=222)
```

另外自 `v1.18.4` 也给 SuperBuilder 引入了`toBuilder` 参数，可以很方便的进行**浅拷贝**对象，效率虽然比手动 builder 慢一点，但也算是挺快的。

```java
@Data
@SuperBuilder(toBuilder = true)
public class A {
    private String aName;

    private String aAge;
}
```

给 B 加入属性 C，测试对象拷贝：

```java
@SuperBuilder(toBuilder = true)
@Data
@EqualsAndHashCode(callSuper = true)
@ToString(callSuper = true)
public class B extends A {
    private String bName;

    private String bAge;

    private C c;
}
```

```java
@AllArgsConstructor
public class C {
    private String name;
}
```

输出结果如下：

```java
public class BuilderMain {
    public static void main(String[] args) {
        B b = B.builder()
                .aName("xxx")
                .aAge("111")
                .bName("yyy")
                .bAge("222")
                .c(new C("zhangsan"))
                .build();

        System.out.println(b);
        System.out.println(b.toBuilder().build());
    }
}
```

```java
B(super=A(aName=xxx, aAge=111), bName=yyy, bAge=222, c=com.github.jitwxs.demo.builder.C@52cc8049)
B(super=A(aName=xxx, aAge=111), bName=yyy, bAge=222, c=com.github.jitwxs.demo.builder.C@52cc8049)
```

## 四、彩蛋

由于 `@SuperBuilder` 刚引入不久，所以还是有一些 BUG 的，比如当你的 SpringBoot 版本为 `2.2.3.RELEASE` 时，或者你的 Lombok 版本低于 `v1.18.12` 时，使用上文的例子，你就会发现竟然无法通过编译。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20200829221256.png)

实际的原因是不支持用 `B` 来命名类，简直是吐出一口老血，好在升级到 `v1.18.12` 版本后就修复了这个问题，可能这就是给不规范命名的人埋的坑吧，哈哈。

> [ISSUE: @SuperBuilder does work on classes named B or C](https://github.com/rzwitserloot/lombok/issues/2297)
