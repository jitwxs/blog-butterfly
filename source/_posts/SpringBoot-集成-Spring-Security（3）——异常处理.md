---
title: SpringBoot 集成 Spring Security（3）——异常处理
categories:
  - 安全框架
  - Spring Security
tags: 异常处理
abbrlink: eb7552c8
date: 2018-05-09 10:48:06
related_repos:
  - name: springboot_security03
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-security/springboot_security03
  - name: springboot_security03_filter
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-security/springboot_security03_filter
---

不知道你有没有注意到，当我们登陆失败时候，Spring security 帮我们跳转到了 `/login?error` Url，奇怪的是不管是控制台还是网页上都没有打印错误信息。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509103952703.png)

这是因为首先 `/login?error` 是 Spring security 默认的失败 Url，其次**如果你不手动处理这个异常，这个异常是不会被处理的**。

## 一、常见异常

我们先来列举下一些 Spring Security 中常见的异常：

- `UsernameNotFoundException`（用户不存在）
- `DisabledException`（用户已被禁用）
- `BadCredentialsException`（坏的凭据）
- `LockedException`（账户锁定）
- `AccountExpiredException` （账户过期）
- `CredentialsExpiredException`（证书过期）
- ...

以上列出的这些异常都是 `AuthenticationException` 的子类，然后我们来看看 Spring security 如何处理 `AuthenticationException` 异常的。

## 二、源码分析

我们知道异常处理一般在过滤器中处理，我们在 `AbstractAuthenticationProcessingFilter` 中找到了对 `AuthenticationException` 的处理：

（1）在 `doFilter()` 中，捕捉了 `AuthenticationException` 异常，并交给了 `unsuccessfulAuthentication()` 处理。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180403142646331.png)

（2）在 `unsuccessfulAuthentication()` 中，转交给了 `SimpleUrlAuthenticationFailureHandler` 类的 `onAuthenticationFailure()` 处理。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180403142555587.png)

（3）在 `onAuthenticationFailure()` 中，首先判断有没有设置 `defaultFailureUrl`。

- 如果没有设置，直接返回 401 错误，即 `HttpStatus.UNAUTHORIZED` 的值。

- 如果设置了，首先执行 `saveException()` 方法。然后判断 `forwardToDestination` ，即是否是服务器跳转，默认使用重定向即客户端跳转。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/2018040314320740.png)

（4）在 `saveException()` 方法中，首先判断`forwardToDestination`，如果使用服务器跳转则写入 Request，客户端跳转则写入 Session。写入名为 `SPRING_SECURITY_LAST_EXCEPTION` ，值为 `AuthenticationException`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180403143720216.png)

至此 Spring security 完成了异常处理，总结一下流程：

--> AbstractAuthenticationProcessingFilter`.doFilter()` 

--> AbstractAuthenticationProcessingFilter.`unsuccessfulAuthentication()` 

--> SimpleUrlAuthenticationFailureHandler.`onAuthenticationFailure()` 

--> SimpleUrlAuthenticationFailureHandler.`saveException()`

## 三、处理异常

上面源码说了那么多，真正处理起来很简单，我们只需要指定错误的url，然后再该方法中对异常进行处理即可。

（1）指定错误Url，`WebSecurityConfig` 中添加 `.failureUrl("/login/error")`

```java
...
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
            // 登录失败Url
            .failureUrl("/login/error")
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
...
```

（2）在 Controller 中处理异常

```java
@RequestMapping("/login/error")
public void loginError(HttpServletRequest request, HttpServletResponse response) {
    response.setContentType("text/html;charset=utf-8");
    AuthenticationException exception =
            (AuthenticationException)request.getSession().getAttribute("SPRING_SECURITY_LAST_EXCEPTION");
    try {
        response.getWriter().write(exception.toString());
    }catch (IOException e) {
        e.printStackTrace();
    }
}
```

我们首先获取了 session 中的 `SPRING_SECURITY_LAST_EXCEPTION` 。为了演示，我只是简单的将错误信息返回给了页面。运行程序，当我们输入错误密码时：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201804/20180403145530517.png)
