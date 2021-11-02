---
title: SpringBoot 配置国际化
tags: i18n
categories: 
  - Java Web
  - Spring Framework
abbrlink: 885663
date: 2018-12-02 21:03:43
related_repos:
  - name: i18n-sample
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-sample/i18n-sample
---

## 一、LocaleResolver

国际化的支持中一个重要的类是 LocaleResolver，它提供了四种默认实现：

1. `AcceptHeaderLocaleResolver`
    没有任何具体实现，通过浏览器头部的语言信息来进行多语言选择。
2. `FixedLocaleResolver`
    设置固定的语言信息，这样整个系统的语言是一成不变的，用处不大。
3. `CookieLocaleResolver`
    将语言信息设置到 Cookie 中，这样整个系统就可以获得语言信息
4. `SessionLocaleResolver`
    将语言信息放到 Session 中，这样整个系统就可以从Session中获得语言信息。

一般来说，我们都使用基于 Cookie 或者是基于 Session 的 LocaleResolver，也可以自定义 LocaleResolver。

当每次请求时，都会调用我们设置的 LocaleResolver，根据它的 `resolveLocale()`方法来解析 `Locale`，从而实现国际化。

## 二、MessageSource

Spring 中定义了一个 MessageSource 接口，以用于支持信息的国际化和包含参数的信息的替换。MessageSource 接口的定义如下：

```java
public interface MessageSource {
    /**
     * 解析code对应的信息进行返回，如果对应的code不能被解析则返回默认信息defaultMessage。
     * @param 需要进行解析的code，对应资源文件中的一个属性名
     * @param 需要用来替换code对应的信息中包含参数的内容，如：{0},{1,date},{2,time}
     * @param defaultMessage 当对应code对应的信息不存在时需要返回的默认值
     * @param locale 对应的Locale
     * @return
     */
    String getMessage(String code, Object[] args, String defaultMessage, Locale locale);

    /**
     * 解析code对应的信息进行返回，如果对应的code不能被解析则抛出异常NoSuchMessageException
     * @param code 需要进行解析的code，对应资源文件中的一个属性名
     * @param args 需要用来替换code对应的信息中包含参数的内容，如：{0},{1,date},{2,time}
     * @param locale 对应的Locale
     * @return 
     * @throws NoSuchMessageException 如果对应的code不能被解析则抛出该异常
     */
    String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException;

    /**
     * 通过传递的MessageSourceResolvable对应来解析对应的信息
     * @param resolvable 
     * @param locale 对应的Locale
     * @return 
     * @throws NoSuchMessageException 如不能解析则抛出该异常
     */
    String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;
}
```

我们主要使用 `ResourceBundleMessageSource` 和 `ReloadableResourceBundleMessageSource`  这两个实现类。

`ResourceBundleMessageSource` 是基于 JDK ResourceBundle 的 MessageSource 接口实现类。它会将访问过的 ResourceBundle 缓存起来，以便于下次直接从缓存中获取进行使用。

如果我们使用了 Springboot 的配置项 `spring.messages.basename`，它就是默认的实现类。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202202942645.png)

`ReloadableResourceBundleMessageSource` 是以 ResourceBundleMessageSource 结尾的，但实际上它跟ResourceBundleMessageSource没有什么直接的关系。

ReloadableResourceBundleMessageSource 也是对 MessageSource 的一种实现，其用法配置等和 ResourceBundleMessageSource 基本一致。所不同的是 ReloadableResourceBundleMessageSource 内部是**使用 PropertiesPersister 来加载对应的文件**，这包括 `properties` 文件和 `xml` 文件，然后使用 java.util.Properties 来保存对应的数据。

