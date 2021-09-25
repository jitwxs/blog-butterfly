---
title: SpringBoot 集成 Spring Security（2）——自动登录
categories:
  - 安全框架
  - Spring Security
abbrlink: 8e248ac0
date: 2018-05-09 10:22:57
related_repos:
  - name: springboot_security02
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-security/springboot_security02
    rel: nofollow noopener noreferrer
    target: _blank
---

在上一章：[《SpringBoot集成Spring Security（1）——入门程序》](/5f5715e6.html)中，我们实现了入门程序，本篇为该程序加上自动登录的功能。

## 一、修改 login.html

在登陆页添加自动登录的选项，注意自动登录字段的 name 属性必须是 `remember-me` ：

```html login.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登陆</title>
</head>
<body>
<h1>登陆</h1>
<form method="post" action="/login">
    <div>
        用户名：<input type="text" name="username">
    </div>
    <div>
        密码：<input type="password" name="password">
    </div>
    <div>
        <label><input type="checkbox" name="remember-me"/>自动登录</label>
        <button type="submit">立即登陆</button>
    </div>
</form>
</body>
</html>
```

## 二、两种实现方式

### 2.1 Cookie 存储

这种方式十分简单，只要在 WebSecurityConfig 中的 configure() 方法添加一个 `rememberMe()` 即可,如下所示：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            // 如果有允许匿名的url，填在下面
//                .antMatchers().permitAll()
            .anyRequest().authenticated()
            .and()
            // 设置登陆页
            .formLogin().loginPage("/login")
            // 设置登陆成功页
            .defaultSuccessUrl("/").permitAll()
            // 自定义登陆用户名和密码参数，默认为username和password
//                .usernameParameter("username")
//                .passwordParameter("password")
            .and()
            .logout().permitAll()
            // 自动登录
            .and().rememberMe();

    // 关闭CSRF跨域
    http.csrf().disable();
}
```

当我们登陆时勾选自动登录时，会自动在 Cookie 中保存一个名为 `remember-me` 的cookie，默认有效期为2周，其值是一个加密字符串：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509100451811.png)

### 2.2 数据库存储

使用 Cookie 存储虽然很方便，但是大家都知道 Cookie 毕竟是保存在客户端的，而且 Cookie 的值还与用户名、密码这些敏感数据相关，虽然加密了，但是将敏感信息存在客户端，毕竟不太安全。

Spring security 还提供了另一种相对更安全的实现机制：**在客户端的 Cookie 中，仅保存一个无意义的加密串（与用户名、密码等敏感数据无关），然后在数据库中保存该加密串-用户信息的对应关系，自动登录时，用 Cookie 中的加密串，到数据库中验证，如果通过，自动登录才算通过。**

#### 2.2.1 基本原理

当浏览器发起表单登录请求时，当通过 `UsernamePasswordAuthenticationFilter` 认证成功后，会经过 `RememberMeService`，在其中有个 `TokenRepository`，它会生成一个 token，首先将 token 写入到浏览器的 Cookie 中，然后将 token、认证成功的用户名写入到数据库中。

当浏览器下次请求时，会经过 `RememberMeAuthenticationFilter`，它会读取 Cookie 中的 token，交给 RememberMeService 从数据库中查询记录。如果存在记录，会读取用户名并去调用 `UserDetailsService`，获取用户信息，并将用户信息放入Spring Security 中，实现自动登陆。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202143630639.png)

RememberMeAuthenticationFilter 在整个过滤器链中是比较靠后的位置，也就是说在传统登录方式都无法登录的情况下才会使用自动登陆。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202144420871.png)

#### 2.2.2 代码实现

首先需要创建一张表来存储 token 信息：

```sql
CREATE TABLE `persistent_logins` (
  `username` varchar(64) NOT NULL,
  `series` varchar(64) NOT NULL,
  `token` varchar(64) NOT NULL,
  `last_used` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`series`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

在 WebSecurityConfig 中注入 `dataSource` ，创建一个 `PersistentTokenRepository` 的Bean：

```java
@Autowired
private DataSource dataSource;

 @Bean
 public PersistentTokenRepository persistentTokenRepository(){
     JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
     tokenRepository.setDataSource(dataSource);
     // 如果token表不存在，使用下面语句可以初始化该表；若存在，请注释掉这条语句，否则会报错。
//        tokenRepository.setCreateTableOnStartup(true);
     return tokenRepository;
 }
```

在 `config()` 中按如下所示配置自动登陆：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            // 如果有允许匿名的url，填在下面
//                .antMatchers().permitAll()
            .anyRequest().authenticated()
            .and()
            // 设置登陆页
            .formLogin().loginPage("/login")
            // 设置登陆成功页
            .defaultSuccessUrl("/").permitAll()
            // 自定义登陆用户名和密码参数，默认为username和password
//                .usernameParameter("username")
//                .passwordParameter("password")
            .and()
            .logout().permitAll()
            // 自动登录
            .and().rememberMe()
                .tokenRepository(persistentTokenRepository())
                // 有效时间：单位s
                .tokenValiditySeconds(60)
                .userDetailsService(userDetailsService);

    // 关闭CSRF跨域
    http.csrf().disable();
}
```

## 三、运行程序

勾选自动登录后，Cookie 和数据库中均存储了 token 信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509102031410.png)
