---
title: Json Web Token 介绍与基本使用
tags: JWT
categories:
  - Java Web
  - SpringBoot
abbrlink: 7ac4f061
date: 2018-05-03 02:25:18
icons: [fas fa-fire red]
related_repos:
  - name: springboot_jwt
    url: https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/springboot_jwt
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、Session 与 JWT

### 1.1 传统 Cookie + Session

`Cookie`+`Session` 的存在主要是为了解决HTTP这一`无状态协议`下**服务器如何识别用户**的问题。

其原理就是在用户登录通过验证后，服务端将数据加密后存储在服务器 `Session` 中，同时服务器将 Session 的标识即 `SessionId` 存放在客户端 Cookie 中。

用户之后发起的请求都会携带 Cookie 信息，服务器根据 `SessionId` 寻回对应的 Session，从而完成验证，确认这是之前登陆过的用户。其工作原理如下图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180503001829205.png)

### 1.2 JWT 技术

JWT，即 `JSON Web Token`，当用户使用它的认证信息登陆系统之后，会返回给用户一个 JWT，用户只需要本地保存即可。

当用户希望访问一个受保护的路由或者资源的时候，在请求头部添加该 JWT。后台获取请求头部的JWT并进行验证，如果合法，则允许用户的行为。
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180503012041674.png)

## 二、JWT 的组成

一个 JWT 实际上就是一个字符串，它由三部分组成：**头部**、**载荷(`Payload`)**与**签名**。

### 2.1 头部

头部用于描述关于该JWT的最基本的信息，例如其**类型**以及**签名所用的算法**等。这也可以被表示成一个 JSON 对象。

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

>注：指明了这是一个JWT，并且我们所用的签名算法（后面会提到）是 HS256 算法。

对JSON串进行Base64编码，得到了 JWT 的 `Header`（头部）。

### 2.2 载荷

