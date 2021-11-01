---
title: Java 代码性能优化之路
categories:
  - Java
  - 性能分析
abbrlink: 94186b3a
date: 2020-05-01 20:21:12
tags:
related_repos:
  - name: performance-optimized
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/performance-optimized
---

## 一、前言

最近一直忙着参与公司的新项目开发，由于临期上线，正在对系统进行性能压测，在这个过程中，发现一些代码有性能优化的空间。因此决定写一篇文章，把本次以及今后，遇到的性能优化的 case 都记录下来，希望对大家们的编码水平能够有所帮助。

## 二、Java 基础

### 2.1 字符串拼接

在我们的系统中，存在着大量的缓存，这些缓存的 key 值根据请求参数的不同而拼接起来，如下代码所示：

```java 优化前
public class LastPriceCache {
    private String keySuffix = "last_price_%s";

    public double getLastPrice(int id) {
        return redisService.get(this.generatorKey(id));
    }

    private String generatorKey(int id) {
        return String.format(keySuffix, id);
    }
}
```

字符串拼接存在性能损耗，如果 id 值是可预期的，完全可以将其在内存中缓存起来，如下代码所示：

```java 优化后
public class LastPriceCache {
    private String keySuffix = "last_price_%s";

    private Map<Integer, String> keyMap = new HashMap<>();

    public double getLastPrice(int id) {
        return redisService.get(this.generatorKey(id));
    }

    private String generatorKey(int id) {
        String result;
        if(!keyMap.containsKey(id)) {
            result = keyMap.get(id);
        } else {
            result = String.format(keySuffix, id);
            keyMap.put(id, result);
        }
        return result;
    }
}
```

### 2.2 装箱类型

在我们项目中的问题代码，是在使用 `BigDecimal` 进行除法操作时，精度保留的小数位数的变量使用了包装类型。可惜在我自己的电脑上没有办法复现出来，也许是公司的电脑配置太低了，哈哈。

因此我就用包装类型累加来举例把，虽然例子都烂大街了，但能说明问题。将如下代码中的 `Long` 改成 `long`，就会得到不一样的耗时结果。

```java
Long value = 0;
for(int i = 0; i < 100_0000; i ++) {
    value += i;
}
```

### 2.3 Stream 合并

在我们的系统中，存在分页查询用户订单的需求，涉及到对源数据的过滤/截取/排序操作，如下代码所示：

```java 优化前
List<Order> sourceOrder = ...;

List<Order> result = sourceOrder.stream().filter(e -> e.getAmount() > 0).collect(Collectors.toList());

if(startId > 0) {
    result = result.stream().filter(e -> e.getId() >= startId).collect(Collectors.toList());
}
if(endId > 0) {
    result = result.stream().filter(e -> e.getId() < endId).collect(Collectors.toList());
}

if(result.size() > limit) {
    result = result.subList(0, limit);
}

Collections.reverse(result);
```

JDK 1.8 中引入了流式操作，很大程度上使代码变得更加简洁，但是使用不当也会拖慢性能。在上面代码中，滥用了流式操作，完全可以进行合并操作，且后续的截取和排序操作也可以整合在流式操作中，如下代码所示：

```java 优化后
List<Order> sourceOrder = ...;

Stream<Order> stream = sourceOrder.stream();

stream = stream.filter(e -> e.getAmount() > 0);

if(startId > 0) {
    stream = stream.filter(e -> e.getId() >= startId);
}
if(endId > 0) {
    stream = stream.filter(e -> e.getId() < endId);
}

Comparator<Order> comparator = Comparator.comparingLong(Order::getId);
if(!isAsc) {
    comparator = comparator.reversed();
}

List<Order> result = stream.limit(limit).sorted(comparator).collect(Collectors.toList());
```

## 三、三方包

### 3.1 FastJson 预热

在我们项目中，采用 FastJson 作为序列化库。FastJson 虽然号称速度非常快，但是其在首次序列化时速度却是让人大跌眼镜，在压测环境下，一下次就被暴露出来了。

只需要在程序启动时，静态加载以下两个 FastJson 类，问题就可以解决。

```java
static {
    new ParserConfig();
    new SerializeConfig();
}
```

### 3.2 浅拷贝

之前项目中，直接使用 spring 框架的 `BeanUtils` 进行浅拷贝，在压测中也发现其耗时比较严重。而使用同为 spring 提供的 `BeanCopier` 则性能很好。

```java BeanCopier 工具类
public class BeanCopierUtils {
    private static final Map<String, BeanCopier> CACHE = new ConcurrentHashMap<>();

    public static void copyProperties(Object source, Object target) {
        BeanCopier copier = getBeanCopier(source.getClass(), target.getClass());
        copier.copy(source, target, null);
    }

    private static BeanCopier getBeanCopier(Class<?> sourceClazz, Class<?> targetClazz) {
        String key = generatorKey(sourceClazz, targetClazz);
        BeanCopier copier;
        if(CACHE.containsKey(key)) {
            copier = CACHE.get(key);
        } else {
            copier = BeanCopier.create(sourceClazz, targetClazz, false);
            CACHE.put(key, copier);
        }
        return copier;
    }

    private static String generatorKey(Class<?> sourceClazz, Class<?> targetClazz) {
        return sourceClazz + "_" + targetClazz;
    }
}
```
