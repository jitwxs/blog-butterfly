---
title: Redis 初探（3）——Redis 的数据类型
categories:
  - 数据库
  - Redis
abbrlink: '63e66407'
date: 2018-03-02 01:23:21
copyright_author: Jitwxs
---

在[《Redis 初探（1）——Redis 的安装》](/e331e26a.html)中，我们说过，Redis支持以下五种数据类型，本章进行详解：

- String类型
- List类型
- Set类型
- SortedSet类型
-  Hash类型

|存储极限|大小|
|:------------- |:------------- |
| String类型的value大小 | 512M |
| Hash类型key的键值对个数 | 4294967295 |
| List类型key个数 | 4294967295 |
| Set/SortedSet类型key个数 | 4294967295 |

## 一、String（字符串）类型

在前面两章中，我们存储的都是 String 类型。该类型增加和删除一个键值对十分简单，如下：

| 含义 | 方法 |
|:------------- |:------------- |
| 获取值 | get key |
| 添加一个键值对 | set key value |
| 获取并重置一个键值对 | getset key value |
| 删除一个键值对 | del key |

前面两个我们之前都使用过了，`getset key value` 有点陌生，它是先获取 key 的 value，然后将该 key 修改为输入的 value，相当于将 `get key` 和 `set key value`这两步合为了一步，看看下面这个例子你就明白了：

```shell
127.0.0.1:6379> getset age 10
"20"
127.0.0.1:6379> get age
"10"
```

删除一个键值对也很简单，如下：

```shell
127.0.0.1:6379> del age
(integer) 1
127.0.0.1:6379> get age
(nil)
```

字符串也可以进行数值操作（Redis内部自动将value转换为数值型），方法如下：

| 含义 | 方法 |
|:------------- |:------------- |
| 值加一 | incr key |
| 值减一 | decr key |
| 值加n | incrby key n |
| 值减n | decrby key n |

**注意：** 如果 key 值不存在，当作0处理；如果 value 值无法转换为整型时，会返回错误信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/2018022800472165.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180228005242867.png)

介绍一下字符串拼接方法，如下：

| 含义 | 方法 |
|:------------- |:------------- |
| 拼接 | append key |
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180228005805506.png)

## 二、Hash（散列）类型

散列，即 Hash，Redis 中的 Hash 类型可以看成**Map集合**。Hash 类型的**每一个 key 的 value 对应于一个 Map，该Map包含多个键值对的数据**，如下图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20180301220938319.png)

### 2.1 赋值

| 含义 | 方法 |
|:------------- |:------------- |
| 为指定key设置一个键值对| hset key field value |
| 为指定key设置多个键值对| hmset key field value[field2 value2...] |

```shell
127.0.0.1:6379> hset myHash name jitwxs
(integer) 1
127.0.0.1:6379> hmset myHash age 20 sex male
(integer) 2
127.0.0.1:6379> 
```

### 2.2 取值

| 含义 | 方法 |
|:------------- |:------------- |
| 返回指定 key 中 field 的值 | hget key field |
| 返回指定 key 中多个 field 的值 | hmget key filed[field2...] |
| 返回指定 key 中所有 field-value | hgetall key |

```shell
127.0.0.1:6379> hget myHash name
"jitwxs"
127.0.0.1:6379> hmget myHash age sex
1) "20"
2) "male"
127.0.0.1:6379> hgetall myHash
1) "name"
2) "jitwxs"
3) "age"
4) "20"
5) "sex"
6) "male"
```

### 2.3 删除

| 含义 | 方法 |
|:------------- |:------------- |
| 删除指定 key 一个或多个 field | hdel key field[field2...] |
| 清空 Hash | del key |

```shell
127.0.0.1:6379> hdel myHash name
(integer) 1
127.0.0.1:6379> hdel myHash age sex
(integer) 2
```

