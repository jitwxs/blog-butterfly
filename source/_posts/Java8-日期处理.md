---
title: Java8 日期处理
categories: Java Basic
abbrlink: 2fc7970f
date: 2018-03-18 19:38:43
---

java中做时间处理时一般会采用`java.util.Date`，但是在Java 8中新增了对日期的处理类，包括`java.time.LocalDate`、`java.time.LocalTime`、`java.time.LocalDateTime`等。

`java.util.Date`和`SimpleDateFormatter`都是**线程不安全**的，而`LocalDate`和`LocalTime`和最基本的String一样，是不变类型，不但**线程安全**，而且不能修改。

在Java 8中，日期和时间被明确划分为`LocalDate`和`LocalTime`，`LocalDate`无法包含时间，`LocalTime`无法包含日期。当然，`LocalDateTime`才能同时包含日期和时间。

新接口更好用的原因是考虑到了日期时间的操作，经常发生往前推或往后推几天的情况。用`java.util.Date`配合`Calendar`要写好多代码。

### 获取实例

```java
// 1.使用now方法
LocalDate localDate = LocalDate.now();
LocalTime localTime = LocalTime.now();
LocalDateTime localDateTime = LocalDateTime.now();

// 2.使用of方法传参
LocalDate localDate2 = LocalDate.of(2018, 03, 18);
LocalTime localTime2 = LocalTime.of(19, 18);
LocalDateTime localDateTime2 = LocalDateTime.of(2018, 03, 18, 19, 18);
//LocalDateTime localDateTime2 = LocalDateTime.of(localDate2, localTime2);

// 3.字符串解析，严格按照规范验证，无效日期抛出DateTimeParseException异常
LocalDate localDate3 = LocalDate.parse("2018-03-18");
LocalTime localTime3 = LocalTime.parse("19:18");
```

### 一些方法

```java
// 取今天：2018-03-18
LocalDate today = LocalDate.now();

// 取本月第1天： 2018-03-01
LocalDate firstDayOfThisMonth = today.with(TemporalAdjusters.firstDayOfMonth());

// 取本月第10天：2018-03-10
LocalDate tenDayOfThisMonth = today.withDayOfMonth(10);

// 取本月最后一天（自动识别28、29、30、31）：2018-03-31
LocalDate lastDayOfThisMonth = today.with(TemporalAdjusters.lastDayOfMonth());

// 取下一天：2018-03-19
LocalDate lastDay = today.plusDays(1);

// 取这个月第一个周一：2018-03-05
LocalDate firstMondayOfThisMonth = today.with(TemporalAdjusters.firstInMonth(DayOfWeek.MONDAY));
```

### JDBC 映射

JDBC映射将把数据库的日期类型和Java 8的新类型关联起来：

```
SQL -> Java
--------------------------
date -> LocalDate
time -> LocalTime
timestamp -> LocalDateTime
```

### LocalDateTime --> Date （使用默认时区）

```java
LocalDateTime localDateTime = LocalDateTime.now();
		
ZonedDateTime zonedDateTime = localDateTime.atZone(ZoneId.systemDefault());

Date date = Date.from(zonedDateTime.toInstant());
```

### Date --> LocalDateTime （使用默认时区）

```java
Date date = new Date();
		
LocalDateTime localDateTime = LocalDateTime.ofInstant(date.toInstant(), ZoneId.systemDefault());
```
