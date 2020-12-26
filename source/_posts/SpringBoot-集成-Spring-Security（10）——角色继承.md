---
title: SpringBoot 集成 Spring Security（10）——角色继承
categories:
  - 安全框架
  - Spring Security
abbrlink: '7886e906'
date: 2019-09-03 00:11:30
related_repos:
  - name: springboot_security10
    url: https://github.com/jitwxs/blog-sample/blob/master/SpringBoot/springboot_security/springboot_security10
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

在本节中，补充下**角色继承**的知识点。角色继承其实是一个十分常见的需求，因为一般系统中角色权限呈金字塔型，高层用户拥有底层用户的权限。

例如存在以下角色：普通用户、VIP 用户、SVIP 用户、星悦会员，那么对应的权限可以是“星悦会员 > SVIP 用户 > VIP 用户 > 普通用户”。那么如何在 Spring Security 中实现这样的功能呢？

## 引言

为了简便起见，我直接使用[《SpringBoot 集成 Spring Security（1）——入门程序》](/5f5715e6.html) 的代码。

在该章中，我们存在两个角色，`ROLE_ADMIN` 和 `ROLE_USER`，并且经过我们的实验，`/admin` 接口只有 ROLE_ADMIN 有权限，`/user` 接口只有 ROLE_USER 有权限。

```java
@RequestMapping("/admin")
@ResponseBody
@PreAuthorize("hasRole('ROLE_ADMIN')")
public String printAdmin() {
    return "如果你看见这句话，说明你有ROLE_ADMIN角色";
}

@RequestMapping("/user")
@ResponseBody
@PreAuthorize("hasRole('ROLE_USER')")
public String printUser() {
    return "如果你看见这句话，说明你有ROLE_USER角色";
}
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180330153402126.png)

但是如果我想让 ROLE_ADMIN 用户继承 ROLE_USER 用户的所有权限，该如何做呢？

## RoleHierarchy 

这里就需要引入 `RoleHierarch ` 了，我们只需要自定义一个 RoleHierarchy，并将其注入容器即可。修改 `WebSecurityConfig`，在其中注入 RoleHierarchy：

```java
@Bean
public RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
    String hierarchy = "ROLE_ADMIN > ROLE_USER";
    roleHierarchy.setHierarchy(hierarchy);
    return roleHierarchy;
}
```

`roleHierarchy.setHierarchy()` 指定了角色的继承关系，参数就是一个字符串，比大小即可，是不是非常简单？

让我们使用 ROLE_ADMIN 账号登陆，发现原本无法访问的 `/user` 接口也可以访问了：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201909/20190902233036282.png)

## 源码

角色关系的实现也比较简单，本质就是将字符串使用正则切分，并将角色关系存放进一个 Map 中，map 的 key 是大的角色，value 是一个 Set，存放所有比它小的角色。然后交由后续处理，有兴趣的可以继续阅读源码。

![buildRolesReachableInOneStepMap()]()

## SpringBoot 2.1?

在本系列开篇时，说过本系列文章环境为`SpringBoot 2.0`，具体的版本号是 2.0.0.RELEASE，有小伙伴可能会说，你这个不是最新的啊，最新的稳定版都到 2.1.X.RELEASE 了。

这是因为该系列第一篇发布时最新的 RELEASE  版本就是 2.0，为了版本稳定本系列均保持为 2.0，有的小伙伴拿着代码但是在 2.1 的环境下运行，可能就会出现不能运行的情况。

请小伙伴们发现无法正常运行情况，优先检查版本是否为 2.0，因为 IDEA 构建时默认就会选择最新的 RELEASE  版本。**别看 2.0 和 2.1 只差了一点点，有些改动可能就会坑死你**。如果确定不是版本问题后，请优先下载源码自行排查问题，如仍无法解决再留言提问。

好了，闲话扯完，在写这篇文章时，我也踩了这个坑，我按照网上的教程后发现无法正常使用，检查版本后发现是 2.1，回退到 2.0 后就可以正常使用。

看了源码后发现，这是因为在 SpringBoot 2.1 中，Spring Security 版本从 5.0 升级到了 5.1，而 RoleHierarchyImpl 这个类关键的解析方法 `buildRolesReachableInOneStepMap()` 实现发生了变化。

![buildRolesReachableInOneStepMap()]()

可以看到在 Spring Security 5.1 中它先用 `readLine()` 分割为多行，然后通过 `split()` 分割大于号左右两边角色。和 Spring Security 5.0 的改动点就是在多个比较之间使用换行符号分割替代空格分隔。

因此如果代码在 SpringBoot 2.0 中这样写的话：

```java
@Bean
public RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
    String hierarchy = "ROLE_ADMIN > ROLE_USER ROLE_USER > ROLE_TOURISTS";
    roleHierarchy.setHierarchy(hierarchy);
    return roleHierarchy;
}
```

在 SpringBoot 2.1 中，就应该改写为：

```java
@Bean
public RoleHierarchy roleHierarchy() {
    String separator = System.lineSeparator();
    
    RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
    String hierarchy = "ROLE_ADMIN > ROLE_USER " + separator + " ROLE_USER > ROLE_TOURISTS";
    roleHierarchy.setHierarchy(hierarchy);
    return roleHierarchy;
}
```

另外换行符大家都知道在不同系统中表示不一样，例如 Windows 中为 `\r\n`，Mac 为 `\r`，Linux 为 `\n`，因此以上代码我是用的 `java.lang` 包的 System 类中封装的方法，不用判断当前操作系统。
