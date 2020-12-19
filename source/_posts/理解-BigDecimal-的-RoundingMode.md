---
title: 理解 BigDecimal 的 RoundingMode
categories: Java
abbrlink: f789a6b2
date: 2019-05-08 23:42:31
copyright_author: Jitwxs
---

在金融等对数据精度计算要求较高的领域，传统 double 运算无法满足要求， `BigDecimal` 类应运而生。实际使用中，`RoundingMode` 这个枚举类控制着小数的舍位原则，本文对该枚举类进行介绍。

### 一、RoundingMode.DOWN

**等价枚举：** BigDecimal.ROUND_DOWN

**舍位原则：** 粗暴截断舍弃位，不考虑任何进位舍位操作

**例：** Scale = 2

```java
Origin：3.33333333333333         OutPut：3.33
Origin：1.976744186046512        OutPut：1.97
Origin：-4.868913857677903       OutPut：-4.86
Origin：-2.307692307692308       OutPut：-2.3
```

### 二、RoundingMode.UP

**等价枚举：** BigDecimal.ROUND_UP

**舍位原则：** 精度保留的最后一位，朝远离数轴的方向进位。正数+1，负数-1

**例：** Scale = 2

```java
Origin：3.33333333333333         OutPut：3.34
Origin：1.976744186046512        OutPut：1.98
Origin：-4.868913857677903       OutPut：-4.87
Origin：-2.307692307692308       OutPut：-2.31
```

### 三、RoundingMode.CEILING

**等价枚举：** BigDecimal.ROUND_CEILING

**舍位原则：** 精度保留的最后一位，朝数轴正方向 round。正数时等价于 `UP`，负数时等价于 `DOWN`

**例：** Scale = 2

```java
Origin：3.33333333333333         OutPut：3.34
Origin：1.976744186046512        OutPut：1.98
Origin：-4.868913857677903       OutPut：-4.86
Origin：-2.307692307692308       OutPut：-2.3
```

### 四、RoundingMode.FLOOR

**等价枚举：** BigDecimal.ROUND_FLOOR

**舍位原则：** 与 `CEILING` 相反，在精度最后一位，朝数轴负方向 round。正数时等价于 `DOWN`，负数时等价于 `UP`

**例：** Scale = 2

```java
Origin：3.33333333333333         OutPut：3.33
Origin：1.976744186046512        OutPut：1.97
Origin：-4.868913857677903       OutPut：-4.87
Origin：-2.307692307692308       OutPut：-2.31
```

### 五、RoundingMode.HALF_UP

**等价枚举：** BigDecimal.ROUND_HALF_UP

**舍位原则：** 四舍五入

**例：** Scale = 2

```java
Origin：3.33333333333333         OutPut：3.33
Origin：1.976744186046512        OutPut：1.98
Origin：-4.868913857677903       OutPut：-4.87
Origin：-2.307692307692308       OutPut：-2.31
Origin：3.555                    OutPut：3.56
Origin：-3.555                   OutPut：-3.56
```

### 六、RoundingMode.HALF_DWON

**等价枚举：** BigDecimal.ROUND_HALF_DWON

**舍位原则：** 五舍六入

**例：** Scale = 2

```java
...
Origin：3.555                    OutPut：3.55
Origin：-3.555                   OutPut：-3.55
```

### 七、RoundingMode.HALF_EVEN

**等价枚举：** BigDecimal.ROUND_HALF_EVEN

**舍位原则：** 又称为“银行家舍入”，当舍入位非 5 时，四舍六入。当舍入位为5时，看舍入位前一位，即保留的最后一位，当其为奇数时进位，否则舍位。

**例：** Scale = 2

```java
...
Origin：3.535                    OutPut：3.54
Origin：-3.535                   OutPut：-3.54
Origin：3.585                    OutPut：3.58
Origin：-3.585                   OutPut：-3.58
```

###  八、RoundingMode.UNNECESSARY

**等价枚举：** BigDecimal.ROUND_UNNECESSARY

**舍位原则：** 断言请求，认为传入的数据一定满足设置的小数模式，如果不满足，抛出 ArithmeticException 异常。

**例：** Scale = 2

```java
...
Origin：3.530                    OutPut：3.53
Origin：3.531                    OutPut：ArithmeticException
```
