---
title: Java Web 中接入支付宝支付
tags: 三方支付
categories:
  - Java Web
  - SpringBoot
typora-root-url: ..
abbrlink: ea57cb90
date: 2018-06-05 00:29:21
related_repos:
  - name: springboot_alipay
    url: https://github.com/jitwxs/blog-sample/blob/master/SpringBoot/springboot_alipay
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

>注：因为没有企业账号，所以本篇文章为**沙箱环境**中，但是其逻辑和真实环境是一样的。

接入支付宝的步骤大致如下：

 1. `申请一个沙箱环境`

 2. `生成签名，并在沙箱环境中设置好签名`
 
 3. `下载官方的SDK结合API学习后开发`

 申请沙箱环境的网址是：[沙箱环境](https://openhome.alipay.com/platform/appDaily.htm)

签名工具及它的使用方法的链接是：[签名工具](https://openhome.alipay.com/platform/appDaily.htm?tab=tool)

官方的API链接是：[API](https://docs.open.alipay.com/270/105898)

官方的Demo是： [Demo For Java](http://p.tb.cn/rmsportal_6680_alipay.trade.page.pay-JAVA-UTF-8.zip)

我自己写好了一个Demo，注释丰富，可以帮助大家学习，比官方的略微复杂一些，地址在文章开头。支付宝支付本质上就是使用它的API，根据上面提供的资料和我的Demo相信应该能够帮助大家学会了。

如果有疑问，欢迎评论留言。

### 特别提示

当你开始运行Demo程序时，可不要用你自己的支付宝进行测试哦，想想也不能用真实的支付宝扣钱啊。

官方提供了一个商家账号，一个买家账号，链接在这：[沙箱账号](https://openhome.alipay.com/platform/appDaily.htm?tab=account)

这个账号也不能在支付宝登录，而要使用沙箱钱包，暂时只支持Android版，链接在这：[沙箱钱包](https://sandbox.alipaydev.com/user/downloadApp.htm)

因为异步通知方法必须要公网能够访问，因此我推荐下我使用的软件，[NatApp](https://natapp.cn/)。能够实现内网穿透，开发用免费版就可以了。

### 如何使用例子

**Step1：** 找到源码sql文件夹下的sql文件，导入到数据库中。对应修改程序配置文件中的数据库连接信息

**Step2：** 按照上面的资料申请好沙箱，生成好签名，修改程序配置文件中支付的配置信息

![修改配置文件](/images/posts/20180605220537873.png)

**这里有一个很大的坑！**

{% note warning %}

如果你设置了内网穿透，例如把`localhost:8080`映射到`xx.example.com`上，那么运行项目请使用`xx.example.com`，而不是继续用`localhost:8080`

{% endnote %}

###  截图

![支付宝在线支付Demo1](/images/posts/20180605000729234.png)

![支付宝在线支付Demo2](/images/posts/20180605000750289.png)

![支付宝在线支付Demo3](/images/posts/20180605000810355.png)

![支付宝在线支付Demo4](/images/posts/20180605000954825.png)
