---
title: Java Fail-Fast 与 Fail-Safe 特性
tags:
  - Fail Fast
  - Fail Safe
categories: Java
icons:
  - fas fa-fire red
references:
  - name: Fail Fast and Fail Safe Iterators in Java
    url: 'https://www.geeksforgeeks.org/fail-fast-fail-safe-iterators-java/'
    rel: nofollow noopener noreferrer
    target: _blank
  - name: fail-fast与fail-safe在Java集合中的应用
    url: 'https://segmentfault.com/a/1190000014541691'
    rel: nofollow noopener noreferrer
    target: _blank
abbrlink: ae176768
date: 2020-08-22 23:08:12
---

## 一、前言

在 Java 的集合结构中，如果我们同时进行遍历（for-each, iterator）和集合修改（add, set, remove...）操作时，就有可能发生异常。例如，线程 T1 在对集合进行遍历，而此时线程 T2 对集合进行添加元素；亦或者线程 T1 在对集合进行遍历的过程中，进行删除元素操作。

不同的集合在遇到上述这种情况时，会有不同的处理。按照处理的不同，划分为 Fail-Fast 和 Non-Fail-Fast（下文统称为 Fail-Safe）两类。前者不允许在迭代的过程中对集合进行增删操作，否则抛出 `ConcurrentModificationException` 异常；后者则允许这种操作，不会抛出异常。

> 注：本处的“对集合进行增删操作”指的是对集合自身的操作，而非借助 iterator 的操作。

## 二、Fail-Fast

`Fail-Fast` 类型的数据结构，在进行迭代时如果检测到集合对象发生了**结构性修改**会立即抛出 `ConcurrentModificationException`，结构性修改指的是对该集合对象进行添加、删除或更新元素的操作。`ArrayList`, `HashMap` 返回的迭代器就是典型的 Fail-Fast。

### 2.1 样例展示

```java
import java.util.*;

public class FailFastExample {
    public static void main(String[] args) {
        Map<String, String> cityCode = new HashMap<>();
        cityCode.put("Delhi", "India");
        cityCode.put("Moscow", "Russia");
        cityCode.put("New York", "USA");  // 此时 HashMap 对象的 modCount=3

        // 返回 Iterator 对象时，会使用字段 expectedModCount 记录当前 HashMap 对象的 modCount
        Iterator<String> iterator = cityCode.keySet().iterator();

        while (iterator.hasNext()) {
            // 每次调用 iterator.next() 时都会检查 expectedModCount 是否与 modCount 相等，若不等则抛出 ConcurrentModificationException
            System.out.println(cityCode.get(iterator.next()));

            // 往 Map 对象添加元素，迭代器在下一次调用 next() 时会抛出异常
            cityCode.put("Istanbul", "Turkey");
        }
    }
} 
```

代码运行后输出如下结果：

```java
India
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.HashMap$HashIterator.nextNode(HashMap.java:1445)
	at java.util.HashMap$KeyIterator.next(HashMap.java:1469)
	at com.example.demo.FailFastExample.main(FailFastExample.java:12)
```

这与 Iterator 自身无关，使用传统的 For-Each 改写也会得到一样的输出：

```java
public class FailFastExample {
    public static void main(String[] args) {
        Map<String, String> cityCode = new HashMap<>();
        cityCode.put("Delhi", "India");
        cityCode.put("Moscow", "Russia");
        cityCode.put("New York", "USA");  // 此时 HashMap 对象的 modCount=3

        for (String s : cityCode.keySet()) {
            System.out.println(cityCode.get(s));
            cityCode.put("Istanbul", "Turkey");
        }
    }
} 
```

### 2.2 工作原理

支持 Fail-Fast 的集合，内部往往会有一个名为 `modCount` 的字段，该字段会在集合内元素被修改时进行更新。在遍历的过程中，如果发现 `modCount` 的值被修改，就会抛出 `ConcurrentModificationException`。以 `ArrayList#forEach()` 方法为例：

```java
@Override
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
  	for (int i=0; modCount == expectedModCount && i < size; i++) {
    		action.accept(elementData[i]);
  	}
  	if (modCount != expectedModCount) {
    		throw new ConcurrentModificationException();
  	}
}
```

### 2.3 小结

线程不安全的集合往往都是 Fail-Fast 的，由于不支持线程并发，Fail-Fast 迭代器选择抛出 `ConcurrentModificationException` 作为一个保底操作，作为开发者请不要利用该异常进行逻辑操作。

