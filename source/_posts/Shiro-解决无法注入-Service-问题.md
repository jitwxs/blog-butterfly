---
title: Shiro 解决无法注入 Service 问题
categories:
  - Java Web
  - Shiro
abbrlink: 6a006994
date: 2018-03-20 16:38:23
---

今天在使用`Spring Boot`集成`Shiro`时出现了无法注入`Service`的问题，解决后来记录一下。

**问题现象：**

在`ShiroRealm`中进行身份验证，要将登陆模块的Service注入进来进行验证，但是其值为null。


```java
public class ShiroRealm extends AuthorizingRealm {

    @Autowired
    ILoginService loginService; // <---这里值为null
    
    ...
}
```

**问题原因：**

 1. `ShiroRelam`属于`filter`即过滤器，它**在Spring未完成注入bean之前就已经拦截了**，因此无法注入。
 2. 对于SpringBoot，没有将`ShiroRealm`注入`Bean`。

**Spring MVC 解决办法：**

对于Spring MVC，解决办法很简单，直接将`Spring MVC`在`web.xml`中提到`shiro`之前即可：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    
    ...

    <!-- Spring MVC 前端控制器 -->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/springmvc.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

	...

    <!-- Shiro拦截器 -->
    <filter>
        <filter-name>shiroFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
        <init-param>
            <param-name>targetFilterLifecycle</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>shiroFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

**Spring Boot 解决办法：**

`ShiroConfig`在注入`SecurityManager`时`setRealm()`的参数`ShiroRealm`**不能自己new**出来，而要先将其注入`Bean`，然后调用（**这里是个坑，很多人其实挂在这一步**）。

**错误写法：**

```java
@Configuration
public class ShiroConfig {
    ...
    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager manager = new DefaultWebSecurityManager();
        manager.setRealm(new shiroRealm());
        return manager;
    }
    ...
}
```

**正确写法：**

```java
@Configuration
public class ShiroConfig {
    ...
    /**
     * 注入ShiroRealm 
     * 不能省略，会导致Service无法注入
     */
    @Bean
    public ShiroRealm myShiroRealm() {
        return new ShiroRealm();
    }

    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager manager = new DefaultWebSecurityManager();
        manager.setRealm(myShiroRealm());
        return manager;
    }
    ...
}
```

**终极大杀器：**

如果上面的方法使用后还是不能注入Service，那么试试下面这样做（**这种方法适用于在普通类中注入Service层**）：

创建一个`SpringBeanFactoryUtils`，用于获取Bean，记得添加`@Component`注解：

```java
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

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

后面使用的时候，使用`getBean()`方法即可：

```java
if (loginService == null) {
	loginService = SpringBeanFactoryUtils.getBean(ILoginService.class);
}
```