JWT 主体内容主要包含以下三种类型：

 1. `Reserved`（**保留声明**），它的含义就像是编程语言的保留字一样，属于JWT规范中规定好的，这类声明不是必须的，但是是建议使用的。有以下几种：
 - `iss`: JWT的签发者

 - `sub`: 该JWT所面向的用户

 - `aud`:JWT的接收方

 - `exp`(expires): 过期时间，这里是一个Unix时间戳

 - `iat`(issued at): 签发时间，这里是一个Unix时间戳

 2. `Public` （**公共声明**）：这类声明需要在 [IANA JSON Web Token Registry](http://www.iana.org/assignments/jwt/jwt.xhtml) 中定义或者提供一个URI，因为要避免重名等冲突。

 3. `Private`（**私有声明**）：根据业务需要自己定义的数据了

例如下面这个 JSON 串，最后一项 user_id 就是我自定义的数据：

```json
{
    "iss": "Jitwxs",
    "iat": 1441593502,
    "exp": 1441594722,
    "aud": "www.example.com",
    "sub": "wxs@example.com",
    "user_id":"10001"
}
```

对上面的 JSON 串进行 Base64 编码，得到了 JWT 的`载荷`

### 2.3 签名

将上面的两个编码后的字符串都用句号 .连接在一起（头部在前），形如：

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmcm9tX3VzZXIiOiJCIiwidGFyZ2V0X3VzZXIiOiJBIn0
```

最后，我们将上面拼接完的字符串用 `HS256` 算法进行加密。

在加密的时候，我们还需要提供一个**密钥（`secret`）**。如果我们用 mySecret作为密钥的话，那么得到结构如下：

```
rSWamyAYwuHCo7IFAgd1oRpSP7nzL7BF5t7ItqpKViM
```

这就是`签名`。最后将这一部分签名也拼接在被签名的字符串后面，我们就得到了完整的JWT，形如：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180503011703499.png)

我们就可以将这个传递给客户端了。

## 三、JWT 在客户端的存储

### 3.1 HTML5 Web Storage

Web 存储就是使用 `localStorage` 或 `sessionStorage` 进行存储，许多人使用这种方式存储，是因为能够防止 `CSRF(跨站请求伪造)`。但是使用 Web 存储，反而会增加 `XSS(跨站脚本攻击)`的风险。

`XSS`，简而言之是一种漏洞，攻击者可以注入在页面上运行的 JavaScript。为了防止 XSS，常见的反应是转义和编码所有不可信的数据。但是只要有一个漏洞存在，那么整个防护体系就完全失效。如果你使用的第三方脚本被攻击，你的 Web 存储仍然会收受到威胁。

### 3.2 Cookies

#### 3.2.1 CSRF

`CSRF(跨站请求伪造)`是一种典型的利用 cookie-session 漏洞的攻击。

假设你经常使用 bank.example.com 进行网上转账，在你提交转账请求时 bank.example.com 的前端代码会提交一个 HTTP 请求:

```http
POST /transfer HTTP/1.1
Host: bank.example.com
cookie: JsessionID=randomid; Domain=bank.example.com; Secure; HttpOnly
Content-Type: application/x-www-form-urlencoded

amount=100.00&routingNumber=1234&account=9876
```

你图方便没有登出 bank.example.com，随后又访问了一个恶意网站，该网站的 HTML 页面包含了这样一个表单：

```html
<form action="https://bank.example.com/transfer" method="post">
    <input type="hidden" name="amount" value="100.00"/>
    <input type="hidden" name="routingNumber" value="evilsRoutingNumber"/>
    <input type="hidden" name="account" value="evilsAccountNumber"/>
    <input type="submit" value="点击就送!"/>
</form>
```

你被“点击就送”吸引了，当你点了提交按钮时你已经向攻击者的账号转了100元。

现实中的攻击可能更隐蔽，恶意网站的页面可能使用 Javascript 自动完成提交。尽管恶意网站没有办法盗取你的 session cookie（从而假冒你的身份），但恶意网站向 bank.example.com 发起请求时，你的 cookie 会被自动发送过去。

很多人用`JWT+Web存储`的本心是为了防护 CRSF。这样做的原因是——因为 Cookie 的发送是完全由浏览器控制的，不受网页本身的控制。所以最简单直接的办法，就是不用 Cookie，不让自动发送认证信息成为可能。问题在于，这么干是有 XSS 风险的。那么怎么在使用 Cookie 的同时，还能防范 CSRF 呢？

#### 3.2.1 CSRF 的防护

#####  3.2.1.1 传统页面网站

在传统页面 Web 网站中，一般会使用 `CSRF Token`。这是个非常流行的做法。像 Tomcat 这类的容器都会自带 CSRF Token 的产生和检查 Filter。

CSRF Token 是这样工作的。客户端要首先向服务器请求一个带有提交表单的页面，服务器返回的页面中会嵌入一个 CSRF Token。当用户提交表单时，`CSRF Token`会被一起携带发给服务器做验证。所以当服务器看到CSRF Token，就可以放心大胆的确认用户的的确确是看看到了提交前的表单界面，从而避免了用户稀里糊涂提交一个被伪造的表单的可能性。

#####  3.2.1.2 SPA 应用

在 SPA 中，客户端与服务器之间的交互主要是通过接口完成的，没有页面的概念。此时，你的确可以照猫画虎的做一个接口让用户拿到 CSRF Token，但这样什么也确认不了。因为攻击者可以调用同样的接口，拿到合法的 CSRF Token。

这时有几种办法：

 1. 给所有接口都添加一个请求 secret，来标记其来自于合法的客户端。这个 secrect 可以是固定的随机字符串，也可以通过某些动态算法产生。对于 CSRF，浏览器只会做自动传 Cookie 而已，并不能帮助传入 secret。这样一来，就可以确定消除 CSRF 的风险。但注意，这个机制仅能防范 CSRF，而不能防范人为的攻击。黑客只要拿得到客户端，就一定能找到生成 secret 的办法。

 2. 用 `Same-Site Cookie`。当用户看到了 B 站点伪造的表单，点击了提交，向站点A发出请求时，被标记了 Same-Site=strict 的 Cookie 是不会被携带的，因为当时的主站点域名B和提交的站点域名 A 不一样。这是 Same-Site=strict 标记是个相对较新的标准。目前大部分浏览器都已经支持了。

 3. 总是用 JSON 格式提交。CSRF 可能发生的一个前提条件是**必须用传统表单**提交。这是因为传统表单提交可以跨域——你在站点B，可以提交表单给站点A。而 Ajax 的请求除非开启 `CORS`，是不允许跨域的，所以天然的屏蔽掉了这个问题。传统表单的提交的格式必然是 application/x-www-form-urlencoded。因此只要保证服务器能够拒绝处理所有这种格式的 POST 请求，就能确保 SPA 不受 CSRF 的影响。

## 四、 JWT 的优缺点与适用场景

### 4.1 优点

 1. **无状态**。Token机制在服务端不需要存储 session 信息，因为 Token 自身包含了所有登录用户的信息，只需要在客户端的 cookie 或本地介质存储状态信息.

 2. **去耦**。不需要绑定到一个特定的身份验证方案。Token 可以在任何地方生成，只要在你的API被调用的时候，你可以进行Token生成调用即可.

 3. **更适用于移动应用**。 当你的客户端是一个原生平台时，Cookie 是不被支持的（你需要通过Cookie容器进行处理），这时采用 Token 认证机制就会简单得多。

 4. **CSRF**。因为不再依赖于 Cookie，所以你就不需要考虑对 CSRF 的防范，但需要防范 XSS。

### 4.2 缺点

 1. **占用太多空间**。上面提到 JWT 存储在 Web 存储中会导致 XSS 攻击，那就只能存储在 Cookie 里了。但是，Cookie 的存储容量是有限制的（通常为 4 KB）。而 JWT 占用的空间又不是那么小，尤其是在使用无状态 JWT 时，所有的数据都被放到 JWT 里，数据大小很快就会超过 Cookie 的容量限制。

 2. **无法完全掌控JWT。**使用了 JWT，无法实现在服务器端对用户请求进行管理——管理员没法统计多少个人登录了，一个人登录了多少次，登陆了什么设备；同时，也无法强行“踢”掉一个用户的登录——JWT一旦生成，在失效之前，总是有效的。如果实现了一个 token 黑名单之类的功能，就等价于实现了 Session 机制，无状态带来的好处就无从谈起。这个限制对于任何一个要认真做用户风险控制的网站来说都是不可能接受的。

 3. **数据实效性差。** JWT 一旦被生成，就不会再和服务端有任何瓜葛，一旦服务端中的相关数据更新，JWT 中存储的数据由于得不到更新，就变成了过期的数据。

以上这些缺点都只针对`无状态 JWT`。如果使用`有状态 JWT`（JWT 中存储认证授权信息的 ID，具体数据存储在服务端），就可以规避这些缺点。不过，有状态 JWT 基本上等同于 Session / Cookie，没有存在价值。

### 4.3 适用场景

JWT 的最佳用途是「一次性授权 Token」，这种场景下的 Token 的特性如下：

- 有效期短

- 只希望被使用一次

例如使用邮箱注册时，需要发一封注册邮件进行验证，只需验证一次，并且超过一定时间后失效。

## 五、提高安全性

### 5.1 使用 https

**http 是明文传输**的。在 http 下，用户输入的任何信息，从他的电脑到服务器之间的每个链路节点都是明文的。在这里个链路中的任何地方，都可以截取到完整的数据，包含你的密码，认证token……

这就是为什么`https`是必须的。https主要提供三个保证：

- 端端加密。通过https交互的原始数据，只有用户的浏览器和最终的服务器可以看到。其他中间节点无法获）。

- 客户端可以认定要访问的服务器就是那个服务器。这是被证书体系所支撑的。一旦浏览器的地址栏出现了网址的证书信息，并且是绿色的提示，那么用户的心里就可以稳了。（当然国内其实也不完全是这样，讲多了查水表，懂者自懂）。

- 服务器可以认定访问的客户端就是合法的客户端。这种模式被称为“2-Way SSL”或者“Mutal SSL”。这种模式是可选的，需要多配置一个客户端证书，一般场景用不到，多见于企业Web服务。

使用了 https 后，为了进一步保证安全，将 Cookie 设置为 Secure。这样，浏览器就可以只在访问 https 网址时才会携带 Cookie。如果不做这样的设置，通过 https 站点设置的 Cookie，仍然会向 http 站点发送。当这个站点的域名解析被劫持，就可能造成向一个伪造的服务器发出你的认证信息。

### 5.2 认证信息不应该永久有效

很多人为了“用户体验”，选择让一个登录永久有效。这样做是非常危险的。因为一旦用户的认证信息被别人获取了，就永久性的失去了防御的机会。

因此，总是要保证认证信息的有效期是有限的。一般根据业务场景的安全级别不同，可以设为若干分钟～若干天。就算是社交娱乐类的应用，有效期最好也不要超过两周。

### 5.3 设计安全的 secret

JWT 唯一存储在服务端的只有一个 secret，个人认为这个 secret 应该设计成和用户相关的，而不是一个所有用户公用的统一值。这样可以有效的提高安全性。

例如用户修改密码时，需要重新生成 jwt，密码就需要和 secret 存在一定的联系。

## 六、总结

>注意：这里的总结是在 Web 开发中

 1. 可以使用 Session 也可以使用 Token 做认证，但是总是要保证服务器端可以管理 Session，通过 Session 是否存在来最终确定认证的有效性

 2. 将认证信息存放在标记为 `HttpOnly`，`Secure`，`Same-Site=strict` 的 `Cookie` 中，从而避免 XSS 和 CSRF

 3. 如果是传统的页面网站，请使用 `CSRF Token` 机制

 4. 总是使用https，只要你的网络链路经过了公网

 5. 保证 token/session **必须有一个有效期**

 6. **不要用** JWT 来做 Web 应用的会话管理，请用 Session / Cookie

## 七、示例程序

### 7.1 HelloWorld

光说不练假把式，下面只用三步来实现一个 SpringBoot 环境下的 JWT 的入门程序。

**导入 JJWT 包简化操作：**

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.0</version>
</dependency>
```

**第一步：JWT 工具类，直接使用即可**

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * JWT工具类
 * @author jitwxs
 * @since 2018/5/2 18:49
 */
public class JwtUtils {
    // 签名密钥（高度保密）
    private static final String SECRET = "qYYjXt7s1C*nEC%9RCwQGFA$YwPr$Jrj";
    // 签名算法
    private static final SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS512;
    // token前缀
    private static final String TOKEN_PREFIX = "Bearer ";

    /**
     * 生成JWT token
     * @param map 传入数据
     * @param maxAge 有效期（单位：ms）
     * @author jitwxs
     * @since 2018/5/3 23:19
     */
    public static String sign(Map<String,Object> map, long maxAge) {
        return sign(map, null, null, maxAge);
    }

    /**
     * 生成JWT token
     * @param map 传入数据
     * @param issuer 签发者
     * @param subject 面向用户
     * @param maxAge 有效期（单位：ms）
     * @author jitwxs
     * @since 2018/5/3 23:19
     */
    public static String sign(Map<String,Object> map, String issuer, String subject, long maxAge) {
        Date now  = new Date(System.currentTimeMillis());
        String jwt = Jwts.builder()
                .setClaims(map) // 设置自定义数据
                .setIssuedAt(now) // 设置签发时间
                .setExpiration(new Date(now.getTime() + maxAge)) // 设置过期时间
                .setIssuer(issuer) // 设置签发者
                .setSubject(subject) // 设置面向用户
                .signWith(signatureAlgorithm, SECRET)
                .compact();
        return TOKEN_PREFIX + jwt;
    }

    /**
     * 验证JWT token并返回数据。当验证失败时，抛出异常
     * @author jitwxs
     * @since 2018/5/3 23:20
     */
    public static Map unSign(String token) {
        try {
            return Jwts.parser()
                    .setSigningKey(SECRET)
                    .parseClaimsJws(token.replace(TOKEN_PREFIX,""))
                    .getBody();
        }catch (Exception e){
            throw new IllegalStateException("Token验证失败："+e.getMessage());
        }
    }

    public static void main(String[] args) {
        Map<String,Object> map = new HashMap<>();
        map.put("userName","admin");
        map.put("userId","001");

        String token = JwtUtils.sign(map, 3600_000);
//        String token = JwtUtils.sign(map, "jitwxs","普通用户",3600_000);
        System.out.println(JwtUtils.unSign(token));
    }
}
```

**第二步：Controller 类**

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestAttribute;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

/**
 * @author jitwxs
 * @date 2018/5/2 21:03
 */
@RestController
public class SystemController {
    private final String USER_NAME_KEY = "USER_NAME";
    private final Long EXP_IMT = 3600_000L;

    @GetMapping("/api/test")
    public Object hellWorld(@RequestAttribute(value = USER_NAME_KEY)  String username) {
        return "Welcome! Your USER_NAME : " + username;
    }

    @PostMapping("/login")
    public Object login(String name, String password) {
        if(isValidPassword(name,password)) {
            // 将用户名传入并生成jwt
            Map<String,Object> map = new HashMap<>();
            map.put(USER_NAME_KEY, name);
            String jwt = JwtUtils.sign(map, EXP_IMT);

            // 将jwt返回给客户端
            return new HashMap<String,String>(){{
                put("token", jwt);
            }};
        }else {
            return new ResponseEntity(HttpStatus.UNAUTHORIZED);
        }
    }

    /**
     * 验证密码是否正确，模拟
     */
    private boolean isValidPassword(String name, String password) {
        return "admin".equals(name) && "admin".equals(password);
    }
}
```
主要就两个方法，一个用于登陆，一个用于测试 JWT。

**第三步：Filter 类并在启动类中注册**

```java
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.util.PathMatcher;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Map;

/**
 * JWT过滤器
 * @author jitwxs
 * @date 2018/5/2 20:43
 */
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final String USER_NAME_KEY = "USER_NAME";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        try {
            // 从用户请求头中取token，key一般为Authorization
            String token = request.getHeader("Authorization");
            Map map = JwtUtils.unSign(token);
            // 将用户名存入request属性
            request.setAttribute(USER_NAME_KEY, map.get(USER_NAME_KEY));
        } catch (Exception e) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, e.getMessage());
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public FilterRegistrationBean jwtAuthenticationFilterRegister() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        //注入过滤器
        registration.setFilter(new JwtAuthenticationFilter());
        //拦截规则
        registration.addUrlPatterns("/api/*");
        //过滤器名称
        registration.setName("jwtAuthenticationFilter");
        return registration;
    }
}
```

到此结束！也许你有点迷糊，我来解释下：

 1. 在 Controller 的 login 方法中，如果用户登陆成功，将会返回一个 `jwt token` 给用户，在这个 token 中，存放了用户的名称。
 2. 用户访问 `api/test`，该 url 会被过滤器所拦截，在过滤器中，会取出 `jwt token` 并验证，**如果验证失败（不存在或错误），抛出异常**；**如果验证成功，取出用户的名称并放入r equest的  ` attribute` 中**。
 3. 如果通过了过滤器，在 `api/test` 中，取出 `request attribute` 中的用户名称并输出。

是不是很简单，让我们运行下程序：

（1）当我们未携带或携带错误的 `jwt token` 访问 `/api/test` 时，抛出异常：
![未携带JWT](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180504004546403.png)

（2）当我们成功登录后，返回 `jwt token`：
![登陆成功](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180504005420850.png)

（3）重新访问 `jwt token`，并在请求头中携带 `jwt token`：
![携带JWT](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/2018050400545552.png)

### 7.2 与 Spring Security 结合

本节将带你体验下 `JWT` 和 `Spring Security` 结合的实际应用。如果你还不了解 Spring Security，参考文章 [《SpringBoot 集成 Spring Security（1）——入门程序》](/5f5715e6.html)。

首先我们先整理下思路，我们的程序对外提供一些api服务，使用这些服务用户需要验证身份，且根据用户的身份，只允许用户访问部分资源，使用到的技术为 Spring Security + JWT。

有了思路，我们来想一下流程，Spring Security 本身可以帮助我们鉴别用户身份和权限，那么流程大概如下：

 1. 用户登录，如果登录失败，返回错误信息；如果登录成功，返回一个 `JWT token` 给用户。
 2. 如果用户登录成功，用户携带着 JWT token 访问 api 服务，如果用户身份允许，则返回 api 结果；否则不允许获取。

为了实现这个流程，我们需要自定义两个过滤器，作用如下：

- 过滤器1：当用户登录成功后，将 JWT token 返回给用户

- 过滤器2：当用户访问 api 服务验证时，验证携带的 token

思路理的差不多了，下面开始实践吧！

#### 7.2.1 准备数据库

数据库为 Spring Security 常规的三张表，即用户表、权限表、用户权限表。

```sql
CREATE TABLE `sys_role`  (
  `id` int(11) NOT NULL,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

INSERT INTO `sys_role` VALUES (1, 'ROLE_ADMIN');
INSERT INTO `sys_role` VALUES (2, 'ROLE_USER');

CREATE TABLE `sys_user`  (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `password` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 3 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

INSERT INTO `sys_user` VALUES (1, 'admin', '123');
INSERT INTO `sys_user` VALUES (2, 'jitwxs', '123');

DROP TABLE IF EXISTS `sys_user_role`;
CREATE TABLE `sys_user_role`  (
  `user_id` int(11) NOT NULL,
  `role_id` int(11) NOT NULL,
  PRIMARY KEY (`user_id`, `role_id`) USING BTREE,
  INDEX `fk_role_id`(`role_id`) USING BTREE,
  CONSTRAINT `fk_role_id` FOREIGN KEY (`role_id`) REFERENCES `sys_role` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `fk_user_id` FOREIGN KEY (`user_id`) REFERENCES `sys_user` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

INSERT INTO `sys_user_role` VALUES (1, 1);
INSERT INTO `sys_user_role` VALUES (2, 2);
```

在配置文件中添加数据库信息：

```ini application.properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/test1?useUnicode=true&characterEncoding=utf-8&useSSL=true
spring.datasource.username=root
spring.datasource.password=root

mybatis.configuration.map-underscore-to-camel-case=true
```

#### 7.2.2 导入依赖、准备实体、Mapper、Service

使用到的依赖包如下，主要就是 spring security、mybatis 和 jjwt：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>1.3.1</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.7</version>
    </dependency>

    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt</artifactId>
        <version>0.9.0</version>
    </dependency>
</dependencies>
```

（1）实体

```java
public class SysUser implements Serializable{
    static final long serialVersionUID = 1L;

    private Integer id;

    private String username;

    private String password;

   // 省略getter/setter
}
```

```java
public class SysRole implements Serializable {
    static final long serialVersionUID = 1L;

    private Integer id;

    private String name;

    // 省略getter/setter
}
```

```java
public class SysUserRole implements Serializable {
    static final long serialVersionUID = 1L;

    private Integer userId;

    private Integer roleId;

    // 省略getter/setter
}
```

（2）Mapper

```java
@Mapper
public interface SysUserMapper {
    @Select("SELECT * FROM sys_user WHERE id = #{id}")
    SysUser selectById(Integer id);

    @Select("SELECT * FROM sys_user WHERE username = #{username}")
    SysUser selectByUsername(String username);
}
```

```java
@Mapper
public interface SysRoleMapper {
    @Select("SELECT * FROM sys_role WHERE id = #{id}")
    SysRole selectById(Integer id);
}
```

```java
@Mapper
public interface SysUserRoleMapper {
    @Select("SELECT * FROM sys_user_role WHERE user_id = #{userId}")
    List<SysUserRole> listByUserId(Integer userId);
}
```

（3）Service

```java
@Service
public class SysUserService {
    @Autowired
    private SysUserMapper userMapper;

    public SysUser selectById(Integer id) {
        return userMapper.selectById(id);
    }

    public SysUser selectByUsername(String username) {
        return userMapper.selectByUsername(username);
    }
}
```

```java
@Service
public class SysRoleService {
    @Autowired
    private SysRoleMapper roleMapper;

    public SysRole selectById(Integer id){
        return roleMapper.selectById(id);
    }
}
```

```java
@Service
public class SysUserRoleService {
    @Autowired
    private SysUserRoleMapper userRoleMapper;

    public List<SysUserRole> listByUserId(Integer userId) {
        return userRoleMapper.listByUserId(userId);
    }
}
```

#### 7.2.3 编写 Controller

Controller中，一个是登录失败的处理，剩下两个模拟 api 资源，一个需要 `ROLE_ADMIN` 身份，一个需要 `ROLE_USER` 身份。其中 `USER_ID_KEY` 为从 request 中取出的值。

```java
@Controller
public class LoginController {
    private final String USER_ID_KEY = "USER_ID";

    @RequestMapping("/login/error")
    @ResponseBody
    public String loginError(HttpServletRequest request, HttpServletResponse response) {
        AuthenticationException exception =
                (AuthenticationException)request.getSession().getAttribute("SPRING_SECURITY_LAST_EXCEPTION");
        return exception.toString();
    }

    @GetMapping("/api/admin")
    @ResponseBody
    @PreAuthorize("hasRole('ROLE_ADMIN')")
    public Object helloAdmin(@RequestAttribute(value = USER_ID_KEY)  Integer userId) {
        return "Welcome Admin! Your USER_ID : " + userId;
    }

    @GetMapping("/api/user")
    @ResponseBody
    @PreAuthorize("hasRole('ROLE_USER')")
    public Object helloUser(@RequestAttribute(value = USER_ID_KEY)  Integer userId) {
        return "Welcome User! Your USER_ID : " + userId;
    }
}
```

#### 7.2.4 准备工具类

工具类除了上面使用的 JwtUtils 以外，还要加上一个用于获取 Bean 的工具类，因为我们要在过滤器中查询用户权限。

```java
/**
 * @author jitwxs
 * @date 2018/3/20 14:22
 */
@Component
public class SpringBeanFactoryUtils implements ApplicationContextAware {
    private static ApplicationContext context = null;

    public static <T> T getBean(Class<T> type) {
        return context.getBean(type);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        if (SpringBeanFactoryUtils.context == null) {
            SpringBeanFactoryUtils.context = applicationContext;
        }
    }
}
```

#### 7.2.5 编写登录的过滤器

前面说了那么多无关紧要的，下面开始重点了，现在编写第一个过滤器，用于登录成功后生成 JWT token 并返回给用户。

```java
import jit.wxs.jwt.entity.SysUser;
import jit.wxs.jwt.service.SysUserService;
import jit.wxs.jwt.utils.JwtUtils;
import jit.wxs.jwt.utils.SpringBeanFactoryUtils;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * 该filter继承自UsernamePasswordAuthenticationFilter
 * 在验证用户名密码正确后，生成一个token，并将token返回给客户端
 * @author jitwxs
 * @since 2018/5/4 10:34
 */
public class JwtLoginFilter extends UsernamePasswordAuthenticationFilter {
    static final String USER_ID_KEY = "USER_ID";

    private AuthenticationManager authenticationManager;

    public JwtLoginFilter(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;
    }

    /**
     * 该方法在Spring Security验证前调用
     * 将用户信息从request中取出，并放入authenticationManager中
     * @author jitwxs
     * @since 2018/5/4 10:35
     */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest req, HttpServletResponse res) throws AuthenticationException {
        try {
            String username = req.getParameter("username");
            String password = req.getParameter("password");

            // 将用户信息放入authenticationManager
            return authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            username,
                            password,
                            Collections.emptyList())
            );
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 该方法在Spring Security验证成功后调用
     * 在这个方法里生成JWT token，并返回给用户
     * @author jitwxs
     * @since 2018/5/4 10:37
     */
    @Override
    protected void successfulAuthentication(HttpServletRequest req, HttpServletResponse res, FilterChain chain, Authentication auth) throws IOException, ServletException {
        String username = ((User)auth.getPrincipal()).getUsername();

	    // 从数据库中取出用户信息
        SysUserService userService = SpringBeanFactoryUtils.getBean(SysUserService.class);
        SysUser user = userService.selectByUsername(username);;

        // 将用户id放入JWT token
        Map<String,Object> map = new HashMap<>();
        map.put(USER_ID_KEY, user.getId());
        String token = JwtUtils.sign(map, 3600_000);

	// 将token放入响应头中
        res.addHeader("Authorization", token);
    }
}
```

#### 7.2.6 编写验证 token 的过滤器

1. 在 `isProtectedUrl()` 方法中规定了只拦截 `/api` 下的所有请求
2. 在 `getAuthentication()` 方法中验证 token，验证失败，返回 null；验证成功，返回的对象中包含用户的角色
3. 在 `doFilterInternal()` 中，如果上面返回值为 null，则抛出异常；否则将返回对象，也就是用户信息注入到框架中，并放行

```java
import jit.wxs.jwt.entity.SysRole;
import jit.wxs.jwt.entity.SysUserRole;
import jit.wxs.jwt.service.SysRoleService;
import jit.wxs.jwt.service.SysUserRoleService;
import jit.wxs.jwt.utils.JwtUtils;
import jit.wxs.jwt.utils.SpringBeanFactoryUtils;
import org.springframework.security.authentication.AuthenticationCredentialsNotFoundException;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.www.BasicAuthenticationFilter;
import org.springframework.util.AntPathMatcher;
import org.springframework.util.PathMatcher;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;

