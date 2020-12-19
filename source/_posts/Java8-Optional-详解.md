---
title: Java8 Optional 详解
tags: Optional
categories: Java
abbrlink: c6a78a53
date: 2018-03-14 10:37:35
copyright_author: Jitwxs
---

在Java8中新增了一个`Optional`类，官方描述是该类是一个容器对象，其中可能**包含一个空或非空的值**。如果存在一个值，`isPresent()`将返回true，`get()`将返回该值。

### 错误使用姿势

简单的根据描述，我们认为Optional可以帮我们解决**NPE问题**，假设任务需求为获取用户的性别，那么可以这样写：

```java
Optional<User> user = ...
if(user.isPresent()) {
	return user.get().getSex();
} else {
	return null;
}
```

其实这样并没有改变原本的思维, 只是本能的认为该类是 User 实例的包装, 这与下面的写法没有本质的区别：

```java
User user = ...
if(user != null) {
	return user.getSex();
} else {
	return null;
}
```

### 正确使用姿势

想要正确的使用，得知道有哪些方法，下面我列出一些常用的API：

| Modifier and Type | Method | Description |
|:------------- |:-------------|:-----| 
| static < T > Optional< T > | empty() | 返回一个空的Otptional实例 |
| static < T > Optional< T > | of(T value) | 传入的值必须非null，否则抛出NPE |
| static < T > Optional< T > | ofNullable(T value)| 当传入null时，返回空的Optional实例 |
| < U > Optional< U > | map(Function<? super T,? extends U> mapper) | 如果值存在，则将所提供的映射函数应用于该函数，否则返回一个描述结果的可选项 |
| T | orElse(T other) | 如果值存在返回该值，否则返回传入的值 |
| T | orElseGet(Supplier<? extends T> other) | 如果值存在返回该值，否则调用其他值并返回该调用的结果 |
| void | ifPresent(Consumer<? super T> consumer) | 当值存在时执行参数上的语句 |
| Optional< T > | filter(Predicate<? super T> predicate) | 当值存在且符合指定规则返回该值，否则返回null |
| < X extends Throwable> T | orElseThrow(Supplier<? extends X> exceptionSupplier) | 如果值存在返回该值，否则抛出指定异常 |

前三个为构造方法，一般使用`ofNullable`即可，因为它能智能的判断参数是否为null，但是该方法也不是一劳永逸的，否则也没必要外露另外两个构造方法，如果我们明确的知道参数为非null或不允许任何null值的存在，可以使用`of`。

对于其他方法，给出例子相信能很快理解：

```java
/* ===== 得到User实例 ==== */

// 等价于 return user.isPresent() ? user.get() : null;
return user.orElse(null); 

// 等价于 return user.isPresent() ? user: getNewUser();
return user.orElseGet(() -> getNewUser());

/*
 等价于 
 if (user.isPresent()) { 
	 System.out.println(user.get()); 
 }
 */
user.ifPresent(System.out::println);
```

```java
/* ===== 得到User.sex ==== */

/*
 等价于：
 if(user.isPresent()) {
	  return user.get().getSex();
 } else {
	  return "男";
 }
*/
return user.map(u -> u.getSex()).orElse("男");
```

总结：使用 `Optional` 时尽量**不直接调用 `Optional.get()` 方法,** `Optional.isPresent()` 更应该被视为一个**私有方法**, 应使用像 `Optional.orElse()`, `Optional.orElseGet()`, `Optional.map()` 等这样的方法。


