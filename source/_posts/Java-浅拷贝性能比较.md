---
title: Java 浅拷贝性能比较
typora-root-url: ..
tags: 浅拷贝
categories: Java
icons:
  - fas fa-fire red
abbrlink: a9fa88a0
date: 2020-08-30 00:53:01
related_repos:
  - name: shallow_copy
    url: https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/shallow_copy
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---
## 一、前言

实际开发中，经常会遇到对象拷贝的需求，本文就结合日常开发过程中，使用到的浅拷贝技术，进行性能比较，看看谁更强。

**重要：** 下面将会花大量篇幅，列出各种类型浅拷贝的代码，你可以直接拖到文章末尾，看性能对比结果。然后再根据你中意的对象回过头来看它的代码，避免疲劳。

---

首先创建一个用于拷贝的 Bean，如下所示：

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.commons.lang3.RandomUtils;

import java.util.Date;

@Data
@Builder
public class User {
    private long id;

    private int age;

    private String name;

    private boolean isMale;

    private School school;

    private Date createDate;

    public static User mock() {
        return User.builder()
                .id(RandomUtils.nextLong())
                .age(RandomUtils.nextInt())
                .name(RandomStringUtils.randomAlphanumeric(5))
                .isMale(RandomUtils.nextBoolean())
                .school(new School(RandomStringUtils.randomAlphanumeric(5), RandomUtils.nextInt()))
                .createDate(new Date())
                .build();
    }
}

@AllArgsConstructor
class School {
    private String name;

    private int code;
}
```

然后编写一个模板类，给各个浅拷贝方法提供预热和耗时统计功能：

```java
public abstract class BaseCopyTest {
    public List<User> prepareData(int size) {
        List<User> list = new ArrayList<>(size);
        IntStream.range(0, size).forEach(e -> list.add(User.mock()));
        return list;
    }

    public User prepareOne() {
        return User.mock();
    }

    public void testCopy(List<User> data) {
        warnUp();

        long startTime = System.currentTimeMillis();

        copyLogic(data);

        System.out.println(name() + ": " + (System.currentTimeMillis() - startTime) + "ms");
    }

    abstract void warnUp();

    abstract void copyLogic(List<User> data);

    abstract String name();
}
```

## 二、工具类

首先介绍下工具类这边，代表“工具类”参赛的选手有：

- `Apache BeanUtils`——廉颇老矣
- `Spring BeanUtils`——夕阳红
- `Spring BeanCopier`——三十而立
- `Spring BeanCopier + Reflectasm`——身强力壮

### 2.1 Apache BeanUtils

`Apache BeanUtils` 算是一个比较古老的工具类，其自身是存在性能问题的，阿里巴巴开发手册中也明确禁止使用该工具，本次对比仍然把它加进来把。

想要用它需要导入依赖包：

```xml
 <dependency>
     <groupId>commons-beanutils</groupId>
     <artifactId>commons-beanutils</artifactId>
     <version>1.9.4</version>
</dependency>
```

```java
public class ApacheBeanUtilsTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        try {
            User target = new User();
            System.out.println(source);
            BeanUtils.copyProperties(target, source);
            System.out.println(target);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            try {
                BeanUtils.copyProperties(new User(), source);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    String name() {
        return "Apache BeanUtils";
    }
}
```

### 2.2 Spring BeanUtils

`Spring BeanUtils` 和 Apache Utils API 很像，但是在效率上比 Apache 效率更高，目前使用的人也不少。引入 `spring-beans` 依赖包后即可使用。

> Spring BeanUtils 的 `copyProperties()` 方法，第一个是源对象，第二个是目标对象。和 Apache BeanUtils 正好相反，要注意避免踩坑。 

```java
public class SpringBeanUtilsTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        User target = new User();
        System.out.println(source);
        BeanUtils.copyProperties(source, target);
        System.out.println(target);
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            BeanUtils.copyProperties(source, new User());
        }
    }

    @Override
    String name() {
        return "Spring BeanUtils";
    }
}
```

### 2.3 Spring BeanCopier

Spring 还为我们提供了一种基于 Cglib 的浅拷贝方式 `BeanCopier`，引入 `spring-core` 依赖包后即可使用，它被认为是取代 `BeanUtils` 的存在。

让我们编写一个工具类来使用 BeanCopier，如下所示：

```java
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

对应的，编写下它的测试类：

```java
public class BeanCopierUtilsTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        User target = new User();
        System.out.println(source);
        BeanCopierUtils.copyProperties(source, target);
        System.out.println(target);
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            BeanCopierUtils.copyProperties(source, new User());
        }
    }

    @Override
    String name() {
        return "Spring BeanCopier";
    }
}
```

### 2.4 Spring BeanCopier + Reflectasm

在大量对象拷贝过程中，new 操作往往是耗时的，Spring BeanCopier 并没有解决 new 这个动作。`Reflectasm` 是一个高性能的反射工具包，可以利用它来解决 new 步骤的耗时。使用 Reflectasm 需要引入依赖：

