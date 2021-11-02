---
title: 详解 MySQL下的行表锁
categories:
  - Database
  - MySQL
abbrlink: bc240583
date: 2019-03-06 00:11:09
---

>阅读本文前，请务必查看先导文章：[数据库基础理论](/1c73b076.html)
>
>阅读本文后，你可以查看延申文章：[[MySQL] 行级锁SELECT ... LOCK IN SHARE MODE 和 SELECT ... FOR UPDATE](https://blog.csdn.net/u012099869/article/details/52778728)

## 一、MyISAM 引擎

`MyISAM` 是 MySQL 5.1 之前的默认搜索引擎，我们都知道，MyISAM 采用**表锁**，即操作一条记录也会锁住整张表。适合做主要查询、非事务的表的引擎。下面演示下 MyISAM 引擎下的读锁与写锁。

首先创建两张 myisam 引擎的表，并准备一些数据：

```sql
CREATE TABLE mylock(
id int not null PRIMARY KEY auto_increment,
name VARCHAR(16)
)ENGINE myisam;

CREATE TABLE course(
id int not null PRIMARY KEY auto_increment,
name VARCHAR(16)
)ENGINE myisam;

INSERT INTO mylock(name) VALUES ('a');
INSERT INTO mylock(name) VALUES ('b');
INSERT INTO mylock(name) VALUES ('b');
INSERT INTO mylock(name) VALUES ('c');
INSERT INTO mylock(name) VALUES ('e');

INSERT INTO course(name) VALUES ('english');
INSERT INTO course(name) VALUES ('math');
INSERT INTO course(name) VALUES ('chinese');
INSERT INTO course(name) VALUES ('pe');
INSERT INTO course(name) VALUES ('music');
```

开启两个会话窗口，事先说明下文截图中如果出现两个窗口，则上方为 session1，下方为 session 2。

### 1.1 读锁

首先演示下 MyISAM 的读锁效果，在 session1 中执行 `lock table mylock read;` 开启 mylock 表的读锁。

**情况一：** session1 和 session2 都可以自由的SELECT mylock 表的内容。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305220127729.png)

**情况二：** session1 不可以 SELECT 其他没有锁住的表，session2 可以。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305220748269.png)

**情况三：** session1 无法对 mylock 表做 INSERT、UPDATE、DELETE 操作。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305221118321.png)

**情况四：** session2 对 mylock 表做 INSERT、UPDATE、DELETE 操作，会阻塞，直到 session1 释放锁。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305221310900.png)

上图中，首先在 session2 中执行 INSERT 语句，执行后发现被阻塞，然后再 session1 中通过 `unlock tables` 解锁，然后 session2 返回执行结果（通过 session2 的执行时间可以看出来）。

### 1.2 写锁

下面演示下 MyISAM 的写锁效果，在 session1 中执行 `lock table mylock write;` 开启 mylock 表的写锁。

**情况一：** session1 无法 SELECT 其他没有锁定的表，session2 可以。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305221819183.png)

**情况二：** session1 可以 SELECT、 INSERT、UPDATE、DELETE mylock 表，session2 不可以，会阻塞，直到 sesson1 释放掉锁。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305222247472.png)

### 1.3 总结

`MyISAM` 在执行**查询**语句前，会自动给涉及的所有表加**读锁**，在执行**插入、删除、修改**操作前，会自动给涉及的表加**写锁**。

对 MyISAM 表的 **读操作，不会阻塞其他进程对同一表的读请求，但会阻塞对同一表的写请求。只有当读锁释放后，才会执行其他进程的写操作。** 这就是`表共享读锁（Table Read Lock）`。

对 MyISAM 表的**写操作，会阻塞其他进程对同一表的读和写操作，只有当写锁释放后，才会执行其他进程的读写操作**。这就`表独占写锁（Table Write Lock）`。

简而言之，**读锁会阻塞写，但不会阻塞读，而写锁会把读和写都阻塞**。

### 1.4 表锁分析

介绍下两个命令，可以帮助我们找到系统中存在的表锁，及具体是哪张表出现了表锁。

（1） `show open tables;`

这个命令帮助我们找到哪张表有表锁，执行后 `In_use` 列为 1 则为有表锁。

（2）`show status like 'table%';`

```sql
mysql> show status like 'table%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Table_locks_immediate      | 118   |
| Table_locks_waited         | 0     |
| Table_open_cache_hits      | 8     |
| Table_open_cache_misses    | 2     |
| Table_open_cache_overflows | 0     |
+----------------------------+-------+
5 rows in set (0.00 sec)
```

这里有两个状态变量记录 MySQL 内部表级锁定的情况，两个变量说明如下：

- `Table_locks_immediate`：产生表级锁定的次数，表示可以立即获取锁的査询次数，每立即获取锁值加1；

- `Table_locks_waited`：出现表级锁定争用而发生等待的次数（不能立即获取锁的次数，每等待一次锁值加1），**此值高则说明存在着较严重的表级锁争用情况**。

## 二、InnoDB 引擎