```shell
127.0.0.1:6379> hmset myHash name jitwxs age 20 sex male
OK
127.0.0.1:6379> del myHash
(integer) 1
127.0.0.1:6379> hgetall myHash
(empty list or set)
```

### 2.4 扩展命令

| 含义 | 方法 |
|:------------- |:------------- |
| 判断指定 key 中 field 是否存在 | hexists key field |
| 返回指定 key 中 field 的数量 | hlen key |
| 获取指定 key 中所有的 field | hkeys key |
| 获取指定 key 中所有的 value | hvals key |

```shell
127.0.0.1:6379> hmset myHash name jitwxs age 20 sex male
OK
127.0.0.1:6379> hexists myHash name
(integer) 1
127.0.0.1:6379> hexists myHash unknown
(integer) 0
127.0.0.1:6379> hlen myHash
(integer) 3
127.0.0.1:6379> hkeys myHash
1) "name"
2) "age"
3) "sex"
127.0.0.1:6379> hvals myHash
1) "jitwxs"
2) "20"
3) "male"
```

## 三、List 类型

Redis 的 List 类型有点像 Java 中的 LinkList，内部实现是一个`双向链表`，双向链表的知识点参考文章：[《数据结构 第二章 线性表》](/abd3f641.html)。

### 3.1 添加

| 含义 | 方法 |
|:------------- |:------------- |
| 从左端添加多个value | lpush key value[value2...] |
| 从右端添加多个value | rpush key value[value2...] |

**注：** 如果 key 不存在会先创建 key，然后添加。

```shell
127.0.0.1:6379> lpush myList a b c
(integer) 3
127.0.0.1:6379> lrange myList 0 -1
1) "c"
2) "b"
3) "a"
127.0.0.1:6379> rpush myList2 1 2 3
(integer) 3
127.0.0.1:6379> lrange myList2 0 -1
1) "1"
2) "2"
3) "3"
```

### 3.2 查看

| 含义 | 方法 |
|:------------- |:------------- |
| 获取 list 从 start 到 end 的值 | lrange key start end |
| 获取 list 中元素数量 | llen key |

因为 List 内部是一个双向链表，因此链表**首元素下标为0，尾元素下标为-1**，因此查看所有元素即：`lrange key 0 -1`。

### 3.3 删除

| 含义 | 方法 |
|:------------- |:------------- |
| 返回并弹出左端元素 | lpop key |
| 返回并弹出右端元素 | rpop key |

**注：** 如果 key 不存在，返回nil。

```shell
127.0.0.1:6379> lpush myList a b c
(integer) 3
127.0.0.1:6379> lrange myList 0 -1
1) "c"
2) "b"
3) "a"
127.0.0.1:6379> lpop myList
"c"
127.0.0.1:6379> rpop myList
"a"
```

### 3.4 扩展命令

#### 3.4.1 添加前检查 key 的存在性

| 含义 | 方法 |
|:------------- |:------------- |
| 从左端添加多个 value | lpushx key value[value2...] |
| 从右端添加多个 value | rpushx key value[value2...] |

这两个方法加了 `x` 的和之前不加 `x` 的不同之处是：如果 key 不存在，将**不进行插入**。

```shell
127.0.0.1:6379> del myList
(integer) 1
127.0.0.1:6379> lpushx myList a b c
(integer) 0
127.0.0.1:6379> lrange myList 0 -1
(empty list or set)
```

#### 3.4.2 根据 value 删除

| 含义 | 方法 |
|:------------- |:------------- |
| 删除 count 个值为 value 的元素| lrem key count value |

若 count > 0，则从左到右删除：

```shell
127.0.0.1:6379> rpush myList 1 2 1 3 5 1
(integer) 6
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "2"
3) "1"
4) "3"
5) "5"
6) "1"
127.0.0.1:6379> lrem myList 2 1
(integer) 2
127.0.0.1:6379> lrange myList 0 -1
1) "2"
2) "3"
3) "5"
4) "1"
```

若 count < 0，则从右向左删除：

