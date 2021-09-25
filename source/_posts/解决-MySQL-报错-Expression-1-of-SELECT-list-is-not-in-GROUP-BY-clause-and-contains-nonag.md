---
title: >-
  解决 MySQL 报错 Expression #1 of SELECT list is not in GROUP BY clause and
  contains nonag...
categories:
  - 数据库
  - MySQL
abbrlink: b6db8e4b
date: 2019-02-25 22:40:00
---

## 一、问题描述

运行 sql 后报错：

```
Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated 
column ‘day_offset’ which is not functionally dependent on columns in GROUP BY clause; 
this is incompatible with sql_mode=only_full_group_by...
```

MySQL 5.7.5 及以上功能依赖检测功能。如果启用了`ONLY_FULL_GROUP_BY SQL` 模式（默认情况下），MySQL将拒绝选择列表，HAVING 条件或 ORDER BY 列表的查询引用在 GROUP BY 子句中既未命名的非集合列，也不在功能上依赖于它们。（5.7.5之前，MySQL没有检测到功能依赖关系，默认情况下不启用ONLY_FULL_GROUP_BY。）

## 二、解决方法

### 2.1 临时解决

执行 SQL：

```sql
select @@global.sql_mode
```

得到查询结果：

```
ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,
NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,
NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

可以看到我们的 `sql_mode` 中含有 `ONLY_FULL_GROUP_BY`，因此只要将它去掉即可。

执行命令：

```sql
SET sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```

**说明：** 需要注意的是，该修改仅在该次（窗口）有效，只是临时修改。

### 2.2 永久解决

#### 2.2.1 Windows 用户

编辑 mysql 配置文件 `my.ini`，在尾部添加以下内容，重新启动 mysql 即可：

```ini
[mysqld] 
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```
#### 2.2.2 Mac/Linux 用户

需要编辑 `/etc/my.cnf` 文件，在尾部添加以下内容，重新启动 mysql 即可：

```ini
[mysqld] 
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

如果该文件不存在，可以新建一个，以下为初始内容：

```ini my.cnf
# Example MySQL config file for medium systems.  
  #  
  # This is for a system with little memory (32M - 64M) where MySQL plays  
  # an important part, or systems up to 128M where MySQL is used together with  
  # other programs (such as a web server)  
  #  
  # MySQL programs look for option files in a set of  
  # locations which depend on the deployment platform.  
  # You can copy this option file to one of those  
  # locations. For information about these locations, see:  
  # http://dev.mysql.com/doc/mysql/en/option-files.html  
  #  
  # In this file, you can use all long options that a program supports.  
  # If you want to know which options a program supports, run the program  
  # with the "--help" option.  
  # The following options will be passed to all MySQL clients  
  [client]
  default-character-set=utf8
  #password   = your_password  
  port        = 3306  
  socket      = /tmp/mysql.sock   
  # Here follows entries for some specific programs  
  # The MySQL server  
  [mysqld]
  character-set-server=utf8
  init_connect='SET NAMES utf8
  port        = 3306  
  socket      = /tmp/mysql.sock  
  skip-external-locking  
  key_buffer_size = 16M  
  max_allowed_packet = 1M  
  table_open_cache = 64  
  sort_buffer_size = 512K  
  net_buffer_length = 8K  
  read_buffer_size = 256K  
  read_rnd_buffer_size = 512K  
  myisam_sort_buffer_size = 8M  
  character-set-server=utf8  
  init_connect='SET NAMES utf8' 
# Don't listen on a TCP/IP port at all. This can be a security enhancement,  
# if all processes that need to connect to mysqld run on the same host.  
# All interaction with mysqld must be made via Unix sockets or named pipes.  
# Note that using this option without enabling named pipes on Windows  
# (via the "enable-named-pipe" option) will render mysqld useless!  
#   
#skip-networking  
  
  # Replication Master Server (default)  
  # binary logging is required for replication  
  log-bin=mysql-bin  
    
    # binary logging format - mixed recommended  
    binlog_format=mixed  
      
      # required unique id between 1 and 2^32 - 1  
      # defaults to 1 if master-host is not set  
      # but will not function as a master if omitted  
      server-id   = 1  
        
    # Replication Slave (comment out master section to use this)  
    #  
    # To configure this host as a replication slave, you can choose between  
    # two methods :  
    #  
    # 1) Use the CHANGE MASTER TO command (fully described in our manual) -  
    #    the syntax is:  
    #  
    #    CHANGE MASTER TO MASTER_HOST=<host>, MASTER_PORT=<port>,  
    #    MASTER_USER=<user>, MASTER_PASSWORD=<password> ;  
    #  
    #    where you replace <host>, <user>, <password> by quoted strings and  
    #    <port> by the master's port number (3306 by default).  
    #  
    #    Example:  
    #  
    #    CHANGE MASTER TO MASTER_HOST='125.564.12.1', MASTER_PORT=3306,  
    #    MASTER_USER='joe', MASTER_PASSWORD='secret';  
    #  
    # OR  
    #  
    # 2) Set the variables below. However, in case you choose this method, then  
    #    start replication for the first time (even unsuccessfully, for example  
    #    if you mistyped the password in master-password and the slave fails to  
    #    connect), the slave will create a master.info file, and any later  
    #    change in this file to the variables' values below will be ignored and  
    #    overridden by the content of the master.info file, unless you shutdown  
    #    the slave server, delete master.info and restart the slaver server.  
    #    For that reason, you may want to leave the lines below untouched  
    #    (commented) and instead use CHANGE MASTER TO (see above)  
    #  
    # required unique id between 2 and 2^32 - 1  
    # (and different from the master)  
    # defaults to 2 if master-host is set  
    # but will not function as a slave if omitted  
    #server-id       = 2  
    #  
    # The replication master for this slave - required  
    #master-host     =   <hostname>  
    #  
    # The username the slave will use for authentication when connecting  
    # to the master - required  
    #master-user     =   <username>  
    #  
    # The password the slave will authenticate with when connecting to  
    # the master - required  
    #master-password =   <password>  
    #  
    # The port the master is listening on.  
    # optional - defaults to 3306  
    #master-port     =  <port>  
    #  
    # binary logging - not required for slaves, but recommended  
    #log-bin=mysql-bin  
      
    # Uncomment the following if you are using InnoDB tables  
    #innodb_data_home_dir = /usr/local/mysql/data  
    #innodb_data_file_path = ibdata1:10M:autoextend  
    #innodb_log_group_home_dir = /usr/local/mysql/data  
    # You can set .._buffer_pool_size up to 50 - 80 %  
    # of RAM but beware of setting memory usage too high  
    #innodb_buffer_pool_size = 16M  
    #innodb_additional_mem_pool_size = 2M  
    # Set .._log_file_size to 25 % of buffer pool size  
    #innodb_log_file_size = 5M  
    #innodb_log_buffer_size = 8M  
    #innodb_flush_log_at_trx_commit = 1  
    #innodb_lock_wait_timeout = 50  
        
    [mysqldump]  
    quick  
    max_allowed_packet = 16M  
    
    [mysql]  
    no-auto-rehash  
    # Remove the next comment character if you are not familiar with SQL  
    #safe-updates  
    default-character-set=utf8   
    
    [myisamchk]  
    key_buffer_size = 20M  
    sort_buffer_size = 20M  
    read_buffer = 2M  
    write_buffer = 2M  
    
    [mysqlhotcopy]  
    interactive-timeout
```
