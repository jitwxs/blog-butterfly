---
title: SpringBoot 集成 Spring Security（9）——解决 UserNotFoundException 不抛出问题
categories:
  - 安全框架
  - Spring Security
tags: 异常处理
abbrlink: d9ec6e24
date: 2019-07-09 17:05:14
copyright_author: Jitwxs
---

## 一、前言

《SpringBoot 集成 Spring Security》系列文章，原本只是我自己学习后写的笔记，没想到受到大家的欢迎，能够对大家带来帮助，让我感到十分高兴。但说起来我也只是初学者，这一系列文章中可能也存在错误，本文是为了解决 UserNotFoundException 这个异常无法抛出而写出。

这个问题大致是这样的，我们知道 Spring Security 的验证处理是由某个 Provider 处理的，在 Provider 中通过对应的 UserDetailsService 的 `loadUserByUsername()` 来决定如何加载数据库中的用户信息。

以 [《SpringBoot集成Spring Security（3）——异常处理》](/eb7552c8.html) 代码为例，采用默认的用户名密码登陆方式，我们在 `CustomUserDetailsService` 类中，有这么一行代码：

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    ...
    // 判断用户是否存在
    if(user == null) {
        throw new UsernameNotFoundException("用户名不存在");
    }

    ...
}
```

但是实际运行后你会发现，当用户不存在时，只会抛出 `BadCredentialsException`，而不是 `UsernameNotFoundException`，百度下后发现这个问题的人不在少数。在本文中，我将介绍为什么会无法抛出 `UsernameNotFoundException`，以及如何解决这个问题。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190708215558621.png)

## 二、导致的原因是什么？

首先说明，出现这种情况只有在你**使用默认的用户名密码登陆方式，且没有自定义 Provider** 的情况下，才会发生！如果你有自定义 Provider，仍然出现这个问题，说明代码写的有问题，这一点在后面我会说。

>由提供的[源码地址](https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/springboot_security)中，第三章和第四章两篇文章是一个项目。分别是 `springboot_security03` 和 `springboot_security03_filter`。由于前者是自定义 Provider 实现，因此理论上不会出现这个问题，所以我们以后者代码为例。

运行 `springboot_security03_filter` 项目，发现当用户不存在时，的确抛出的是 BadCredentialsException。我们给抛出异常那行代码加断点，进行调试。

```java
@Service("userDetailsService")
public class CustomUserDetailsService implements UserDetailsService {
    ...
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        ...
        if (user == null) {
            throw new UsernameNotFoundException("用户名不存在");  // TODO 该行断点
        }
        ...
    }
}
 ```

给 `org.springframework.security.authentication.ProviderManager` 第 174 行加上断点，该行是 provider 调用 UserDetailsService 位置。

运行后，首先进入该断点行中，根据断点信息已知 providers 一共只有1个，当前的 providers 是 DaoAuthenticationProvider。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709011813797.png)

DaoAuthenticationProvider 是 Spring Security 默认的用户名密码登陆的处理 provider，符合预期。跳到下一断点处，此时进入 UserDetailsService，由于用户不存在，跳转到异常抛出处。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709012207760.png)

下面采用单步调试，如图所示，抛出异常后在 `DaoAuthenticationProvider` 的 `retrieveUser()` 方法中被捕获。随后执行了 mitigateAgainstTimingAttack() 方法，虽然我没看出这方法有啥实际作用。但这不重要，下一行将该异常又抛出去了。

被抛到了 DaoAuthenticationProvider 的父类 AbstractUserDetailsAuthenticationProvider 的 `authenticate()` 方法中。如图所示，在 catch 到 UsernameNotFoundException 后，有个关键的 `hideUserNotFoundExceptions` 变量。当 hideUserNotFoundExceptions 为true 时，在这个地方被重新包装成了 BadCredentialsException 抛出去。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709013252203.png)

检查后发现 true 为其默认值，这就是导致 UserNotFoundException 无法抛出的原因。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709013839485.png)

## 三、如何解决？

解决的办法也很简单，我们只要想办法把这个变量默认值改掉就好了。因此我们需要手动注入 DaoAuthenticationProvider，在注入时候把值改了。`WebSecurityConfig` 类改动如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709014423310.png)

首先自己注入下 DaoAuthenticationProvider：

```java
@Bean
public DaoAuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setHideUserNotFoundExceptions(false);
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
}
```

通过调用 `setHideUserNotFoundExceptions` 改变其默认值，这样就 OK 啦！同时这里需要指定加密方式和 UserDetailsService，因此原本默认的全局配置 config() 方法就可以不要了，删除方法：

```java
//    @Override
//    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
//        auth.userDetailsService(userDetailsService).passwordEncoder(new PasswordEncoder() {
//            @Override
//            public String encode(CharSequence charSequence) {
//                return charSequence.toString();
//            }
//
//            @Override
//            public boolean matches(CharSequence charSequence, String s) {
//                return s.equals(charSequence.toString());
//            }
//        });
//    }
```

这里将加密方式也注入 bean，方便调用：

```java
 @Bean
 public PasswordEncoder passwordEncoder() {
     return new PasswordEncoder() {
         @Override
         public String encode(CharSequence charSequence) {
             return charSequence.toString();
         }

         @Override
         public boolean matches(CharSequence charSequence, String s) {
             return s.equals(charSequence.toString());
         }
     };
 }
 ```

重新运行程序，已经没毛病了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709015035758.png)

## 四、勘误

这一小节的题目为“勘误”，勘的就是我前面几篇文章，以及博客示例程序中的错误。错误的发现是当我运行 `springboot_security03` 项目时，依然出现了这个问题。而那个项目采用的是自定义 provider，如果你已经明白这个错误的缘由后，就会知道这个错误的根本原因是 DaoAuthenticationProvider 的父类 AbstractUserDetailsAuthenticationProvider 中存在变量来控制是否将异常进行包装。

那么我自己定义 provide，跟这 DaoAuthenticationProvider 屁关系都没有，怎么也会出现这个错误呢，显然代码是有毛病的。

因此我运行 `springboot_security03` 后，也在 ProviderManager Line 174 行断点调试后，发现 providers 竟然是两个，DaoAuthenticationProvider 也在列。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709020041511.png)

此时我还没发现问题，我继续单步调试，自己定义的 provider 中的确把 UsernameNotFoundException 跑出去了，我正高兴了，突然发现它竟然又跳入了下一次循环，此时 provider 是 DaoAuthenticationProvider，且它通过了`provider.supports()` 校验，开始进入正式执行流程了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709020514562.png)

我暗道不好，根据前文我们知道 DaoAuthenticationProvider 默认最后把 UsernameNotFoundException 包装成了 BadCredentialsException。当 providers 循环遍历结束后，取了 lastException，并把它抛出去。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709021005931.png)

由于是先执行 CustomAuthenticationProvider 后执行 DaoAuthenticationProvider，故最终自定义的 provider 的 UsernameNotFoundException，被 DaoAuthenticationProvider 的 BadCredentialsException 给覆盖了。

发现问题了，下面开始解决问题，大致有以下三种方案。

**Plan A：** 我们在 provider 循环中让 CustomAuthenticationProvider 和 DaoAuthenticationProvider 掉个顺序不就好了？

咳咳，不得不说真是一个十分糟主意，虽然我没咋研究 providers 的顺序是咋生成的，暂且认为是字典序吧，我难道自定义 providrs 还得考虑首字母命名顺序吗？ PASS!

**Plan B：** 根据[《SpringBoot集成Spring Security（7）——认证流程》](/a28c0db7.html)，我们知道如果我们要自定义一种登陆方式，那么 xxxProvider、xxxUserDetailsService、xxxToken，这三个应该是一体的。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709022031849.png)

provider 决定了调用哪个 userDetailsService，support() 方法决定了这个 provider 支持的 token。但是我在这个项目中偷了懒，我只自定义了 provider 和 userDetailsService，却没有定义专属的 token，而是使用 UsernamePasswordAuthenticationToken。

那么 UsernamePasswordAuthenticationToken 将会同时被我的 provider 以及 DaoAuthenticationProvider 所支持。那么 PlanB 就是定义一个专属的 token，跟[《SpringBoot 集成 Spring Security（8）——短信验证码登录》](/d208d079.html)一样。

这的确是一个好主意，而且我认为这也是 Spring Security 在非 DEMO 项目中正确的使用方式，provider + userDetailsService + token 三者捆绑。

然而这毕竟只是一个 demo，而且还是第三章的 demo，考虑到入手难度，不想引入太多的类。PASS！

**Plan 3：** 换个思路，这个 DaoAuthenticationProvider 是咋加进去，如果它不在 providers 循环中，不就没问题了？

修改 WebSecurityConfig 类如下图，去除了 `auth .userDetailsService(userDetailsService).passwordEncoder(passwordEncoder())` 配置，这会导致将 DaoAuthenticationProvider 加入到 providers 集合中。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190709025121463.png)

>我参考 https://blog.csdn.net/wzl19870309/article/details/70314085 这篇文章，找到了解决方案。