/**
 * JWT过滤器
 * 在访问受限URL前，验证JWT token
 * @author jitwxs
 * @since 2018/5/2 20:43
 */
public class JwtAuthenticationFilter extends BasicAuthenticationFilter {
    private static final PathMatcher pathMatcher = new AntPathMatcher();

    static final String USER_ID_KEY = "USER_ID";

    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        if (isProtectedUrl(request)) {
            UsernamePasswordAuthenticationToken authentication = getAuthentication(request);

            // 如果验证失败，设置异常；否则将UsernamePasswordAuthenticationToken注入到框架中
            if (authentication == null) {
                //手动设置异常
                request.getSession().setAttribute("SPRING_SECURITY_LAST_EXCEPTION", new AuthenticationCredentialsNotFoundException("权限认证失败"));
                // 转发到错误Url
                request.getRequestDispatcher("/login/error").forward(request, response);
            } else {
                SecurityContextHolder.getContext().setAuthentication(authentication);
                filterChain.doFilter(request, response);
            }
        }
    }

    /**
     * 验证token
     * @return 成功返回包含角色的UsernamePasswordAuthenticationToken；失败返回null
     */
    private UsernamePasswordAuthenticationToken getAuthentication(HttpServletRequest request) {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        String token = request.getHeader("Authorization");

        if (token != null) {
            Map map = JwtUtils.unSign(token);

            Integer userId = (Integer) map.get(USER_ID_KEY);
            if (userId != null) {
                // 将用户id放入request中
                request.setAttribute(USER_ID_KEY, userId);

                // 从数据库中获取用户角色
                SysUserRoleService userRoleService = SpringBeanFactoryUtils.getBean(SysUserRoleService.class);
                SysRoleService roleService = SpringBeanFactoryUtils.getBean(SysRoleService.class);
                List<SysUserRole> userRoles = userRoleService.listByUserId(userId);
                for (SysUserRole userRole : userRoles) {
                    SysRole role = roleService.selectById(userRole.getRoleId());
                    authorities.add(new SimpleGrantedAuthority(role.getName()));
                }

                // 这里直接注入角色，因为JWT已经验证了用户合法性，所以principal和credentials直接为null即可
                return new UsernamePasswordAuthenticationToken(null, null, authorities);
            }
            return null;
        }
        return null;
    }

    //只对/api/*下请求拦截
    private boolean isProtectedUrl(HttpServletRequest request) {
        return pathMatcher.match("/api/**", request.getServletPath());
    }
}
```

#### 7.2.7 注入过滤器

完成了最关键的过滤器，下面只要将这两个过滤器注入到 Spring Security 中即可。

首先给出 `CustomUserDetailsService` ，这是 Spring Security 的知识，不再赘述。

```java
@Service("userDetailsService")
public class CustomUserDetailsService implements UserDetailsService {
    @Autowired
    private SysUserService userService;

