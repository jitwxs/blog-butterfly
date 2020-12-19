---
title: Java 设计模式——观察者模式
categories:
  - Java
  - 设计模式
abbrlink: 57c88403
date: 2019-05-27 00:12:09
copyright_author: Jitwxs
---

## 一、前言

好久没更新设计模式系列了，这周闲来无事，就水一把，介绍个简单的——`观察者模式`。

所谓观察者模式，本质是就是**发布与订阅**，在日常生活中发布/订阅的例子有很多，比如大家微信里面的公众号，你可以订阅微信公众号，公众号发布文章后，微信会将文章推送给你。。。

## 二、发布 / 订阅

在上面提到的公众号的例子，就是一个观察者模式。

你作为一名普通用户就是**观察者**，你可以关注或者取关公众号，当公众号发布消息时，你会收到消息推送。

公众号运营者就是消息的发布者，即**被观察者**，当用户关注你时，你需要将用户添加到你的推送列表中；同理，当用户取关时，你要把他从推送列表中移除（当然真实情况这是微信平台处理的）。当你发布消息时，应该告知所有关注了你的用户。

### 2.1 耦合实现

先定义公众号类，内部 `users` 集合保存所有关注该公众号的用户，`addUser()` 和 `removeUser()` 方法分别表示关注和取关，`pushMessage()` 方法表示发布新消息。

```java 微信公众号类
public class WeChatPublicAccount {
    private List<WeChatUser> users;

    public WeChatPublicAccount() {
        this.users = new ArrayList<>();
    }

    public void addUser(WeChatUser user) {
        users.add(user);
    }

    public void removeUser(WeChatUser user) {
        users.remove(user);
    }

    public void pushMessage(String message) {
        System.out.println("公众号发布新消息：" + message);
        for (WeChatUser user : users) {
            System.out.println("推送消息给用户：" + user.getUsername());
            user.receivePush(message);
        }
    }
}
```

接着实现普通用户类，主要的方法就是 `receivePush()` 方法，用于接收公众号的消息推送。

```java 微信用户类
public class WeChatUser {
    private String username;

    public WeChatUser(String username) {
        this.username = username;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public void receivePush(String message) {
        System.out.println("【" + this.username + "】接收新消息：" + message);
    }
}
```

编写一个测试类运行下，创建三个用户关注 jitwxs 的公众号，然后公众号发布消息。随后用户 lisi 取关，当再次发布消息时，lisi 已经无法收到消息推送。

```java
public class Main {
    public static void main(String[] args) {
        WeChatUser zhangsan = new WeChatUser("zhangsan");
        WeChatUser lisi = new WeChatUser("lisi");
        WeChatUser wangwu = new WeChatUser("wangwu");

        WeChatPublicAccount jitwxs = new WeChatPublicAccount();
        // 关注公众号：jitwxs
        jitwxs.addUser(zhangsan);
        jitwxs.addUser(lisi);
        jitwxs.addUser(wangwu);

        // 发布消息
        jitwxs.pushMessage("Hello");
        System.out.println("=================================");

        // lisi 取消关注
        jitwxs.removeUser(lisi);
        // 发布消息
        jitwxs.pushMessage("World");
    }
}
```

运行结果如下所示：

```console
公众号发布新消息：Hello
推送消息给用户：zhangsan
【zhangsan】接收新消息：Hello
推送消息给用户：lisi
【lisi】接收新消息：Hello
推送消息给用户：wangwu
【wangwu】接收新消息：Hello
=================================
公众号发布新消息：World
推送消息给用户：zhangsan
【zhangsan】接收新消息：World
推送消息给用户：wangwu
【wangwu】接收新消息：World
```

### 2.2 解耦实现

Java 设计模式的一大设计原则就是要符合**开闭原则**，以上的实现明显是不符合要求的，公众号类和普通用户耦合严重。比如微信有一天又出了一个啥啥号，这个号推送给关注者的是别的啥啥东西，但是其原理还是发布订阅的原理，而代码却不能复用，需要重新实现。

所谓解耦，主要就是将相似的地方抽成抽象的东西（接口或抽象类），不同的业务场景去实现或继承这个抽象的类。