这两者的详细比较可以参考文章：[Spring（21）——国际化MessageSource](https://blog.csdn.net/elim168/article/details/77891450)

## 三、入门程序

### 3.1 国际化文件

在 resource 目录下建立文件夹 `i18`，在其中建立以下6个文件：

(1) login.properties、login_zh_CN.properties:

```ini login.properties & login_zh_CN.properties
login.username=用户名
login.password=密码
login.title=登录
login.verify.code=验证码
```

(2) login_en_US.properties:

```ini login_en_US.properties
login.username=username
login.password=password
login.title=login
login.verify.code=verifyCode
```

(3) messages.properties、messages_zh_CN.properties

```ini messages.properties & messages_zh_CN.properties
language.en_US=英语
language.zh_CN=中文
welcome.msg={0}：欢迎登陆系统
```

(4) messages_en_US.properties

```ini messages_en_US.properties
language.en_US=English
language.zh_CN=Chinese
welcome.msg={0}：Welcome to login
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202203905940.png)

`zh_CN` 和 `en_US` 代表相应语言的文件，而不带后缀的代表默认的语言文件，当出现不能匹配的语言时，从不带后缀的文件中读取。

### 3.2 application.yaml

```yaml
spring:
  messages:
    encoding: UTF-8
    cache-duration: 3600 # 加载国际化文件的缓存时间，单位为秒，默认为永久缓存
    basename: i18n/messages,i18n/login
    fallback-to-system-locale: false # 当找不到当前语言的资源文件时,如果为true默认找当前系统的语言对应的资源文件，如果为false即加载系统默认的如messages.properties文件
    always-use-message-format: false # MessageFormat定义了如何展示信息的格式；默认：false
```

`spring.messages.basename` 指定了国际化文件的路径，默认在 `resource` 目录下查找。i18n 代表了目录名，messages 代表了国际化文件的前缀，同一个前缀的文件为一组，多组之前使用逗号隔开。

### 3.3 CustomSessionLocaleResolver

这里我们使用自定义的 LocaleResolver，实现基于 Session 的 LocaleResolver。

在 `resolveLocale` 方法中，首先检查请求参数是否携带 `18N_LANGUAGE`，如果存在，则将其保存到 session中；如果不存在，从 session 中取出并返回。如果 session 中也不存在，使用预设的，即 `Locale.getDefault()`。

```java
public class CustomSessionLocaleResolver implements LocaleResolver {
    private static final String I18N_LANGUAGE = "language";
    private static final String I18N_LANGUAGE_SESSION = "i18n_language_session";

    @Override
    public Locale resolveLocale(HttpServletRequest req) {
        String i18nLanguage = req.getParameter(I18N_LANGUAGE);
        Locale locale = Locale.getDefault();
        if (!StringUtils.isEmpty(i18nLanguage)) {
            String[] language = i18nLanguage.split("_");
            locale = new Locale(language[0], language[1]);

            //将国际化语言保存到session
            HttpSession session = req.getSession();
            session.setAttribute(I18N_LANGUAGE_SESSION, locale);
        } else {
            //如果没有带国际化参数，则判断session有没有保存，有保存，则使用保存的，也就是之前设置的，避免之后的请求不带国际化参数造成语言显示不对
            HttpSession session = req.getSession();
            Locale localeInSession = (Locale) session.getAttribute(I18N_LANGUAGE_SESSION);
            if (localeInSession != null) {
                locale = localeInSession;
            }
        }
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest req, HttpServletResponse res, Locale locale) {

    }
}
```

### 3.4 I18nConfig

该类为配置类，目前只是简单的将 CustomSessionLocaleResolver 注入进去：

```java
@Configuration
public class I18nConfig {
    @Bean
    public LocaleResolver localeResolver() {
        return new CustomSessionLocaleResolver();
    }
}
```

### 3.5 login.html

编写一个登录页来测试下，这里使用到了模板引擎，因此首先在 pom 中导入依赖：

```xml pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

然后编写 login 页面：

```html login.html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="#{login.title}">登陆</title>
</head>
<body>
<div class="container">
    <div class="form row">
        <div class="form-horizontal col-md-offset-3" id="login_form">
            <h3 class="form-title"></h3>
            <div>
                <a th:text="#{language.zh_CN}" th:href="@{/login(language='zh_CN')}">中文</a>
                <a th:text="#{language.en_US}" th:href="@{/login(language='en_US')}">英文</a>
            </div>
            <form class="form-horizontal" method="post" action="/login">
                <div class="col-md-9">
                    <div class="form-group">
                        <input class="form-control" type="text" th:placeholder="#{login.username}" name="username" autofocus="autofocus"/>
                    </div>
                    <div class="form-group">
                        <input class="form-control" type="password" th:placeholder="#{login.password}" name="password"/>
                    </div>
                    <div class="form-group">
                        <input class="form-control" type="text" th:placeholder="#{login.verify.code}" name="verifyCode" maxlength="4"/>
                    </div>
                    <div class="form-group">
                        <button type="submit" class="btn btn-success pull-right" th:text="#{login.title}">登录</button>
                    </div>
                </div>
            </form>
        </div>
    </div>
</div>
<script>
</script>
</body>
</html>
```

###  3.6 测试

这里就不再自定义 MessageSource 了，使用默认的 MessageSource，编写 Controller 进行测试：

```java
@Controller
public class TestController {
    @Autowired
    private MessageSource messageSource;

    @GetMapping("/test")
    @ResponseBody
    public Object test(Locale locale) {
        String[] params = {"Jack Zhang"};
        return messageSource.getMessage("welcome.msg", params, locale);
    }

    @GetMapping({"/login","/"})
    public String showLoginPage(){
        return "login.html";
    }
}
```

首先我们来测试下 "/test" 接口。首次请求时，session 中没有数据，因此使用默认的 `Locale.getDefault()`，也就是 zh_CN：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202205452674.png)

在 CustomSessionLocaleResolver 中，我们知道，只要增加 `language` 参数就能实现修改 语言，当修改语言为 en_US 时：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018120220591629.png)

当我们去掉参数后继续访问，语言已经被切换到 en_US 了，因为是从 session 中读取：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202205903911.png)

下面来测试下 login 页面，因为刚刚切换了语言，所以默认是 en_US:

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202210135825.png)

切换为中文后：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181202210221553.png)

去掉请求参数，也一样保持中文：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/201812022102540.png)

## 四、更多自定义功能

可以参考文章：[自己动手在Spring-Boot上加强国际化功能](http://zzzzbw.cn/article/7)
