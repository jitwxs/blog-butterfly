---
title: MySQL 常用用户权限管理命令
categories:
  - 数据库
  - MySQL
abbrlink: 5aeb4f97
date: 2020-05-12 22:25:12
copyright_author: Jitwxs
---

## 一、用户

### 1.1 创建用户

```sql
-- 创建用户，并允许其在任何IP登陆
create user 'username'@'%' identified by 'password';

-- 创建用户，并允许其在任何主机登陆，不设置密码
create user 'username'@'%';

-- 创建用户，并仅允许使用jitwxs.cn域名的主机登陆
create user 'username'@'jitwxs.cn' identified by 'password';

-- 创建用户，并仅允许使用192.168.1.1的主机登陆
create user 'username'@'192.168.1.1' identified by 'password';

-- 创建用户，并仅允许使用192.168.1开头的主机登陆
create user 'username'@'192.168.1.%' identified by 'password';
```

说明：

- 密码可以为空，如为空，则可以免密登陆

- 如果主机位使用 `%`，表示允许任意地址的主机登陆

- 主机位可以使用域名或者 IP 地址，但是不允许既有数字又有字母

- 主机位中可以使用 `%` 进行通配，例如：`%.jitwxs.cn` 或 `192.168.1.%`

### 1.2 修改用户

> 一般在修改完密码后，需要手动执行命令，将配置刷新到内存：`flush privileges;`

```sql
-- 重新设置用户名和登陆IP
rename user 'old_username'@'old_ip_addr' to 'new_username'@'new_ip_addr';

-- 修改密码
set password for 'username'@'ip_addr'=Password('new_password');
```

### 1.3 删除用户

```sql
drop user 'username'@'ip_addr';
```

## 二、权限

> 一般在调整完权限后，需要手动执行命令，将配置刷新到内存：`flush privileges;`

### 2.1 查看权限

```sql
-- 查看用户所有权限
show grants for 'username'@'ip_addr';
```

### 2.2 权限授予

```sql
-- 授予用户所有库所有表的所有权限
grant all privileges on *.* to 'username'@'ip_addr';

-- 授予用户database1库所有表的所有权限
grant all privileges on `database1`.* to 'username'@'ip_addr';

-- 授予用户database1库table1表的所有权限
grant all privileges on `database1`.`table1` to 'username'@'ip_addr';

-- 授予用户database1库所有表的只读权限
grant select on `database1`.* to 'username'@'ip_addr';

-- 授予用户database1库table1表的插入、更新权限
grant insert,update on `database1`.`table1` to 'username'@'ip_addr';
```

### 2.3 权限回收

```sql
-- 回收用户所有权限
revoke all,grant option from 'username'@'ip_addr';

-- 回收用户database1库所有表的只读权限
revoke select ON `database1`.* FROM 'xiangsheng.wu'@'%';
```

### 2.4 授予什么就收回什么

基于 2.1 节，我们授予权限是可以有不同粒度的，且这些权限保存的位置也是不一样的，比如：

- 授予全局权限: `*.*`, 保存在: `mysql.user`

- 授予某个库的权限: `database1.*`, 保存在: `mysql.db`

- 授予某张表的权限: `database1.tabl1`, 保存在: `mysql.tables_priv`

因此，你要确保当初授予时是如何授予的，回收时候就要咋么回收。

例如授予时采用 `.*` 的方式授予了所有表的权限，那么回收的时候也必须使用 `.*` 的方式全部回收，而不能使用 `.table_xx` 的方式回收某一张表的权限，因为它们压根就不是一个粒度的配置。

### 2.5 附：常用权限列表

| 权 限                   | 作用范围             | 作 用                         |
| :---------------------- | :------------------- | :---------------------------- |
| all                     | 服务器               | 所有权限                      |
| select                  | 表、列               | 选择行                        |
| insert                  | 表、列               | 插入行                        |
| update                  | 表、列               | 更新行                        |
| delete                  | 表                   | 删除行                        |
| create                  | 数据库、表、索引     | 创建                          |
| drop                    | 数据库、表、视图     | 删除                          |
| reload                  | 服务器               | 允许使用flush语句             |
| shutdown                | 服务器               | 关闭服务                      |
| process                 | 服务器               | 查看线程信息                  |
| file                    | 服务器               | 文件操作                      |
| grant option            | 数据库、表、存储过程 | 授权                          |
| references              | 数据库、表           | 外键约束的父表                |
| index                   | 表                   | 创建/删除索引                 |
| alter                   | 表                   | 修改表结构                    |
| show databases          | 服务器               | 查看数据库名称                |
| super                   | 服务器               | 超级权限                      |
| create temporary tables | 表                   | 创建临时表                    |
| lock tables             | 数据库               | 锁表                          |
| execute                 | 存储过程             | 执行                          |
| replication client      | 服务器               | 允许查看主/从/二进制日志状态  |
| replication slave       | 服务器               | 主从复制                      |
| create view             | 视图                 | 创建视图                      |
| show view               | 视图                 | 查看视图                      |
| create routine          | 存储过程             | 创建存储过程                  |
| alter routine           | 存储过程             | 修改/删除存储过程             |
| create user             | 服务器               | 创建用户                      |
| event                   | 数据库               | 创建/更改/删除/查看事件       |
| trigger                 | 表                   | 触发器                        |
| create tablespace       | 服务器               | 创建/更改/删除表空间/日志文件 |
| proxy                   | 服务器               | 代理成为其它用户              |
| usage                   | 服务器               | 没有权限                      |

## 三、备份与恢复

### 3.1 备份

```sql
-- 备份database1库所有表结构+数据
mysqdump -u username database1 > database1.sql -p

-- 备份database1库所有表结构
mysqdump -u username -d database1 > database1.sql -p
```

### 3.2 恢复

```sql
-- 进入数据库
use database1;
-- 将备份文件恢复到数据库中
mysqdump -u username -d database1 < database1.sql -p
```