InnoDB 引擎是 MySQL 5.1 之后的表默认引擎，相较于 MyISAM 的表锁，**InnoDB 采用行锁，而且支持事务操作**。

创建 i_mylock 表，并准备一些数据：

```sql
CREATE TABLE i_mylock(
id int not null PRIMARY KEY auto_increment,
name VARCHAR(16)
)ENGINE innodb;

INSERT INTO i_mylock(name) VALUES ('a');
INSERT INTO i_mylock(name) VALUES ('b');
INSERT INTO i_mylock(name) VALUES ('b');
INSERT INTO i_mylock(name) VALUES ('c');
INSERT INTO i_mylock(name) VALUES ('e');
```

首先 MySQL 默认开启了自动提交功能，因此当我们进行测试时，要么关闭掉 MySQL 的自动提交功能（`set autocommit = 0`），要么显式声明事务（`begin/commit/rollback`），本文采用显式声明事务的方式。

>我们知道 InnoDB 的行锁默认在执行 INSERT、UPDATE、DELETE 语句时会**对影响行加行锁**。为了简便用户操作，当我们按下执行回车键到执行成功的这段时间，MySQL 会自动帮助我们开启事务，并提交/回滚事务，来确保这段执行时间内影响行是被锁定的，这就是 MySQL 的`自动提交功能`。

另外 MySQL 的默认隔离级别为 **可重复读 Repeated Read**，下文例子也均在该隔离级别下演示。

### 2.1 行锁

下面演示对 `id=1` 的行同时进行 UPDATE 操作。session1 首先开启事务，并对该行进行修改，在 session1 提交事务之前，session2 也对该行进行修改，但是因为存在行锁所以操作被**阻塞**掉。session1 提交事务后，session2 恢复执行。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305230205949.png)

### 2.2 锁住指定行

默认情况下 SELECT 语句是不会触发行锁的，例如下图中 session1 开启事务后，session2 仍然能够查询到 session1 查询的同一行记录。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305231058742.png)

如果我们想要实现锁住想要的行，可以通过给行锁加**意向排它锁**来实现，即 `SELECT ... FOR UPDATE`，如下图：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305232530937.png)

### 2.3 间隙锁

我们用范围条件而不是相等条件检索数据时，InnoDB 会给符合条件的已有数据记录的索引项加锁;对于键值在条件范围内但并不存在的记录,叫做“`间隙(GAP)`”，InnoDB **也会对这个“间隙”加锁**,这种锁机制就是所谓的`间隙锁(Next-Key锁)`。

下面演示一个间隙锁的例子，session1 中使用 IX 锁锁住了表中 id >=5 的记录，虽然影响条数只有一条，但是 InnoDB 却会对这个不存在的间隙加锁，加锁范围为 **(5, ∞]** ，因此当 session2 插入一条记录为 6 的数据会被阻塞。

![间隙锁](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305233414260.png)

间隙锁的弱点是当锁定一个范围键值之后，即使某些不存在的键值也会被无辜的锁定，而造成在锁定的时候**无
法插入/修改锁定键值范围内的任何数据**，在某些场景下这可能会对性能造成很大的危害，使用需要十分谨慎。

### 2.4 行锁失效

当我们在使用 InnoDB 的行锁的时候，需要十分注意的一个问题是**行锁失效**。MySQL InnoDB 的行锁是通过**索引上的索引项**来实现的，因此**只有通过索引条件检索数据时**，才会触发行锁，**否则 InnoDB 会使用表锁**。

至于什么情况下会导致索引失效，在本文开头的前导文章链接中已经叙述过了，就不再赘述了，下面直接给出行锁失效引发表锁的例子。

![行锁失效](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201903/20190305234849764.png)

### 2.5 行锁分析

这里也介绍一个用于行锁的分析命令：`show status like 'innodb_row_lock%';`

```sql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+--------+
| Variable_name                 | Value  |
+-------------------------------+--------+
| Innodb_row_lock_current_waits | 0      |
| Innodb_row_lock_time          | 174133 |
| Innodb_row_lock_time_avg      | 24876  |
| Innodb_row_lock_time_max      | 51045  |
| Innodb_row_lock_waits         | 7      |
+-------------------------------+--------+
5 rows in set (0.00 sec)
```

对各个状态量的说明如下:

|变量|描述|
|:--|:--|
|Innodb_row_lock_current_waits |当前正在等待锁定的数量|
|Innodb_row_lock_time          |从系统启动到现在锁定总时间长度|
|Innodb_row_lock_time_avg      |每次等待所花平均时间|
|Innodb_row_lock_time_max      |从系统启动到现在等待最常的一次所花的时间|
|Innodb_row_lock_waits|系统启动后到现在总共等待的次数|

对于这5个状态变量,比较重要的主要是 `Innodb_row_lock_time`(等待总时长)、`Innodb_row_lock_time_avg`(等待平均时长)、`Innodb_row_lock_waits`（等待总次数）这三项。

尤其是当等待次数很高，而且每次等待时长也不小的时候，我们就需要分析系统中为什么会有如此多的等待，然后根据分析结果着手指定优化计划。
