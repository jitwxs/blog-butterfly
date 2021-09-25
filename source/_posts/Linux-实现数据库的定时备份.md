---
title: Linux 实现数据库的定时备份
tags:
  - Linux
  - 数据库
categories: Linux
abbrlink: a804ec35
date: 2017-10-30 23:10:27
---

在项目中，数据往往是十分重要的，这就要求我们对数据库进行实时备份。幸运的是，Linux 支持这样的功能。本文将讲述如何定时实现数据库的备份。

## 一、创建备份文件夹

我们以备份到 `/home` 路径下为例，创建 backup 文件夹：

```shell
[root@iZuf643t8c5urcnhm494emZ ~]# cd /home
[root@iZuf643t8c5urcnhm494emZ home]# mkdir backup
[root@iZuf643t8c5urcnhm494emZ home]# ls
backup
```

## 二、编写备份脚本程序

```bash
#!/usr/bin/bash

mysqldump -uUserName -pPassWord DatabaseName > /home/backup/DatabaseName_$(date +%Y%m%d_%H%M%S).sql

mysqldump -uUserName -pPassWord DatabaseName | gzip > /home/backup/DatabaseName_$(date +%Y%m%d_%H%M%S).sql.gz
```

上面这段脚本中第一行是备份了 .sql 文件，第二行是生成压缩包，根据自己需求选择。其中：

- UserName 数据库用户名

- PassWord 数据库密码

- DatabaseName 数据库名

进入 `backup` 文件夹，编写用来定时执行的脚本程序（脚本名没有要求），以备份 `express_manager_system` 数据库为例，给出我的示例：

```shell
[root@iZuf643t8c5urcnhm494emZ home]# cd backup/
[root@iZuf643t8c5urcnhm494emZ backup]# vim bkexpress_manager_system.sh
```

```bash bkexpress_manager_system.sh
#!/usr/bin/bash

mysqldump -uroot -proot express_manager_system > /home/backup/express_manager_system_$(date +%Y%m%d_%H%M%S).sql

mysqldump -uroot -proot express_manager_system | gzip > /home/backup/express_manager_system_$(date +%Y%m%d_%H%M%S).sql.gz
```

为脚本添加可执行权限：

```shell
[root@iZuf643t8c5urcnhm494emZ backup]# chmod +x bkexpress_manager_system.sh
```

## 三、定时执行脚本程序

这里我们用到 Linux 的 `crontab` 命令，首先检测是否安装：

```shell
[root@iZuf643t8c5urcnhm494emZ backup]# crontab -l
-bash: /usr/bin/crontab: 没有那个文件或目录
```

我使用的是 CentOS 系统，执行以下命令安装：

```shell
[root@iZuf643t8c5urcnhm494emZ backup]# yum install -y crontabs
```

`crontab`我们使用以下两个参数：

- -l：列出该用户的所有设置

- -e：编辑该用户的设置

我们执行 `crontab -e` 命令，添加一行下面的代码，来定时执行我们脚本的命令：

```
0 8 * * * /home/backup/bkexpress_manager_system.sh
```

我这行代码的意思是每天8点执行一次脚本。

关于这里配置文件每一列的具体含义，可以参考[crontab命令](http://man.linuxde.net/crontab)，这里不再赘述。

```shell
[root@iZuf643t8c5urcnhm494emZ backup]# ls
bkexpress_manager_system.sh
express_manager_system_20171029_080001.sql
express_manager_system_20171029_080001.sql.gz
express_manager_system_20171030_080001.sql
express_manager_system_20171030_080001.sql.gz
```

这个定时任务是我几天前创建的，这里已经有这几天生成的备份文件了，至此数据库的定时备份就已经完成了。
