---
title: Linux MySQL 安装教程
categories: 
  - 数据库
  - MySQL
typora-root-url: ..
abbrlink: 783eb9d
date: 2017-12-09 17:46:34
copyright_author: Jitwxs
---

## 一、Ubuntu 16.04

>sudo apt-get install mysql-server

一旦安装完成，MySQL 服务器应该自动启动。您可以在终端提示符后运行以下命令来检查 MySQL 服务器是否正在运行：

>sudo netstat -tap | grep mysql

如果要卸载：

>sudo apt-get autoremove mysql-server

## 二、Centos

如果 Centos 系统版本大于7，默认源使用 `mariadb` 替换掉了 MySQL, 如果仍要安装 MySQL，只能采取源码安装。

### 2.1 配置 YUM 源

(1) 下载 mysql 源安装包

>wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm

(2)  安装 mysql 源

>yum localinstall mysql57-community-release-el7-8.noarch.rpm

(3) 检查 mysql 源是否安装成功

>yum repolist enabled | grep "mysql.*-community.*"

### 2.2 安装 MySQL

>yum install mysql-community-server

启动 MySQL：

>systemctl start mysqld


查看 MySQL 的启动状态:

>systemctl status mysqld

```shell
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor preset: disabled)
   Active: active (running) since 五 2016-06-24 04:37:37 CST; 35min ago
 Main PID: 2888 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─2888 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

6月 24 04:37:36 localhost.localdomain systemd[1]: Starting MySQL Server...
6月 24 04:37:37 localhost.localdomain systemd[1]: Started MySQL Server.
```
### 2.3 开机启动

>systemctl enable mysqld
>systemctl daemon-reload

### 2.4 修改 root 本地登录密码

mysql 安装完成之后，在 `/var/log/mysqld.log` 文件中给 root 生成了一个默认密码。通过下面的方式找到 root 默认密码，然后登录 mysql 进行修改：

>grep 'temporary password' /var/log/mysqld.log

获取初始密码后登陆并修改密码：

> mysql -u root -p
> ALTER USER 'root'@'localhost' IDENTIFIED BY 'newPassword'; 

## 三、MySQL 密码检查策略

mysql5.7 默认安装了`密码安全检查插件（validate_password）`，默认密码检查策略要求密码必须包含：**大小写字母、数字和特殊符号，并且长度不能少于8位**。否则会提示`ERROR 1819 (HY000): Your password does not satisfy the current policy requirements`错误。

通过 msyql 环境变量可以查看密码策略的相关信息：

> show variables like '%password%';

![MySQL 密码检查策略](/images/posts/20180511010542698.png)

| 变量名| 数值 |
|:------------- |:-------------|
| validate_password_policy | 密码策略，默认为 MEDIUM 策略 |
| validate_password_dictionary_file | 密码策略文件，策略为 STRONG 才需要 |
| validate_password_length | 密码最少长度 |
| validate_password_mixed_case_count | 大小写字符长度，至少1个 |
| validate_password_number_count | 数字至少1个 |
| validate_password_special_char_count | 特殊字符至少1个  |

共有以下几种密码策略：

| 策略 | 检查规则 |
|:------------- |:-------------|
| 0 or LOW | Length |
| 1 or MEDIUM | Length; numeric, lowercase/uppercase, and special characters |
| 2 or STRONG | Length; numeric, lowercase/uppercase, and special characters; dictionary file |

在 `/etc/my.cnf` 文件添加 `validate_password_policy` 配置，指定密码策略

```shell
# 选择0（LOW），1（MEDIUM），2（STRONG）其中一种，选择 2 需要提供密码字典文件
validate_password_policy=0
```

如果不需要密码策略，添加 `my.cnf` 文件中添加如下配置禁用即可：

```
validate_password = off
```

重新启动 mysql 服务使配置生效：

>systemctl restart mysqld