那么当我们使用集合在进行遍历时，如何知道哪些方法会导致 Fail-Fast 呢，粗略的说对集合进行添加、删除、更新操作都会导致，具体哪些方法你需要**看该类的源码，到底哪些方法会进行 `modCount` 变更操作**。

## 三、Fail-Safe

`Fail-Safe` 类型的数据结构，当集合对象发生结构性修改时**不会抛出任何异常**。这是因为迭代器在执行开始前，基于原集合类元拷贝了一份元素副本，因此迭代器执行的操作，如遍历、添加或删除等，并非原集合类对象的内部元素。也就是说，对原集合类对象的修改不会影响迭代器中的元素。`CopyOnWriteArrayList` 是这类的典型，例如：

```java
import java.util.concurrent.CopyOnWriteArrayList; 
import java.util.Iterator; 

class FailSafe { 
    public static void main(String args[]) 
    { 
        CopyOnWriteArrayList<Integer> list = new CopyOnWriteArrayList<Integer>(new Integer[] { 1, 3, 5, 8 }); 
        Iterator itr = list.iterator(); 
        while (itr.hasNext()) { 
            Integer no = (Integer)itr.next(); 
            System.out.println(no); 
            if (no == 8) 
                // 14不会被打印，因为迭代器遍历的是原集合对象内部元素的拷贝
                list.add(14); 
        } 
    } 
} 
```

代码运行后输出如下结果：

```java
1
3
5
8
```

另一种 Fail-Safe 的实现，不是依赖于拷贝副本的方式来实现，而是基于 Java 的内存弱一致性（weakly consistent，和强一致性相对于），意思就是 T1 线程在遍历的过程中，T2 线程产生的并发修改操作，可能会在 T1 输出中体现，也可能不体现，不强制要求 T1、T2 线程的内存一致。`ConcurrentHashMap` 是这类的典型，例如：

```java
import java.util.concurrent.ConcurrentHashMap; 
import java.util.Iterator; 

public class FailSafeItr { 
    public static void main(String[] args)  { 

        // 创建 ConcurrentHashMap 对象
        ConcurrentHashMap<String, Integer> map   = new ConcurrentHashMap<String, Integer>(); 

        map.put("ONE", 1); 
        map.put("TWO", 2); 
        map.put("THREE", 3); 
        map.put("FOUR", 4); 

        // 从集合对象中创建迭代器
        Iterator it = map.keySet().iterator(); 

        while (it.hasNext()) { 
            String key = (String)it.next(); 
            System.out.println(key + " : " + map.get(key)); 

            // 7会被打印，即对原集合元素的修改会在迭代器中反映出来，因此没有额外的拷贝空间
            map.put("SEVEN", 7); 
        } 
    } 
} 
```

代码运行后输出如下结果：

```java
ONE : 1
FOUR : 4
TWO : 2
THREE : 3
SEVEN : 7
```

## 四、总结

### 4.1 关键点

Fail Fast 迭代器的几个关键点：

- 遍历时一旦发现原集合对象发生修改则会抛出 `ConcurrentModificationException`.
- 被遍历的是原集合对象的元素。
- 这类迭代器不需要额外内存空间。

Fail Safe 迭代器的几个关键点：

- 允许在进行迭代时对原集合对象的元素进行修改。
- 在遍历的同时修改集合对象的元素，不会抛出任何异常。
- 一种实现是：遍历的是原集合类对象元素的拷贝，需要额外的内存空间用于拷贝的元素数组。
- 另一种实现是：`weakly consistent`，内存弱一致性。

### 4.2 数据结构枚举

下面列出分别使用 Fail-Fast 和 Fail-Safe 的数据结构，由于相关类太多，仅列出我用到的，欢迎留言补充。

（1）Fail-Fast

- ArrayList、LinkedList、Vector【虽然是线程安全的】
- HashSet、LinkedHashSet、TreeSet
- HashMap、LinkedHashMap、TreeMap、Hashtable【虽然是线程安全的】
- Collections.synchronizedXXX() 包装出的【虽然是线程安全的】

（2）Fail-Safe

- CopyOnWriteArrayList
- CopyOnWriteArraySet、ConcurrentSkipListSet、Sets.newConcurrentHashSet()【Guava 包提供的】
- ConcurrentHashMap
