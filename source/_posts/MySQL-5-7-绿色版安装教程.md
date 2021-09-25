---
title: MySQL 5.7 绿色版安装教程
categories: 
  - 数据库
  - MySQL
abbrlink: f5cfbb3c
date: 2017-11-29 01:36:57
---

## 一、下载安装包

首先[前往官网](https://dev.mysql.com/downloads/mysql/)下载 MySQL，也可以直接下载 [mysql-5.7.20-winx64.zip](https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.20-winx64.zip)

## 二、配置 MySQL

（1）下载完毕后（我此时版本为5.7.20），将压缩包解压，将解压后的整个文件夹放在你要放的位置，这里我放在 D 盘根目录下，即路径为：

>D:\mysql-5.7.20-winx64

（2）在该路径下，在其中新建 `my.ini` 文件（注意：**文件名后缀为ini**），并在其中编辑以下代码（注意：代码中的**两个路径要替换成自己实际的路径**）：

```ini my.ini
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8 
[mysqld]
#设置3306端口
port = 3306 
# 设置mysql的安装目录
basedir=D:\mysql-5.7.20-winx64
# 设置mysql数据库的数据的存放目录
datadir=D:\mysql-5.7.20-winx64\data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB 
```

注意 `datadir` 必须是名为 `data` 的文件夹，且此文件夹在安装前**要确保不能存在**。

（3）不要忘记将 `D:\mysql-5.7.20-winx64\bin` 目录加入系统 `PATH 环境变量`中。

## 三、开始安装

（1）以管理员权限打开 `cmd`，进入 `D:\mysql-5.7.20-winx64\bin` 目录，依次执行以下命令：

```shell
mysqld install

mysqld --initialize --console
```

（2）执行结束后，记录下 屏幕中 `root@localhost: ` 后面的字符串，此为生成的随机密码，登录用户名默认为 root。

特别注意，如果屏幕中并没有生成随机密码，那么密码其实被保存在了**mysql目录下的 data 子目录**中一个以 `err`为后缀的文件中，使用记事本打开就可以看到密码了。

（3）打开任务管理器->服务，启动 `MySQL` 服务：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201711/20171129012319909.png)

## 四、完成安装

（1）在 `cmd` 中执行 `mysql -u root -p` 运行 mysql，输入之前记录的密码，就可以成功登陆了。

（2）登陆成功后输入 `SET PASSWORD = PASSWORD('你的新密码');` 来修改初始密码。

## 五、重置MySQL密码

### 5.1 方法一

1.在 `my.ini` 的 `[mysqld]` 字段加入：

```init
skip-grant-tables
```

2.重启mysql服务，这时的mysql不需要密码即可登录数据库，然后进入 mysql：

```shell
mysql>use mysql;
mysql>set password=password('新密码') WHERE User='root';
mysql>flush privileges;
```

3.运行之后最后去掉 `my.ini` 中的 `skip-grant-tables`，重启 mysqld 即可。

### 5.2 方法二

1.停止 mysql 服务，打开命令行窗口，在 bin 目录下使用 `mysqld-nt.exe` 启动，在命令行窗口执行: `mysqld-nt --skip-grant-tables`

2.然后另外打开一个命令行窗口，登录 mysql，此时无需输入 mysql 密码即可进入。

3.按以上方法修改好密码后,关闭命令行运行 mysql 的那个窗口，此时即关闭了 mysql，如果发现 mysql 仍在运行的话可以结束掉对应进程来关闭。

4.启动 mysql 服务。