    @Autowired
    private SysRoleService roleService;

    @Autowired
    private SysUserRoleService userRoleService;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        SysUser user = userService.selectByUsername(s);

        // 判断用户是否存在
        if(user == null) {
            throw new UsernameNotFoundException("用户名不存在");
        }

        // 添加权限
        List<SysUserRole> userRoles = userRoleService.listByUserId(user.getId());
        for (SysUserRole userRole : userRoles) {
            SysRole role = roleService.selectById(userRole.getRoleId());
            authorities.add(new SimpleGrantedAuthority(role.getName()));
        }

        // 返回UserDetails实现类
        return new User(user.getUsername(), user.getPassword(), authorities);
    }
}
```

在 `WebSecurityConfig` 中注入我们的过滤器，也就是 `addFilter()` 方法：

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private CustomUserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(new PasswordEncoder() {
            @Override
            public String encode(CharSequence charSequence) {
                return charSequence.toString();
            }

            @Override
            public boolean matches(CharSequence charSequence, String s) {
                return s.equals(charSequence.toString());
            }
        });
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                // 设置登陆页
                .formLogin().loginPage("/login")
                // 设置登陆失败页
                .failureUrl("/login/error")
                // 设置登陆成功页
                .defaultSuccessUrl("/").permitAll()
                .and()
                .addFilter(new JwtLoginFilter(authenticationManager()))
                .addFilter(new JwtAuthenticationFilter(authenticationManager()))
                .logout().permitAll();

        http.csrf().disable();
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        // 设置拦截忽略文件夹，可以对静态资源放行
        web.ignoring().antMatchers("/css/**", "/js/**");
    }
}
```

#### 7.2.8 运行程序

下面来验证下，运行程序。

在未登录情况下，是无法访问 api 资源的：
![未登录](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/2018050922062413.png)

![未登录](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509220629367.png)

我们使用一个角色为 ROLE_USER 的用户登录，登录成功后会返回一个token，在响应头：
![登录成功](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509220921978.png)

下面带着这个 token 再次访问 api 资源：
![携带JWT](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509221133208.png)

![携带JWT](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201805/20180509221141478.png)