观察者模式中对于公众号和普通用户有着专门的名词，对于公众号，称为 **Subject**，我习惯称之为**被观察者**；对于普通用户，称为 **Observer**，我习惯称之为**观察者**。

> 我这里的观察者和被观察者的抽象均采用接口实现，使用抽象类进行抽象也是可以的。

首先对被观察者实现抽象，被观察者主要有以下几个行为：

- 添加观察者
- 删除观察者
- 通知观察者

```java 被观察者抽象类
public interface Subject {
    void registerObserver(Observer o);
    void removeObserver(Observer o);
    void notifyObserver(String message);
}
```

对于观察者，主要的行为就是提供一个 `update()` 方法，当被观察者执行通知方法时调用。

```java 观察者抽象类
public interface Observer {
    void update(String message);
}
```

有了这两个抽象类以后，再重新定义下公众号类，此时就需要实现 Subject 类：

```java
public class WeChatPublicAccount implements Subject {
    private List<Observer> list;

    public WeChatPublicAccount() {
        this.list = new ArrayList<>();
    }

    @Override
    public void registerObserver(Observer o) {
        list.add(o);
    }

    @Override
    public void removeObserver(Observer o) {
        list.remove(o);
    }

    @Override
    public void notifyObserver(String message) {
        System.out.println("公众号发布新消息：" + message);
        for(Observer observer : list) {
            observer.update(message);
        }
    }
}
```

重新定义普通用户类，需要实现 Observer 类：

```java
public class WeChatUser implements Observer {
    private String username;

    public WeChatUser(String username) {
        this.username = username;
    }

    @Override
    public void update(String message) {
        System.out.println("【" + this.username + "】接收新消息：" + message);
    }
}
```

编写测试类运行：

```java
public class Main {
    public static void main(String[] args) {
        Observer zhangsan = new WeChatUser("zhangsan");
        Observer lisi = new WeChatUser("lisi");
        Observer wangwu = new WeChatUser("wangwu");

        Subject jitwxs = new WeChatPublicAccount();
        // 注册观察者
        jitwxs.registerObserver(zhangsan);
        jitwxs.registerObserver(lisi);
        jitwxs.registerObserver(wangwu);

        // 发布消息
        jitwxs.notifyObserver("Hello");
        System.out.println("=================================");

        // 移除观察者：lisi
        jitwxs.removeObserver(lisi);
        // 发布消息
        jitwxs.notifyObserver("World");
    }
}
```

运行结果如下所示：

```console
公众号发布新消息：Hello
【zhangsan】接收新消息：Hello
【lisi】接收新消息：Hello
【wangwu】接收新消息：Hello
=================================
公众号发布新消息：World
【zhangsan】接收新消息：World
【wangwu】接收新消息：World
```

### 2.3 书面描述

> 以上都是我用通俗的话进行的表述，总得有点书面话撑撑文章，哈哈。下面给出关于观察者模式的书面介绍，主要参考书籍：《设计模式之禅》。

观察者模式（Observer Pattern）也叫做发布订阅模式（Publish/Subscrible），它定义对象之间一种一对多的依赖关系，使得每当一个对象改变状态，则所有依赖于它的对象都会得到通知并自动更新。

![观察者模式](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20190527000410.png)

观察者模式通用类图如上所示，一般至少包含四个角色。

- **Subject 被观察者**

  定义被观察者必须要实现的职责，它必须能够动态地增加、取消观察者。它的主要职责是管理观察者并通知观察者。

- **Observer 观察者**

  观察者接收到消息后，进行 update 操作，对接收到的消息进行处理。

- **ConcreteSubject 具体被观察者**

  定义业务逻辑，同时定义对哪些事件进行通知。

- **ConcreteObserver 具体观察者**

  接收到消息后，实现具体的处理逻辑。

## 三、Java 中的观察者模式

Java 已经原生为我们提供了观察者模式中 Subjet 和 Observer 的抽象：

- Subject 被观察者：**java.util.Observable**（抽象类）
- Observer 观察者：**java.util.Observer**（接口）

因此如果没有特殊的业务需求，直接使用这两个已经提供好的类即可。