```shell
127.0.0.1:6379> rpush myList 1 2 1 3 5 1
(integer) 6
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "2"
3) "1"
4) "3"
5) "5"
6) "1"
127.0.0.1:6379> lrem myList -2 1
(integer) 2
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "2"
3) "3"
4) "5"
```

若 count = 0，删除所有：

```shell
127.0.0.1:6379> rpush myList 1 2 1 3 5 1
(integer) 6
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "2"
3) "1"
4) "3"
5) "5"
6) "1"
127.0.0.1:6379> lrem myList 0 1
(integer) 3
127.0.0.1:6379> lrange myList 0 -1
1) "2"
2) "3"
3) "5"
```

#### 3.4.3 根据下标设置 value

| 含义 | 方法 |
|:------------- |:------------- |
| 设置下标为 index 的元素值。0代表最左边元素，-1代表最右边元素，下标不存在时抛出异常。 | lset key index value |

```shell
127.0.0.1:6379> rpush myList 1 2 3
(integer) 3
127.0.0.1:6379> lset myList 1 5
OK
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "5"
3) "3"
127.0.0.1:6379> lset myList 3 5
(error) ERR index out of range
```

#### 3.4.4 相对于某元素插入 value

| 含义 | 方法 |
|:------------- |:------------- |
| 在 pivot 元素前插入value | linsert key before pivot value |
| 在p ivot 元素后插入value | linsert key after pivot value |

注：如果 pivot 不存在，不插入。

```shell
127.0.0.1:6379> rpush myList 1 2 3
(integer) 3
127.0.0.1:6379> linsert myList before 2 a
(integer) 4
127.0.0.1:6379> linsert myList after 2 b
(integer) 5
127.0.0.1:6379> lrange myList 0 -1
1) "1"
2) "a"
3) "2"
4) "b"
5) "3"
```

#### 3.4.5 将链表 A 右边元素移出并添加到链表B  左边

| 含义 | 方法 |
|:------------- |:------------- |
| 将链表 A 右边元素移出并添加到链表 B 左边 | rpoplpush listA listB |

```shell
127.0.0.1:6379> rpush myListA 1 2 3
(integer) 3
127.0.0.1:6379> rpush myListB a b c
(integer) 3
127.0.0.1:6379> rpoplpush myListA myListB
"3"
127.0.0.1:6379> lrange myListA 0 -1
1) "1"
2) "2"
127.0.0.1:6379> lrange myListB 0 -1
1) "3"
2) "a"
3) "b"
4) "c"
127.0.0.1:6379> rpoplpush myListA myListA
"2"
127.0.0.1:6379> lrange myListA 0 -1
1) "2"
2) "1"

```

## 四、Set 类型

Redis 的 Set 类型和 Java 中的 Set 类型一样，它具有两个重要的特点：`无序性`和`唯一性`，具体不再赘述。

### 4.1 基本操作

| 含义 | 方法 |
|:------------- |:------------- |
| 向 set 中添加成员，如果成员已存在，不再添加 | sadd key member[member2...] |
| 向 set 中删除成员 | srem key member[member2...] |
| 获取 set 中所有成员 | smembers key |
| 判断指定成员是否存在于 set 中 | sismember key member|

```shell
127.0.0.1:6379> sadd mySet 1 2 3
(integer) 3
127.0.0.1:6379> smembers mySet
1) "1"
2) "2"
3) "3"
127.0.0.1:6379> srem mySet 2 3 5
(integer) 2
127.0.0.1:6379> smembers mySet
1) "1"
127.0.0.1:6379> sismember mySet 1
(integer) 1
127.0.0.1:6379> sismember mySet 2
(integer) 0

```

### 4.2 集合操作

| 含义 | 方法 |
|:------------- |:------------- |
| 集合的差集 | sdiff key1 key2[key3...] |
| 集合的交集 | sinter key1 key2[key3...] |
| 集合的并集 | sunion key1 key2[key3...]|

