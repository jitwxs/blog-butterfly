---
title: 详解 Guava Cache
categories: Java
tags: [Cache, Guava]
abbrlink: 8fec5912
date: 2019-04-19 21:19:30
references:
  - name: Guava -CachesExplained
    url: https://github.com/google/guava/wiki/CachesExplained
  - name: Google Guava 3-缓存
    url: http://ifeve.com/google-guava-cachesexplained
  - name: google guava cache缓存基本使用讲解
    url: https://www.cnblogs.com/vikde/p/8045226.html
---
## 一、Guava Cache

{% note warning %}

推荐大家改用 [Caffine](/126e3eed.html)，性能更佳！

{% endnote %}

一般在项目中，本地缓存的实现为 `ConcurrentHashMap`，它具有线程安全、持久有效的特点。但是相较于传统缓存，它不具备缓存过期、缓存移除等特性，`Google Guava` 包内的 `Cache` 模块可能会给你一个新的选择。

Guava 目前托管于 [GitHub](https://github.com/google/guava)，在项目中引入也是十分简单：

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>27.1-jre</version>
    <!-- or, for Android: -->
    <version>27.1-android</version>
</dependency>
```

## 二、初始化

### 2.1 CacheLoader

如果我们希望对缓存中所有的 key 值使用同一个加载策略，那么推荐使用 `CacheLoader `方式加载。假设我们需要缓存的数据为：

```java
private static Map<String, List<Integer>> data = new HashMap<String, List<Integer>>() {{
    put("nanjing", new ArrayList<>(Arrays.asList(1, 3, 5, 7, 9)));
    put("beijing", new ArrayList<>(Arrays.asList(2, 4, 6, 8, 10)));
    put("shanghai", new ArrayList<>(Arrays.asList(0, -1, 11)));
}};
```

使用 `CacheLoader` 方式初始化：

```java
LoadingCache<String, List<Integer>> cache = CacheBuilder.newBuilder()
    .maximumSize(3)
    .build(new CacheLoader<String, List<Integer>>() {
        @Override
        public List<Integer> load(String i) throws Exception {
            return data.get(i);
        }
    });
```

其中 `maximumSize()`为缓存的最大容量，当缓存超过最大容量时，基于 `LRU` 清理缓存。`load()` 方法为缓存**未命中时的加载策略**。

同时使用 `get(K)` 方法进行缓存读取：

```java
try {
    System.out.println(cache.get("nanjing"));
} catch (ExecutionException e) {
    e.printStackTrace();
}

// output: [1, 3, 5, 7, 9]
```

> 调用时优先读取缓存，如果缓存中不存在该 key 值，则调用初始化时的 `load()` 方法，将该 key 值存入缓存，再返回。

### 2.2 Callable

`Callable` 则是另一种较为灵活的初始化形式，它在初始化的时候不指明加载策略：

```java
Cache<String, List<Integer>> cache2 = CacheBuilder.newBuilder().maximumSize(3).build();
```

在读取时再指定缓存的加载策略：

```java
try {
    List<Integer> list = cache2.get("nanjing", new Callable<List<Integer>>() {
        @Override
        public List<Integer> call() throws Exception {
            return data.get("nanjing");
        }
    });
    System.out.println(list);
} catch (ExecutionException e) {
    e.printStackTrace();
}

// output: [1, 3, 5, 7, 9]
```

> 一般情况下，使用 `CacheLoader` 初始化，如果你需要更为灵活的缓存加载策略，使用 `Callable` 初始化。

## 三、初始化属性

在上一节初始化的时候使用了 `maximumSize` 属性，除此之外，还有这些常用属性：

| 属性                                   | 描述                                                   |
| :-------------------------------------- | :------------------------------------------------------ |
| concurrencyLevel(4)                    | 并发级别为8，即可以同时写缓存的线程数，默认为4。       |
| initialCapacity(10)                    | 缓存的初始容量，默认为10                               |
| removalListener()                      | 指定缓存的移除通知                                     |
| expireAfterAccess(8, TimeUnit.SECONDS) | 缓存项在给定时间内没有被读/写访问，则回收。            |
| expireAfterWrite(8, TimeUnit.SECONDS)  | 缓存项在给定时间内没有被写访问（新增或覆盖），则回收。 |
| refreshAfterWrite(8, TimeUnit.SECONDS) | 缓存项在写访问（新增或覆盖）后给定时间，刷新。         |

## 四、基本操作

### 4.1 显式插入

Guava Cache 也支持使用 `put(key, value)` 方法手动的向缓存中插入数据，这会直接覆盖掉缓存中已存在的值。例如：

```java
cache.put("guangzhou", new ArrayList<>(Arrays.asList(101, 110, 222)));
```

### 4.2 显式清除

你可以手动的清除缓存项，主要方法包括：

- 根据 key 清除：`cache.invalidate(key)`
- 批量清除：`cache.invalidateAll(keys)`
- 清除所有：`cache.invalidateAll()`

### 4.3 getUnchecked

缓存的 `get()` 方法由于在缓存中不存在时，需要通过 `load()` 方法向缓存中加载值，可能出现异常，因此 `get()` 方法会抛出 `ExecutionException` 异常。

如果你定义的 `load()` 方法的实现没有抛出任何异常，那么可以直接使用 `getUnchecked()` 方法替换 `get()` 方法。

```java
// com.google.common.cache.LoadingCache#getUnchecked
public V getUnchecked(K key) {
    try {
        return this.get(key);
    } catch (ExecutionException var3) {
        throw new UncheckedExecutionException(var3.getCause());
    }
}
```

### 4.4 loadALL

 `get(K)` 方法根据 key 值查询缓存，通过 `getAll(Iterable<? extends K>)` 方法可以进行批量查询。默认情况下， `getAll()` 方法对缓存中不存在的值，会对每个值单个调用一次 `load()` 方法。

因此我们在初始化缓存的时候，可以通过重载 `loadAll()` 方法，来提升 `getAll()` 的性能。

```java
LoadingCache<String, List<Integer>> cache = CacheBuilder.newBuilder()
    .maximumSize(3)
    .build(new CacheLoader<String, List<Integer>>() {
        @Override
        public List<Integer> load(String i) throws Exception {
            return data.get(i);
        }

        @Override
        public Map<String, List<Integer>> loadAll(Iterable<? extends String> keys) throws Exception {
            return super.loadAll(keys);
        }
    });
```

### 4.5 asMap

Guava Cache 底层也是基于 `ConcurrentHashMap` 实现的，通过调用 `asMap()` 方法就能够得到缓存的 ConcurrentHashMap 实例：

```java
ConcurrentMap<String, List<Integer>> map = cache.asMap();
```
        
`asMap()` 方法是取得了缓存底层存储的实例，因此向 map 中 `put()` 就相当于向缓存中存储，但是从 map 中 `get()` 如果缓存中不存在的话，返回的是 null，即不不执行 `load()` 方法。

以上是我的理解，下面贴出官方原文便于参考：

> Note that no method on the `asMap` view will ever cause entries to be automatically loaded into the cache. Further, the atomic operations on that view operate outside the scope of automatic cache loading, so`Cache.get(K, Callable<V>)` should always be preferred over `Cache.asMap().putIfAbsent` in caches which load values using either `CacheLoader` or `Callable`.

## 五、缓存回收

Guava Cache 是完全基于内存的缓存，因此缓存越大对于内存的压力也就越大，Guava Cache 提供了三种缓存回收机制：**基于容量回收**、**定时回收**和**基于引用回收**。

### 5.1 基于容量回收

还记得初始化缓存时指定的 `maximumSize` 属性吗，当缓存容量逼近 `maximumSize` 阈值时，就会触发缓存回收，采用的回收策略是  `LRU(最近最少使用)` 策略。

除了根据缓存大小来回收以外，还可以根据**权重**进行回收，通过 `maximumWeight()` 指定**总的权重数**，`weigher()` 方法指定**如何计算缓存项的权重**。

需要注意的是，当总权重数逼近 `maximumWeight` 阈值时，就会触发缓存回收，因此一个良好的权重计算方法尤为重要。

```java
LoadingCache<String, List<Integer>> cache = CacheBuilder.newBuilder()
    .maximumSize(3)
    .maximumWeight(20)
    .weigher(new Weigher<String, List<Integer>>() {
        @Override
        public int weigh(String key, List<Integer> value) {
            // 根据缓存 value 的个数作为权重
            return value.size();
        }
    })
    .build(new CacheLoader<String, List<Integer>>() {
        @Override
        public List<Integer> load(String i) throws Exception {
            return data.get(i);
        }
    });
```

### 5.2 定时回收 

- `expireAfterAccess(long, TimeUnit)`：缓存项在给定时间内没有被读/写访问，则回收。
- `expireAfterWrite(long, TimeUnit)`：缓存项在给定时间内没有被写访问（新增或覆盖），则回收。

### 5.3 基于引用回收

通过使用**弱引用的键**或**弱引用的值**或**软引用的值**，可以在进行 `GC` 的时候回收缓存：

- `CacheBuilder.weakKeys()`：**使用弱引用存储键**。当键没有其它（强或软）引用时，缓存项可以被 GC。
- `CacheBuilder.weakValues()`：**使用弱引用存储值**。当值没有其它（强或软）引用时，缓存项可以 GC。
- `CacheBuilder.softValues()`：使用软引用存储值。由于 JVM 只会在内存不足时，才回收软引用，因此考虑到性能影响，不推荐使用该属性。

> 因为垃圾回收使用` ==` 判断是否有引用，因此使用上面属性后缓存的键或值使用 `==` 而不再使用 `equals()`。

## 六、回收策略

在实际使用中，一般通过 `expireAfterWrite` 和 `refreshAfterWrite` 进行缓存回收，下面介绍下这两种的回收策略。

测试模拟两个线程，Thread-1 每次读取后 sleep 3秒，Thread-2 每次读取后 sleep 5秒，缓存 load() 方法随机返回一个数字。代码如下：

```java
public static void main(String[] args) {
    LoadingCache<Integer, String> cache = CacheBuilder.newBuilder()
            .maximumSize(3)
            .removalListener(e -> System.out.println(e.getKey() + "->" + e.getValue() + " 被移除，移除原因：" + e.getCause()))
            .build(new CacheLoader<Integer, String>() {
                @Override
                public String load(Integer i) throws Exception {
                    System.out.println(Thread.currentThread().getName() + " 加载数据开始");
                    TimeUnit.SECONDS.sleep(5);
                    Random random = new Random();
                    System.out.println(Thread.currentThread().getName() + " 加载数据结束");
                    return "value:" + random.nextInt(10000);
                }
            });

    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss");
    new Thread(() -> {
        try {
            for (int i = 0; i < 20; i++) {
                String value = cache.get(1);
                System.out.println(Thread.currentThread().getName() + " " + LocalTime.now().format(formatter) + " " + value);
                TimeUnit.SECONDS.sleep(3);
            }
        } catch (Exception ignored) {
        }
    }).start();

    new Thread(() -> {
        try {
            for (int i = 0; i < 12; i++) {
                String value = cache.get(1);
                System.out.println(Thread.currentThread().getName() + " " + LocalTime.now().format(formatter) + " " + value);
                TimeUnit.SECONDS.sleep(5);
            }
        } catch (Exception ignored) {
        }
    }).start();
}
```

### 6.1 仅配置 expireAfterWrite

当我们仅为缓存配置 `expireAfterWrite` 时：

```java
.expireAfterWrite(8, TimeUnit.SECONDS)
```

启动程序，运行结果如下：

```java
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-2 15:34:21 value:3893
Thread-1 15:34:21 value:3893
Thread-1 15:34:24 value:3893
Thread-2 15:34:26 value:3893
Thread-1 15:34:27 value:3893
1->value:3893 被移除，移除原因：EXPIRED
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-1 15:34:35 value:4751
Thread-2 15:34:35 value:4751
Thread-1 15:34:38 value:4751
...
```

1. 首先 Thread-1 发现缓存没有数据，因此加载数据，由于 `load()` 是**原子操作**，因此 Thread-2 阻塞。
2. 当 Thread-1 加载完毕后，Thread-1 和 Thread-2 按照约定的 sleep 时间读取缓存。
3. 由于配置的 `expireAfterWrite` 为 8s，因此在 Thread-1 下次读取(15:34:30)时，**缓存过期**，此时Thread-1 重新加载数据。
4. Thread-2 原本应该在 15:34:31 时读取，但是此时缓存已经**因为过期被移除**。由于 Thread-1 已经开始加载，且  `load()` 方法是原子操作，因此 Thread-2 阻塞。
5. 在 Thread-1 加载完毕后，Thread-1 和 Thread-2 继续读取缓存。

**总结：**

{% note quote %}

当到达过期时间时，某一线程加载数据，其他线程阻塞至数据加载完毕。

{% endnote %}

### 6.2 仅配置 refreshAfterWrite

当我们仅为缓存配置 `refreshAfterWrite` 时：

```java
.refreshAfterWrite(8, TimeUnit.SECONDS)
```

启动程序，运行结果如下：

```java
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-2 15:55:45 value:5978
Thread-1 15:55:45 value:5978
Thread-1 15:55:48 value:5978
Thread-2 15:55:50 value:5978
Thread-1 15:55:51 value:5978
Thread-1 加载数据开始
Thread-2 15:55:55 value:5978
Thread-1 加载数据结束
1->value:5978 被移除，移除原因：REPLACED
Thread-1 15:55:59 value:3345
Thread-2 15:56:00 value:3345
Thread-1 15:56:02 value:3345
...
```

1. 首先 Thread-1 发现缓存没有数据，因此加载数据，由于 `load()` 是原子操作，因此 Thread-2 阻塞。
2. 当 Thread-1 加载完毕后，Thread-1 和 Thread-2 按照约定的 sleep 时间读取缓存。
3. 由于配置的 `refreshAfterWrite` 为 8s，因此在 Thread-1 下次读取(15:55:54)时，**缓存需要刷新**，此时 Thread-1 去重新加载数据。
4. Thread-2 在 15:55:55 时读取数据，虽然此时数据已经过期，但是由于 Thread-1 并没有更新完毕，因此 Thread-2 直接读取过期数据 5978。
5. Thread-1 更新数据完毕后，移除过期数据 5978，此时 Thread-1、Thread-2 再读取就是新数据 3345。

**总结：**

{% note quote %}

当到达刷新时间时，某一线程加载新数据，其他线程不阻塞读取旧数据。当新数据加载完毕时，所有线程读取新数据。

{% endnote %}

### 6.3 配置 expireAfterWrite 和 refreshAfterWrite

**注：** 在二者均配置的情况下，具体的执行策略不仅和二者的配置时间有关，还和加载的耗时有关，具体的策略涵盖以下四种：

1. expireAfterWrite < refreshAfterWrite
    - expireAfterWrite + 加载耗时 < refreshAfterWrite
    - expireAfterWrite + 加载耗时 > refreshAfterWrite
2.  refreshAfterWrite < expireAfterWrite
    - refreshAfterWrite + 加载耗时 < expireAfterWrite 
    - refreshAfterWrite + 加载耗时 > expireAfterWrite 

#### 6.3.1 expireAfterWrite < refreshAfterWrite

当我们为缓存配置如下时：

```java
.expireAfterWrite(6, TimeUnit.SECONDS)
.refreshAfterWrite(8, TimeUnit.SECONDS)
```

启动程序，运行结果如下：

```java
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-1 10:08:56 value:9299
Thread-2 10:08:56 value:9299
Thread-1 10:08:59 value:9299
Thread-2 10:09:01 value:9299
1->value:9299 被移除，移除原因：EXPIRED
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-1 10:09:07 value:890
Thread-2 10:09:07 value:890
Thread-1 10:09:10 value:890
Thread-2 10:09:12 value:890
1->value:890 被移除，移除原因：EXPIRED
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-2 10:09:18 value:2471
Thread-1 10:09:18 value:2471
...
```

1. 首先 Thread-1 发现缓存没有数据，因此加载数据，由于 `load()` 是原子操作，因此 Thread-2 阻塞。
2. 当 Thread-1 加载完毕后，Thread-1 和 Thread-2 按照约定的 sleep 时间读取缓存。
3. 配置的 `expireAfterWrite` 为 6s，因此缓存在 10:09:02 过期。当 Thread-1 在 10:09:02 读取时重新加载数据。
4. 由于 expireAfterWrite 执行时其他线程阻塞，因此 Thread-2 下次读取(10:09:06)时阻塞（数据在 10:09:07 加载完毕）。
5. 配置的 `refreshAfterWrite` 为 8s，因此缓存本应在 10.09:04 刷新，但是由于数据在 10:09:02 ~ 10:09:07 执行过期加载操作
6. 在 Thread-1 加载完毕后，Thread-1 和 Thread-2 继续读取缓存。
7. 此时 write 时间重置，expireAfterWrite 和 refreshAfterWrite 从加载完毕后时间重新计算。

**总结：** 

{% note quote %}

当 expireAfterWrite < refreshAfterWrite 情况下：

1. **expireAfterWrite + 加载耗时 < refreshAfterWrite 时**，先触发 expireAfterWrite，当重新加载完毕后，由于 write 时间重置，因此 refreshAfterWrite 配置无效。
2. **expireAfterWrite + 加载耗时 > refreshAfterWrite 时**，先触发 expireAfterWrite，在加载过程中 refreshAfterWrite 时间到了，但是由于 expireAfterWrite 加载时是阻塞的，refreshAfterWrite 被阻塞，等到加载完毕时 write 时间重置，因此 refreshAfterWrite 配置无效。

{% endnote %}

#### 6.3.2 refreshAfterWrite < expireAfterWrite

当我们为缓存配置如下时：

```java
.expireAfterWrite(8, TimeUnit.SECONDS)
.refreshAfterWrite(5, TimeUnit.SECONDS)
```

启动程序，运行结果如下：

```java
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-1 16:18:22 value:7593
Thread-2 16:18:22 value:7593
Thread-1 16:18:25 value:7593
Thread-2 加载数据开始
Thread-1 16:18:28 value:7593
Thread-2 加载数据结束
Thread-1 16:18:32 value:3558
1->value:7593 被移除，移除原因：EXPIRED
Thread-2 16:18:32 value:3558
Thread-1 16:18:35 value:3558
...
```

1. 首先 Thread-1 发现缓存没有数据，因此加载数据，由于 `load()` 是原子操作，因此 Thread-2 阻塞。
2. 当 Thread-1 加载完毕后，Thread-1 和 Thread-2 按照约定的 sleep 时间读取缓存。
3. 由于配置的 `refreshAfterWrite` 为 5s，因此在 Thread-2 下次读取(16:18:27)时，缓存刷新，此时Thread-2 重新加载数据。
4. Thread-1 在 16:18:28 时读取数据，虽然此时数据已经过期，但是由于 Thread-2 并没有更新完毕，因此 Thread-1 不会阻塞，读取过期数据7593。
5. 根据配置的 `expireAfterWrite` 为 8s，因此数据应该在 16:18:30 过期。但是数据在 16:18:27 ~ 16:18:32 期间因为需要刷新而重新加载中，因此 expireAfterWrite 并不会执行，而是**等待 refreshAfterWrite 执行完毕，将旧数据移除，直接拿 refreshAfterWrite 的结果作为 expireAfterWrite 的结果**。因此你可以看到上面输出的移除原因是 EXPIRED 而不是 REPLACED。

以上模拟了 **refreshAfterWrite + 加载耗时 > expireAfterWrite**（5 + 5 > 8）的情况，让我们调一下参数再看一下：

```java
.expireAfterWrite(12, TimeUnit.SECONDS)
.refreshAfterWrite(5, TimeUnit.SECONDS)
```

此时 **refreshAfterWrite + 加载耗时 < expireAfterWrite**（5 + 5 < 12），运行结果如下：

```java
Thread-1 加载数据开始
Thread-1 加载数据结束
Thread-2 11:32:27 value:5510
Thread-1 11:32:27 value:5510
Thread-1 11:32:30 value:5510
Thread-2 加载数据开始
Thread-1 11:32:33 value:5510
Thread-1 11:32:36 value:5510
Thread-2 加载数据结束
1->value:5510 被移除，移除原因：REPLACED
Thread-2 11:32:37 value:5492
Thread-1 11:32:39 value:5492
Thread-1 11:32:42 value:5492
Thread-2 加载数据开始
Thread-1 11:32:45 value:5492
Thread-2 加载数据结束
1->value:5492 被移除，移除原因：REPLACED
Thread-2 11:32:47 value:7624
...
```

1. 首先 Thread-1 发现缓存没有数据，因此加载数据，由于 `load()` 是原子操作，因此 Thread-2 阻塞。
2. 当 Thread-1 加载完毕后，Thread-1 和 Thread-2 按照约定的 sleep 时间读取缓存。
3. 由于配置的 `refreshAfterWrite` 为 5s，因此在 Thread-2 下次读取(11:32:32)时，缓存刷新，此时Thread-2 重新加载数据。
4. Thread-1 在 11:32:33 时读取数据，虽然此时数据已经过期，但是由于 Thread-2 并没有更新完毕，因此 Thread-1 不会阻塞，读取过期数据 5510。
5. 根据配置的 `expireAfterWrite` 为 12s，因此数据应该在 11:32:39 过期。但是数据在 11:32:37 已经被 refreshAfterWrite 更新完毕了，因此 write 时间重置，过期时间被更改为 11:32:49。
6. **expireAfterWrite 永远不会被执行**，因为还没到 11:32:49 ，在 11:32:42就开始刷新，在11:32:47就刷新完毕，write 时间再次重置。

**总结：** 

{% note quote %}

当 expireAfterWrite > refreshAfterWrite 情况下：

1. **refreshAfterWrite + 加载耗时 < expireAfterWrite 时**，先触发 refreshAfterWrite，当重新加载完毕后，由于 write 时间重置，因此 expireAfterWrite 配置无效。
2. **refreshAfterWrite + 加载耗时 > expireAfterWrite 时**，先触发 refreshAfterWrite，在加载过程中 expireAfterWrite 时间到了，但是 expireAfterWrite 并不会再去加载，而是直接阻塞等待 refreshAfterWrite ，并将其结果作为 expireAfterWrite 的结果。因此 refreshAfterWrite 和 expireAfterWrite 同时生效，expireAfterWrite 阻塞等待 refreshAfterWrite 的结果。

{% endnote %}

### 6.4 缓存清理和缓存刷新

**（1）什么时候清理？**

使用 CacheBuilder 构建的缓存不会"自动"执行清理和回收工作，也不会在某个缓存项过期后马上清理，也没有诸如此类的清理机制，它会在写操作或者读操作时再进行清理。

这样做的原因在于：如果要自动地持续清理缓存，就必须有一个线程，这个线程会和用户操作竞争共享锁。此外，某些环境下线程创建可能受限制，这样 CacheBuilder 就不可用了。

你也可以通过 `ScheduledExecutorService`，在其中定时执行 `cache.cleanUp()` 手动清理。

**（2）什么时候刷新？**

缓存项**只有在被检索时才会真正刷新**。如果你在缓存上同时声明 `expireAfterWrite` 和 `refreshAfterWrite`，缓存并不会因为刷新盲目地定时重置，如果缓存项没有被检索，那刷新就不会真的发生，缓存项在过期时间后也变得可以回收。

**（3）扩展刷新**

在构建缓存时，通过重载 `reaload(Key, OldValue)` 自定义刷新行为。

```java
LoadingCache<String, List<Integer>> cache = CacheBuilder.newBuilder()
    .maximumSize(3)
    .build(new CacheLoader<String, List<Integer>>() {
        @Override
        public List<Integer> load(String i) throws Exception {
            return data.get(i);
        }

        @Override
        public ListenableFuture<List<Integer>> reload(String key, List<Integer> oldValue) throws Exception {
            return super.reload(key, oldValue);
        }
    });
```

## 七、移除通知

在构建缓存时，通过指定 `removalListener` 来监听缓存的移除。

```java
LoadingCache<String, List<Integer>> cache = CacheBuilder.newBuilder()
    .maximumSize(3)
    .removalListener(e -> System.out.println(e.getKey() + "->" + e.getValue() + " 被移除，移除原因：" + e.getCause()))
    .build(new CacheLoader<String, List<Integer>>() {
        @Override
        public List<Integer> load(String i) throws Exception {
            return data.get(i);
        }
    });
```

需要注意的是，`removalListener` 在缓存被移除时是同步调用的，因此需要注意不要在其中做复杂操作，避免拖慢正常的缓存请求。

```java
// com.google.common.cache.LocalCache#processPendingNotifications
void processPendingNotifications() {
    RemovalNotification<K, V> notification;
    while ((notification = removalNotificationQueue.poll()) != null) {
      try {
        removalListener.onRemoval(notification);
      } catch (Throwable e) {
        logger.log(Level.WARNING, "Exception thrown by removal listener", e);
      }
    }
}
```

你可以通过 `RemovalListeners.asynchronous(RemovalListener, Executor)` 将其装饰为异步请求。

## 八、统计信息

在构建缓存时，通过添加 `.recordStats()` 来开启 Guava Cache 的统计功能，开启后通过 `cache.stats()` 得到 `CacheStats` 对象，提供了以下等统计信息：

| 函数                 | 描述                                 |
| :-------------------- | :------------------------------------ |
| hitRate()            | 缓存命中率                           |
| averageLoadPenalty() | 加载新值的平均时间，单位：ns         |
| evictionCount()      | 缓存项被回收的总数，不包括显式清除。 |
| loadExceptionRate()  | 缓存加载异常比率                     |
