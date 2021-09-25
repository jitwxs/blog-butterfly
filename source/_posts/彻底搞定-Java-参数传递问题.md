---
title: 彻底搞定 Java 参数传递问题
categories: Java
abbrlink: 800a108d
date: 2019-07-02 23:51:34
icons: [fas fa-fire red]
---

## 一、引言

在开始正文前照例扯扯闲话，说说这篇文章的来源把。今天同事在处理一个 BUG 时产生了疑问，代码类似这样：

```java
public static void main(String[] args) {
    User user = null;
    func(user);
    String name = user.getName();
}

public static void func(User user) {
    user = getUser2();
    ...
}
```

当程序运行时，`user.getName()` 执行抛出 NPE，这是一个典型的关于 Java 是值传递还是引用传递的问题。

特别是有 C 语言的同学，在对 Java 基础掌握不是很牢的情况下，可能就会想当然的认为传递的是一个引用（指针），方法内部的属性改变会影响方法外。

当同事一开始让我看得时候，我也被绕了，这就说明自己基础掌握不牢固啊。故写下这篇文章，将这类问题一劳永逸的解决掉。

## 二、值传递还是引用传递？

在正式开始之前，我们必须得明确 Java 中形参的传递到底时值传递还是引用传递？

**Java 是值传递，不论传递的是基本数据类型还是引用数据类型，在 Java 中不存在引用传递！**

## 三、不同数据类型的传递策略

### 3.1 基本数据类型

Java 中的基本数据类型，即 byte、short、int、long、float、double、char、boolean。函数调用时**传递的是它们的值**，因此对形参的修改，不会影响实参。

这一点不论是啥语言，都是这样的，因此这是明确的。

### 3.2 引用数据类型

Java 中的引用数据类型，也就是我们说的对象，函数调用时**传递的是该对象的地址，而不是该对象的引用**。

假设存在 User 类：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class User {
    private String name;
}
```

对于如下程序，考虑它的输出：

```java
public static void main(String[] args) {
    User user = new User("zhangsan");
    func(user);
    System.out.println(user.getName());
}

private static void func(User user) {
    user = new User("lisi");
    user.setName("wangwu");
}
```

下面画图理解下，在 main() 函数中，创建了对象 User，我们假设它的地址为 0x1，那么变量 user 所指向的就是地址为 0x1 的 User。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190702231524840.png)

因为 Java 对于引用类型，值传递的是对象的地址。因此在进入 func() 函数后，形参 user 仍然指向地址 0x1。

当执行 `user = new User("lisi")` 这条语句后，新创建了一个地址为 0x2 的 User 对象，并让 user 指向了这个新的对象，那么它和原本的地址为 0x1 的对象的连接就中断了。因此下一行修改 name 值为“wangwu”修改的就是 0x2 ，与 0x1 无关。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190702231725963.png)

在方法执行结束后，由于该方法返回值为 void，因此执行结束后，main() 函数的 user 仍然是指向 0x1 的。func() 函数内部的改动相当于是对局部变量的修改，因此程序输出为“zhangsan”。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190702231524840.png)

明白了之后，再考虑下将 func() 函数中的 `user = new User("lisi")` 语句删除掉，程序的输出将会变为“wangwu”。这是因为在 func() 函数中 user 一直都是指向 0x1 的，因此在返回 main() 函数后，0x1 地址上的值已经被修改了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190702232714604.png)

最后，让我们回到引言中提到的代码，那里只是将 0x1 改成了 null 而已，相信你已经明白为什么会导致 NPE 了。

### 3.3 包装类型和 String

按照 3.2 节的结论，你会发现将对象换成包装类型或 String 类型，就不适用了。

```java
public static void main(String[] args) {
    String name = "zhangsan";
    func(name);
    System.out.println(name);
}

private static void func(String name) {
    name = "lisi";
}
```

如上代码所示，String 属于引用类型，按照 3.2 节结论，传递的是它的地址，那么为什么程序输出结果为 “zhangsan” 呢？

这是因为 String 类，比较特殊，它是不可变类（final），它的底层实现是 `final char value[]`，对于 final 自然是无法修改的，同理包装类也是不可变的。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190702233601669.png)

将 String 改成 StringBuilder 或者 StringBuffer 返回结果就会和 3.2 节的结论一致。因为其内部的 char[] 数组并不是 final 类型，只是最终输出的时候才调用 toString() 方法，将数组转变为 String 而已。

```java
public static void main(String[] args) {
    StringBuilder sb = new StringBuilder("hello");
    func(sb);
    System.out.println(sb); // Output: Hello World
}

private static void func(StringBuilder sb) {
    sb.append(" world");
}
```

## 四、总结

总结下整篇文章的结论：**Java 采用值传递，不存在引用传递。**

1. 对于基本数据类型，传递的为数值，形参内部的改变不影响实参。
2. 对于引用数据类型，传递的是对象的地址，对形参的修改影响的是该地址上的对象。
3. 对于包装类型和 String 类型，比较特殊。虽然它们属于引用数据类型，但由于其底层实现为 final，因此它们是不能被修改的。

---

最后来到题加深下印象：

```java
class MyData {
    int a = 10;
}
public class MethodArgumentTest {
    public static void main(String[] args) {
        int i = 1;
        String str = "hello";
        Integer num = 200;
        int[] arr = {1, 2, 3, 4, 5};
        MyData my = new MyData();
        MyData my1 = new MyData();

        change(i, str, num, arr, my, my1);

        System.out.println("i = " + i);
        System.out.println("str = " + str);
        System.out.println("num = " + num);
        System.out.println("arr = " + Arrays.toString(arr));
        System.out.println("my.a = " + my.a);
        System.out.println("my1.a = " + my1.a);
    }

    private static void change(int j, String s, Integer n, int[] a, MyData m, MyData m1) {
        j += 1;
        s += "world";
        n += 1;
        a[0] += 1;
        m.a += 1;
        m1 = new MyData();
        m1.a = 2;
    }
}
```

输出：

```console
i = 1
str = hello
num = 200
arr = [2, 2, 3, 4, 5]
my.a = 11
my1.a = 10
```