```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>reflectasm</artifactId>
    <version>1.11.9</version>
</dependency>
```

改造 BeanCopierUtils 类代码后如下：

```java
public class BeanCopierReflectasmUtils {
    private static final Map<String, BeanCopier> BEAN_COPIER_MAP = new ConcurrentHashMap<>();

    private static final Map<String, ConstructorAccess> CONSTRUCTOR_ACCESS_CACHE = new ConcurrentHashMap<>();

    private static final int MAX_CACHE_SIZE = 512;

    public static void copyProperties(Object source, Object target) {
        BeanCopier copier = getBeanCopier(source.getClass(), target.getClass());
        copier.copy(source, target, null);
    }

    public static <T> T copyProperties(T source, Class<T> targetClass) {
        if (source == null) {
            return null;
        }

        T target;
        try {
            ConstructorAccess<T> constructorAccess = getConstructorAccess(targetClass);
            target = constructorAccess.newInstance();
        } catch (RuntimeException e) {
            try {
                target = targetClass.newInstance();
            } catch (InstantiationException | IllegalAccessException e1) {
                throw new RuntimeException(String.format("Create new instance of %s failed: %s", targetClass, e.getMessage()));
            }
        }
        copyProperties(source, target);
        return target;
    }

    public static <T> List<T> copyProperties(List<?> sourceList, Class<T> targetClass) {
        if (CollectionUtils.isEmpty(sourceList)) {
            return Collections.emptyList();
        }

        ConstructorAccess<T> constructorAccess = getConstructorAccess(targetClass);
        List<T> resultList = new ArrayList<>(sourceList.size());
        for (Object source : sourceList) {
            T target;
            try {
                target = constructorAccess.newInstance();
            } catch (RuntimeException e) {
                try {
                    target = targetClass.newInstance();
                } catch (InstantiationException | IllegalAccessException e1) {
                    throw new RuntimeException(String.format("Create new instance of %s failed: %s", targetClass, e.getMessage()));
                }
            }

            copyProperties(source, target);
            resultList.add(target);
        }
        return resultList;
    }

    private static <T> ConstructorAccess<T> getConstructorAccess(Class<T> targetClass) {
        ConstructorAccess<T> constructorAccess = CONSTRUCTOR_ACCESS_CACHE.get(targetClass.getName());
        if(constructorAccess != null) {
            return constructorAccess;
        }
        try {
            constructorAccess = ConstructorAccess.get(targetClass);
            if (CONSTRUCTOR_ACCESS_CACHE.size() > MAX_CACHE_SIZE) {
                CONSTRUCTOR_ACCESS_CACHE.clear();
            }
            CONSTRUCTOR_ACCESS_CACHE.put(targetClass.getName(),constructorAccess);
        } catch (Exception e) {
            throw new RuntimeException(String.format("Create new instance of %s failed: %s", targetClass, e.getMessage()));
        }
        return constructorAccess;
    }

    private static BeanCopier getBeanCopier(Class<?> sourceClazz, Class<?> targetClazz) {
        String key = generatorKey(sourceClazz, targetClazz);
        BeanCopier copier;
        if(BEAN_COPIER_MAP.containsKey(key)) {
            copier = BEAN_COPIER_MAP.get(key);
        } else {
            copier = BeanCopier.create(sourceClazz, targetClazz, false);
            BEAN_COPIER_MAP.put(key, copier);
        }
        return copier;
    }

    private static String generatorKey(Class<?> sourceClazz, Class<?> targetClazz) {
        return sourceClazz + "_" + targetClazz;
    }
}
```

如上所示，拷贝方法通过 class 进行反射创建对象，并对 `ConstructorAccess` 进行缓存，提高效率。编写下它对应的测试类：

```java
public class BeanCopierReflectasmUtilsTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        try {
            System.out.println(source);
            System.out.println(BeanCopierReflectasmUtils.copyProperties(source, User.class));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            User target = BeanCopierReflectasmUtils.copyProperties(source, User.class);
        }
    }

    @Override
    String name() {
        return "Spring BeanCopier Reflectasm";
    }
}
```

## 三、原生类

回过头来介绍下代表 Java “原生类”参赛的选手：

- new——祖师爷
- clone——瘦死的骆驼比马大

### 3.1 new

咱们 java 面向对象编程学习的第一个关键字，非 new 莫属了。虽然浅拷贝用 new 未免太过于傻瓜，但还是把它请出来，看看它的性能咋样。

```java
public class NewTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        try {
            User target = new User();
            System.out.println(source);
            target.setId(source.getId());
            target.setAge(source.getAge());
            target.setName(source.getName());
            target.setMale(source.isMale());
            target.setSchool(source.getSchool());
            target.setCreateDate(source.getCreateDate());
            System.out.println(target);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            User target = new User();
            target.setId(source.getId());
            target.setAge(source.getAge());
            target.setName(source.getName());
            target.setMale(source.isMale());
            target.setSchool(source.getSchool());
            target.setCreateDate(source.getCreateDate());
        }
    }

    @Override
    String name() {
        return "Java New";
    }
}
```