```shell
127.0.0.1:6379> sadd mySet1 a b c 1
(integer) 4
127.0.0.1:6379> sadd mySet2 1 2 3 b
(integer) 4
127.0.0.1:6379> sdiff mySet1 mySet2
1) "a"
2) "c"
127.0.0.1:6379> sdiff mySet2 mySet1
1) "2"
2) "3"
127.0.0.1:6379> sinter mySet1 mySet2
1) "1"
2) "b"
127.0.0.1:6379> sunion mySet1 mySet2
1) "c"
2) "1"
3) "b"
4) "2"
5) "a"
6) "3"
```

### 4.3 扩展命令

| 含义 | 方法 |
|:------------- |:------------- |
| 求 set 中成员数量 | scard key |
| 随即返回一个成员 | srandmember key |
| 将多个集合的差集存储在 desc 中 | sdiffstore desc key1 key2[key3...]|
| 将多个集合的交集存储在 desc 中 | sinterstore desc key1 key2[key3...]|
| 将多个集合的并集存储在 desc 中 | sunionstore desc key1 key2[key3...]|

```shell
127.0.0.1:6379> smembers mySet1
1) "a"
2) "c"
3) "1"
4) "b"
127.0.0.1:6379> smembers mySet2
1) "3"
2) "b"
3) "2"
4) "1"
127.0.0.1:6379> scard mySet1
(integer) 4
127.0.0.1:6379> srandmember mySet1
"c"
127.0.0.1:6379> sdiffstore mySet3 mySet1 mySet2
(integer) 2
127.0.0.1:6379> smembers mySet3
1) "a"
2) "c"
```

## 五、SortedSet 类型

SortedSet 和 Set 的区别是，SortedSet 中每一个成员都有一个 `score（分数）`与之关联，Redis 通过 score 来为集合中的元素进行排序（默认为升序）。

### 5.1 添加/获取元素

| 含义 | 方法 |
|:------------- |:------------- |
| 添加成员。如果成员存在，会用新的 score 替代原有的 score，返回值是新加入到集合中的成员个数 | zadd key score member[score2 member2... ] |
| 获取指定成员的 score | zscore key member |
| 获取 key 中成员个数 | scard key |
| 获取集合中下标从 start 到 end 的成员，[withscores]表明返回的成员包含其 score | zrange key start end[withscores] |
| 上面方法的反转| zrevrange key start end[withscores]|

```shell
127.0.0.1:6379> zadd mySort 82 wangnima 100 cat 33 dog 43 jitwxs 80 zhouyang 60 liuchang
(integer) 6
127.0.0.1:6379> zscore mySort jitwxs
"100"
127.0.0.1:6379> zcard mySort
(integer) 6
127.0.0.1:6379> zrange mySort 0 -1
1) "dog"
2) "liuchang"
3) "zhouyang"
4) "wangnima"
5) "cat"
6) "jitwxs"
127.0.0.1:6379> zrange mySort 0 -1 withscores
 1) "dog"
 2) "33"
 3) "liuchang"
 4) "60"
 5) "zhouyang"
 6) "80"
 7) "wangnima"
 8) "82"
 9) "cat"
10) "100"
11) "jitwxs"
12) "100"
127.0.0.1:6379> zrevrange mySort 0 -1 withscores
 1) "jitwxs"
 2) "100"
 3) "cat"
 4) "100"
 5) "wangnima"
 6) "82"
 7) "zhouyang"
 8) "80"
 9) "liuchang"
10) "60"
11) "dog"
12) "33"
```

### 5.2 删除元素

| 含义 | 方法 |
|:------------- |:------------- |
| 删除成员 | zrem key member[member2...]|
| 按照下标范围删除成员 | zremrangebyrank key start stop |
| 按照 score 范围删除成员 | zremrangebyscore key min max |

```shell
127.0.0.1:6379> zrem mySort wangnima
(integer) 1
127.0.0.1:6379> zcard mySort
(integer) 5
```

