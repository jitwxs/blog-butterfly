---
title: SpringBoot 集成 Spring Security（5）——权限控制
categories:
  - 安全框架
  - Spring Security
abbrlink: 71543ef6
date: 2018-05-15 19:09:28
icons: [fas fa-fire red]
related_repos:
  - name: springboot_security05
    url: https://github.com/jitwxs/blog-sample/blob/master/SpringBoot/springboot_security/springboot_security05
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

在第一篇中，我们说过，**用户<–>角色<–>权限**三层中，暂时不考虑权限，在这一篇，是时候把它完成了。

为了方便演示，这里的权限只是对角色赋予权限，也就是说同一个角色的用户，权限是一样的。当然了，你也可以精细化到为每一个用户设置权限，但是这不在本篇的探讨范围，有兴趣可以自己实验，原理都是一样的。

## 一、数据准备

### 1.1 创建 sys_permission 表

让我们先创建一张权限表，名为 `sys_permission`：

```sql
CREATE TABLE `sys_permission` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `url` varchar(255) DEFAULT NULL,
  `role_id` int(11) DEFAULT NULL,
  `permission` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `fk_roleId` (`role_id`),
  CONSTRAINT `fk_roleId` FOREIGN KEY (`role_id`) REFERENCES `sys_role` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
```

内容就是两条数据，通过 `url` + `role_id` + `permission` 唯一标识了一个角色访问某一url时的权限，其中权限暂定为c、r、u、d，代表了增、删、改、查。

![sys_permission 表数据](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180515185020939.png)

### 1.2 创建 POJO、Mapper、Service

（1）pojo

```java
package jit.wxs.entity;

import java.io.Serializable;
import java.util.Arrays;
import java.util.List;

/**
 * 权限实体类
 * @author jitwxs
 * @since 2018/5/15 18:11
 */
public class SysPermission implements Serializable {
    static final long serialVersionUID = 1L;

    private Integer id;

    private String url;

    private Integer roleId;

    private String permission;

    private List permissions;

    // 省略除permissions外的getter/setter

    public List getPermissions() {
        return Arrays.asList(this.permission.trim().split(","));
    }

    public void setPermissions(List permissions) {
        this.permissions = permissions;
    }
}
```

这里需要注意的时相比于数据库，多了一个 `permissions` 属性，该字段将 `permission` **按逗号分割**为了 list。

（2）mapper

```java
import jit.wxs.entity.SysPermission;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;

import java.util.List;

@Mapper
public interface SysPermissionMapper {
    @Select("SELECT * FROM sys_permission WHERE role_id=#{roleId}")
    List<SysPermission> listByRoleId(Integer roleId);
}
```

```java
import jit.wxs.demo.entity.SysRole;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;

@Mapper
public interface SysRoleMapper {
    @Select("SELECT * FROM sys_role WHERE id = #{id}")
    SysRole selectById(Integer id);

    @Select("SELECT * FROM sys_role WHERE name = #{name}")
    SysRole selectByName(String name);
}
```

（3）Service

SysPermissionService 中有一个方法，根据 roleId 获取所有的 `SysPermission`。

```java
package jit.wxs.service;

import jit.wxs.entity.SysPermission;
import jit.wxs.mapper.SysPermissionMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class SysPermissionService {
    @Autowired
    private SysPermissionMapper permissionMapper;

    /**
     * 获取指定角色所有权限
     */
    public List<SysPermission> listByRoleId(Integer roleId) {
        return permissionMapper.listByRoleId(roleId);
    }
}
```

```java
import jit.wxs.demo.entity.SysRole;
import jit.wxs.demo.mapper.SysRoleMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class SysRoleService {
    @Autowired
    private SysRoleMapper roleMapper;

    public SysRole selectById(Integer id) {
        return roleMapper.selectById(id);
    }

    public SysRole selectByName(String name) {
        return roleMapper.selectByName(name);
    }
}
```

### 1.3 修改接口

```java
@Controller
public class LoginController {
    ...

    @RequestMapping("/admin")
    @ResponseBody
    @PreAuthorize("hasPermission('/admin','r')")
    public String printAdminR() {
        return "如果你看见这句话，说明你访问/admin路径具有r权限";
    }

    @RequestMapping("/admin/c")
    @ResponseBody
    @PreAuthorize("hasPermission('/admin','c')")
    public String printAdminC() {
        return "如果你看见这句话，说明你访问/admin路径具有c权限";
    }
}
```

让我们修改下我们要访问的接口，`@PreAuthorize("hasPermission('/admin','r')")` 是关键，参数1指明了**访问该接口需要的url**，参数2指明了**访问该接口需要的权限**。

## 二、PermissionEvaluator

我们需要自定义对 `hasPermission()` 方法的处理，就需要自定义 `PermissionEvaluator`，创建类 `CustomPermissionEvaluator`，实现 `PermissionEvaluator` 接口。

```java
import jit.wxs.entity.SysPermission;
import jit.wxs.service.SysPermissionService;
import jit.wxs.service.SysRoleService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.stereotype.Component;

import java.io.Serializable;
import java.util.Collection;
import java.util.List;

@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {
    @Autowired
    private SysPermissionService permissionService;
    @Autowired
    private SysRoleService roleService;

    @Override
    public boolean hasPermission(Authentication authentication, Object targetUrl, Object targetPermission) {
        // 获得loadUserByUsername()方法的结果
        User user = (User)authentication.getPrincipal();
        // 获得loadUserByUsername()中注入的角色
        Collection<GrantedAuthority> authorities = user.getAuthorities();

        // 遍历用户所有角色
        for(GrantedAuthority authority : authorities) {
            String roleName = authority.getAuthority();
            Integer roleId = roleService.selectByName(roleName).getId();
            // 得到角色所有的权限
            List<SysPermission> permissionList = permissionService.listByRoleId(roleId);

            // 遍历permissionList
            for(SysPermission sysPermission : permissionList) {
                // 获取权限集
                List permissions = sysPermission.getPermissions();
                // 如果访问的Url和权限用户符合的话，返回true
                if(targetUrl.equals(sysPermission.getUrl())
                        && permissions.contains(targetPermission)) {
                    return true;
                }
            }
        }

        return false;
    }

    @Override
    public boolean hasPermission(Authentication authentication, Serializable serializable, String s, Object o) {
        return false;
    }
}
```

在 `hasPermission()` 方法中，参数 1 代表**用户的权限身份**，参数 2 参数 3 分别和 `@PreAuthorize("hasPermission('/admin','r')")` 中的参数对应，即**访问 url 和权限**。

思路如下：

 1. 通过 `Authentication` 取出登录用户的所有 `Role`

 2. 遍历每一个 `Role`，获取到每个`Role`的所有 `Permission`

 3. 遍历每一个 `Permission`，只要有一个 `Permission` 的 `url` 和传入的url相同，且该 `Permission` 中包含传入的权限，返回 true

 4. 如果遍历都结束，还没有找到，返回false

下面就是在 `WebSecurityConfig` 中注册 `CustomPermissionEvaluator `：

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    ...
    /**
     * 注入自定义PermissionEvaluator
     */
    @Bean
    public DefaultWebSecurityExpressionHandler webSecurityExpressionHandler(){
        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setPermissionEvaluator(new CustomPermissionEvaluator());
        return handler;
    }
    ...
}
```

## 三、运行程序

当我使用角色为 `ROLE_USER` 的用户仍然能访问，因为该用户访问 `/admin` 路径具有 `r` 权限：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018051519070954.png)