### 3.2 clone

clone 也是 Java 原生提供的拷贝方法，并且据说性能还不错，我司项目里面就还有许多用 clone 的实现。咱们也拉出来比划比划：

使用 clone 咱们得先让对象实现 `Cloneable` 接口，修改 User：

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User implements Cloneable {
    private long id;

    private int age;

    private String name;

    private boolean isMale;

    private School school;

    private Date createDate;

    public static User mock() {
        return User.builder()
                .id(RandomUtils.nextLong())
                .age(RandomUtils.nextInt())
                .name(RandomStringUtils.randomAlphanumeric(5))
                .isMale(RandomUtils.nextBoolean())
                .school(new School(RandomStringUtils.randomAlphanumeric(5), RandomUtils.nextInt()))
                .createDate(new Date())
                .build();
    }

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return null;
    }
}

@AllArgsConstructor
class School {
    private String name;

    private int code;
}
```

再来编写它的测试类：

```java
public class CloneTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        System.out.println(source);
        System.out.println(source.clone());
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            Object target = source.clone();
        }
    }

    @Override
    String name() {
        return "Java Clone";
    }
}
```

## 四、Lombok

最后咱们咱来介绍下 Lombok 的浅拷贝，代表 Lombok 出场的有两位选手：

- toBuilder——后起之秀
- newBuilder——迅雷不及掩耳之势

### 4.1 toBuilder

想要开启 Lombok 的 toBuilder 功能，需要将 User 类上方的 `@Builder` 修改为 `@Builder(toBuilder = true)` 即可，编写它的测试类：

```java
public class ToBuilderTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        System.out.println(source);
        System.out.println(source.toBuilder().build());
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            User target = source.toBuilder().build();
        }
    }

    @Override
    String name() {
        return "Lombok toBuilder";
    }
}
```

### 4.2 newBuilder

再来介绍下 Lombok 的 newBuilder，它有点类似于 new，有点傻瓜，但也把它列出来，看看性能咋样：

```java
public class NewBuilderTest extends BaseCopyTest {

    @Override
    void warnUp() {
        User source = prepareOne();
        System.out.println(source);
        System.out.println(this.copy(source));
    }

    @Override
    void copyLogic(List<User> data) {
        for(User source : data) {
            User target = this.copy(source);
        }
    }

    private User copy(User source) {
        return User.builder()
                .id(source.getId())
                .age(source.getAge())
                .name(source.getName())
                .isMale(source.isMale())
                .school(source.getSchool())
                .createDate(source.getCreateDate())
                .build();
    }

    @Override
    String name() {
        return "Lombok newBuilder";
    }
}
```

## 五、测试

经过漫长的选手出场介绍，咱们终于可以进行性能对比了。首先介绍下本机器配置信息：

- Win10 专业版 1909
- AMD Ryzen 5 3600 6-Core
- 16GB RAM

测试均采用单线程测试，压测不同数据量情况下各种方式的耗时结果，测试结果如下（单位ms）。

| 类别                         | 1K   | 1W   | 10W  | 100W  | 500W  | 1000W  |
| ---------------------------- | ---- | ---- | ---- | ----- | ----- | ------ |
| Apache BeanUtils             | 27   | 134  | 1331 | 12902 | 28231 | 128566 |
| Spring BeanUtils             | 4    | 21   | 217  | 1949  | 2004  | 19755  |
| Spring BeanCopier            | 1    | 6    | 60   | 546   | 528   | 5114   |
| Spring BeanCopier Reflectasm | 2    | 8    | 72   | 569   | 563   | 5325   |
| Java New                     | 0    | 3    | 21   | 47    | 44    | 192    |
| Java Clone                   | 0    | 2    | 15   | 92    | 95    | 834    |
| Lombok toBuilder             | 0    | 1    | 10   | 40    | 42    | 263    |
| Lombok newBuilder            | 0    | 1    | 8    | 40    | 43    | 273    |

![Java 对象浅拷贝](/images/posts/20200830004302.png)

排除掉 BeanUtils 后，结果如下：

![Java 对象浅拷贝](/images/posts/20200830005819.png)

最后简单总结下：

1. 禁止使用 Apache BeanUtils，性能差到离谱
2. 不推荐使用 Spring BeanUtils，可以使用 Spring BeanCopier 替代
3. Spring BeanCopier Reflectasm 和 Spring BeanCopier 相比提升不了性能，但是写起来更简便（不需要显式 new 对象）
4. Java 原生的 new 和 clone 性能很高，可以使用 clone
5. Lombok 的 toBuilder 速度也很快，并且写起来很方便，推荐使用