```shell
127.0.0.1:6379> zrange mySort 0 -1
1) "dog"
2) "liuchang"
3) "zhouyang"
4) "cat"
5) "jitwxs"
127.0.0.1:6379> zremrangebyrank mySort 0 2
(integer) 3
127.0.0.1:6379> zrange mySort 0 -1
1) "cat"
2) "jitwxs"

```

```shell
127.0.0.1:6379> zrange mySort 0 -1 withscores
 1) "dog"
 2) "33"
 3) "jitwxs"
 4) "43"
 5) "liuchang"
 6) "60"
 7) "zhouyang"
 8) "80"
 9) "wangnima"
10) "82"
11) "cat"
12) "100"
127.0.0.1:6379> zremrangebyscore mySort 50 85
(integer) 3
127.0.0.1:6379> zrange mySort 0 -1 withscores
1) "dog"
2) "33"
3) "jitwxs"
4) "43"
5) "cat"
6) "100"
```

### 5.3 扩展方法

| 含义 | 方法 |
|:------------- |:------------- |
| 返回 score 在[min,max]的成员并按照 score 排序。[withscores]：显示 score；[limit offset count]：从 offst 开始返回 count 个成员| zrangebyscore key min max[withscores][limit offset count]|
| 设置指定成员增加的分数，返回值是修改后的分数 | zincrby key increment member |
| 获取分树在[min,max]的成员数量 | zcount key min max |
| 返回成员在集合中的排名（升序） | zrank key member |
| 返回成员在集合中的排名（降序） | zrevrank key member |

```shell
127.0.0.1:6379> zrange mySort 0 -1 withscores
 1) "dog"
 2) "33"
 3) "jitwxs"
 4) "43"
 5) "liuchang"
 6) "60"
 7) "zhouyang"
 8) "80"
 9) "wangnima"
10) "82"
11) "cat"
12) "100"
127.0.0.1:6379> zrangebyscore mySort 30 86 limit 2 3
1) "liuchang"
2) "zhouyang"
3) "wangnima"
127.0.0.1:6379> zcount mySort 0 60
(integer) 3
127.0.0.1:6379> zincrby mySort 17 jitwxs
"60"
127.0.0.1:6379> zrank mySort jitwxs
(integer) 1
127.0.0.1:6379> zrevrank mySort jitwxs
(integer) 4
```

## 六、key 的通用命令

| 含义 | 方法 |
|:------------- |:------------- |
| 获取所有于 pattern 匹配的 key。*：任意一个或多个字符，？：任意一个字符| keys pattern |
| 删除指定 key | del key[key2...]|
| 判断 key 是否存在 | exists key |
| 为 key 重命名| rename key newKey |
| 设置过期时间（单位s） | expire key |
| 获 取key 剩余的过期时间（单位s）。若没有设置过期时间，返回-1；超时不存在返回-2| ttl key |
| 获取 key 类型，key 不存在返回none | type key|

>注：如果你设置了一个 key 的过期时间，如果又不想让它过期，可以执行命令 `persist key`。

```shell
127.0.0.1:6379> keys *
 1) "unknown"
 2) "mySet2"
 3) "float_num"
 4) "myListB"
 5) "mySet3"
 6) "userName"
 7) "myListA"
 8) "int_num"
 9) "mySort"
10) "mySet1"
127.0.0.1:6379> del unknown
(integer) 1
127.0.0.1:6379> type myListA
list
127.0.0.1:6379> rename myListA myList
OK
127.0.0.1:6379> exists myListA
(integer) 0
```

```shell
127.0.0.1:6379> ttl mySort
(integer) -1
127.0.0.1:6379> expire mySort 30
(integer) 1
127.0.0.1:6379> ttl mySort
(integer) -2
127.0.0.1:6379> keys *
1) "mySet2"
2) "myList"
3) "float_num"
4) "myListB"
5) "mySet3"
6) "userName"
7) "int_num"
8) "mySet1"
```
