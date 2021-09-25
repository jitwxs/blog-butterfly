---
title: SpringBoot 集成 Shiro 安全框架
tags: 
  - Shiro
  - SpringBoot
categories:
  - 安全框架
  - Shiro
abbrlink: 30819bdf
date: 2018-03-20 23:23:32
related_repos:
  - name: shiro-sample
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-sample/shiro-sample
    rel: nofollow noopener noreferrer
    target: _blank
---

Shiro 是在 Java Web 开发中，比较常见的安全框架技术，在本文章，将介绍如何在 SpringBoot 中去使用 Shiro。在本篇文章中，使用的技术为：SpringBoot 2.0、Shiro 和 MyBatis-Plus，下面就跟着我来一步步实践吧。

## 一、导入依赖

```xml
<!-- SpringBoot Web包 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- SpringBoot AOP包，用于Shiro授权 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<!--Shiro核心包 -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.2.5</version>
</dependency>

<!--MyBatis-Plus和SpringBoot整合包 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatisplus-spring-boot-starter</artifactId>
    <version>1.0.5</version>
</dependency>
    
<!--MyBatis-Plus核心包 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus</artifactId>
    <version>2.1.9</version>
</dependency>

<!-- 数据库连接包 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

## 二、自定义 ShiroRealm

继承 `AuthorizingRealm` 类，并重写用于认证的 `doGetAuthenticationInfo` 和用于授权的 `doGetAuthorizationInfo`，如果你在 SSM 中使用过 Shiro，就很简单了。

```java
/**
 * @author jitwxs
 * @date 2018/3/20 9:35
 */
public class ShiroRealm extends AuthorizingRealm {

    @Autowired
    IUserService userService;

    @Autowired
    IUserRoleService userRoleService;

    @Autowired
    IRoleService roleService;

    @Autowired
    IPermissionService permissionService;

    @Autowired
    IRolePermissionService rolePermissionService;

    /**
     * 用户认证
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        if(token.getPrincipal() == null) {
            return null;
        }

        // 获取用户名和密码
        String name = token.getPrincipal().toString();
        String password = new String((char[])token.getCredentials());

        User user = userService.findByName(name);

        if(user  == null || !password.equals(user.getPassword())) {
            throw new IncorrectCredentialsException("登录失败，用户名或密码错误");
        }

        //验证authenticationToken和simpleAuthenticationInfo的信息
        SimpleAuthenticationInfo simpleAuthenticationInfo = new SimpleAuthenticationInfo(name,password, getName());

        // 返回一个身份信息
        return simpleAuthenticationInfo;
    }

    /**
     * 角色权限和对应权限添加
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        // 获取用户名
        String name = (String)principalCollection.getPrimaryPrincipal();
        // 获取用户对象
        User user = userService.findByName(name);
        // 添加角色和权限

        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();

        List<Role> roles = getRoles(user.getId());

        for(Role role : roles) {
            // 添加角色
            simpleAuthorizationInfo.addRole(role.getName());

            // 添加权限
            List<Permission> permissions = getPermission(role.getId());
            for(Permission permission : permissions) {
                simpleAuthorizationInfo.addStringPermission(permission.getName());
            }
        }
        return simpleAuthorizationInfo;
    }


    private List<Role> getRoles(String userId) {
        List<UserRole> userRoles = userRoleService.selectList(
                new EntityWrapper<UserRole>()
                    .eq("user_id", userId));
        List<Role> list = new ArrayList<>();

        for(UserRole userRole : userRoles) {
            Role role = roleService.selectById(userRole.getRoleId());
            list.add(role);
        }
        return list;
    }

    private List<Permission> getPermission(String roleId) {
        List<RolePermission> rolePermissions = rolePermissionService.selectList(
                new EntityWrapper<RolePermission>()
                    .eq("role_id", roleId));

        List<Permission> list = new ArrayList<>();

        for(RolePermission rolePermission : rolePermissions) {
            Permission permission = permissionService.selectById(rolePermission.getPermissionId());
            list.add(permission);
        }
        return list;
    }
}
```

## 三、设置 ShiroConfig

SpringBoot 相较于 SSM，将配置使用注解形式引入，那么原本写入 xml 的 Shrio 配置文件，就可以用注解注入了：

```java
/**
 * @author jitwxs
 * @date 2018/3/20 10:00
 */
@Configuration
public class ShiroConfig {
    /**
     * 注入ShiroRealm
     * 不能省略，会导致ShiroRealm中service无法注入
     */
    @Bean
    public ShiroRealm shiroRealm() {
        return new ShiroRealm();
    }

    /**
     * 注入securityManager
     */
    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager manager = new DefaultWebSecurityManager();
        manager.setRealm(shiroRealm());
        return manager;
    }

    /**
     * Filter工厂，设置过滤条件
     */
    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();

        // Shiro的核心安全接口
        bean.setSecurityManager(securityManager);

        // 设置登陆页
        bean.setLoginUrl("/login");

        // 自定义拦截规则
        Map<String,String> map = new HashMap<>(16);
        map.put("/", "anon");
        // 设置登出
        map.put("/logout", "logout");
        // 对所有用户认证
        map.put("/**", "authc");

        bean.setFilterChainDefinitionMap(map);
        return bean;
    }

    /**
     * 注册AuthorizationAttributeSourceAdvisor
     * 如果要开启注解@RequiresRoles等注解，必须添加
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager manager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(manager);

        return advisor;
    }
}
```

## 四、编写 Controller 测试

这里只是简单的编写了一些测试方法：

```java
@Controller
public class LoginController {
    @Autowired
    IUserService userService;

    @GetMapping("/")
    public String index() {
        return "index.html";
    }

    @GetMapping("/login")
    public String login() {
        return "login.html";
    }

    @PostMapping("/login")
    public String login(User user) {

        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken usernamePasswordToken = new UsernamePasswordToken(
                user.getName(), user.getPassword());
        //进行验证，这里可以捕获异常，然后返回对应信息
        subject.login(usernamePasswordToken);

        return "redirect:/home";
    }

    @GetMapping("/home")
    public String home() {
        return "home.html";
    }

    @RequestMapping("/logout")
    public String logout() {
        return "redirect:/logout";
    }

    @RequiresRoles("admin")
    @RequiresPermissions("create")
    @RequestMapping("/adminCreate")
    @ResponseBody
    public String adminCreate() {
        return "只有[admin]身份且具有[create]权限才能看见这句话";
    }
}
```

## 五、测试程序

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180320224257488.png)

我的数据库拥有以下身份和权限：

| Role | Permissions |
|:-------------|:-------------|
| admin | create、update、delete、select |
| teacher | create、select |
| student | select |

拥有以下几位用户：

| User | Role |
|:-------------|:-------------|
| jitwxs | admin |
| zhangsan | student |

当我测试该方法时：

```java
@RequiresRoles("admin")
@RequiresPermissions("update")
@RequestMapping("/adminCreate")
@ResponseBody
public String adminCreate() {
    return "只有[admin]身份且具有[create]权限才能看见这句话";
}
```
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180320224642625.png)

## 六、番外

在这个例子中，踩了几个坑，罗列一下：

**（1）ShiroRealm 中无法注入 Service**

解决办法：[《Shiro 解决无法注入 Service 问题》](/6a006994.html) 

**（2）ShiroRealm 的授权方法 `doGetAuthorizationInfo()` 没有调用**

解决办法：缺少 AOP，导入 AOP 包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
