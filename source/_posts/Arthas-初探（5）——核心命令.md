---
title: Arthas 初探（5）——核心命令
tags: Arthas
categories:
  - Java
  - 性能分析
abbrlink: 5ed3293c
date: 2020-12-27 20:01:49
---

## 一、前言

在本章节中，将学习以下 Arthas 的核心命令，同时我也会附上官方文档的链接，方便大家查阅：

- [monitor](https://arthas.aliyun.com/doc/monitor.html) 方法执行监控
- [watch](https://arthas.aliyun.com/doc/watch.html) 方法执行数据观测
- [trace](https://arthas.aliyun.com/doc/trace.html) 方法内部调用路径，并输出方法路径上的每个节点上耗时
- [stack](https://arthas.aliyun.com/doc/stack.html) 输出当前方法被调用的调用路径

- [tt](https://arthas.aliyun.com/doc/tt.html) 记录下指定方法每次调用的入参和返回信息，并能对这些不同时间下调用的信息进行观测

watch/trace/monitor/stack/tt 命令都支持 `-v` 参数。当命令执行之后，没有输出结果。有两种可能：

1. 匹配到的函数没有被执行
2. 条件表达式结果是 false

但用户区分不出是哪种情况。使用 `-v` 选项，则会打印 `Condition express` 的具体值和执行结果，方便确认。

## 二、monitor

对匹配 `class-pattern`／`method-pattern`／`condition-express`的类、方法的调用进行监控。

`monitor` 命令是一个非实时返回命令。

> 实时返回命令是输入之后立即返回，而非实时返回的命令，则是不断的等待目标 Java 进程返回信息，直到用户输入 `Ctrl+C` 为止。

服务端是以任务的形式在后台跑任务，植入的代码随着任务的中止而不会被执行，所以任务关闭后，不会对原有性能产生太大影响，而且原则上，任何 Arthas 命令不会引起原有业务逻辑的改变。

| 参数名称            | 参数说明                                |
| ------------------- | --------------------------------------- |
| *class-pattern*     | 类名表达式匹配                          |
| *method-pattern*    | 方法名表达式匹配                        |
| *condition-express* | 条件表达式                              |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配    |
| `[c:]`              | 统计周期，默认值为120秒                 |
| [b]                 | 在**方法调用之前**计算condition-express |

### 2.1 监控维度

| 监控项    | 说明                       |
| --------- | -------------------------- |
| timestamp | 时间戳                     |
| class     | Java类                     |
| method    | 方法（构造方法、普通方法） |
| total     | 调用次数                   |
| success   | 成功次数                   |
| fail      | 失败次数                   |
| rt        | 平均RT                     |
| fail-rate | 失败率                     |

### 2.2 指定时间间隔监控指定方法

每隔 5 秒监控一次类 demo.MathGame 中的 primeFactors 方法：

```shell
monitor -c 5 demo.MathGame primeFactors
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227200943802.png)

### 2.3 方法执行后过滤统计结果

方法执行后，每隔 5 秒监控一次类 demo.MathGame 中的 primeFactors 方法，筛选第一个参数 <= 2 的数据：

```shell
monitor -c 5 demo.MathGame primeFactors "params[0] <= 2"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227201138631.png)

### 2.4 方法执行前过滤统计结果

方法执行前，每隔 5 秒监控一次类 demo.MathGame 中的 primeFactors 方法，筛选第一个参数 <= 2 的数据：

```shell
monitor -b -c 5 demo.MathGame primeFactors "params[0] <= 2"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227201229931.png)

## 三、watch

能方便的观察到指定方法的调用情况。能观察到的范围为：`返回值`、`抛出异常`、`入参`，通过编写 OGNL 表达式进行对应变量的查看。

watch 的参数比较多，主要是因为它能在 4 个不同的场景观察对象：

| 参数名称            | 参数说明                                   |
| ------------------- | ------------------------------------------ |
| *class-pattern*     | 类名表达式匹配                             |
| *method-pattern*    | 方法名表达式匹配                           |
| *express*           | 观察表达式                                 |
| *condition-express* | 条件表达式                                 |
| [b]                 | 在**方法调用之前**观察                     |
| [e]                 | 在**方法异常之后**观察                     |
| [s]                 | 在**方法返回之后**观察                     |
| [f]                 | 在**方法结束之后**(正常返回和异常返回)观察 |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配       |
| [x:]                | 指定输出结果的属性遍历深度，默认为 1       |

这里重点要说明的是观察表达式，观察表达式的构成主要由 ognl 表达式组成，所以你可以这样写 `"{params,returnObj}"`，只要是一个合法的 ognl 表达式，都能被正常支持。

### 3.1 特别说明

- watch 命令定义了4个观察事件点，即 `-b` 方法调用前，`-e` 方法异常后，`-s` 方法返回后，`-f` 方法结束后
- 4个观察事件点 `-b`、`-e`、`-s` 默认关闭，`-f` 默认打开，当指定观察点被打开后，在相应事件点会对观察表达式进行求值并输出
- 这里要注意 `方法入参` 和 `方法出参` 的区别，有可能在中间被修改导致前后不一致，除了 `-b` 事件点 `params` 代表方法入参外，其余事件都代表方法出参
- 当使用 `-b` 时，由于观察事件点是在方法调用前，此时返回值或异常均不存在

### 3.2 观察方法出参和返回值

观察 demo.MathGame 类中 primeFactors 方法出参和返回值，结果属性遍历深度为 2。
params 表示所有参数数组(因为不确定是几个参数)，returnObject 表示返回值。

```shell
watch demo.MathGame primeFactors "{params,returnObj}" -x 2
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227201627726.png)

### 3.3 观察方法入参

观察方法入参，对比前一个例子，返回值为空（事件点为方法执行前，因此获取不到返回值）

```shell
watch demo.MathGame primeFactors "{params,returnObj}" -x 2 -b
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227201748514.png)

### 3.4 同时观察方法调用前和方法返回后

同时观察方法调用前和方法返回后，参数里 `-n 2`，表示只执行两次（一前一后）。

- 这里输出结果中，第一次输出的是方法调用前的观察表达式的结果，第二次输出的是方法返回后的表达式的结果

- params 表示参数，target 表示执行方法的对象，returnObject 表示返回值
- 结果的输出顺序和事件发生的先后顺序一致，和命令中 `-s -b` 的顺序无关

```shell
watch demo.MathGame primeFactors "{params,target,returnObj}" -x 2 -b -s -n 2
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227202109565.png)

### 3.5 观察当前对象中的属性

```shell
 watch demo.MathGame primeFactors 'target' -x 2 -n 1
```

如果觉得深度不够的话，可以调整 -x 的值，来看的更仔细。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227202340548.png)

然后使用 `target.field_name` 访问当前对象的某个属性

```shell
[arthas@18132]$ watch demo.MathGame primeFactors 'target.illegalArgumentCount' -x 2 -n 1
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 19 ms, listenerId: 23
ts=2020-12-27 20:32:15; [cost=0.0806ms] result=@Integer[1300]
Command execution times exceed limit: 1, so command will exit. You can set it with -n option.
```

### 3.6 条件表达式例子

只有满足条件表达式的调用，才会有响应。监控第一个出参小于 0 的调用。

```shell
watch demo.MathGame primeFactors "{params[0],target}" "params[0]<0"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227202525578.png)

### 3.7 观察异常信息的例子

监控抛出异常的调用，并打印第一个出参：

```shell
watch demo.MathGame primeFactors "{params[0],throwExp}" -e -x 2
```

- `-e` 表示抛出异常时才触发
- express 中，表示异常信息的变量是 `throwExp`

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227202733330.png)

### 3.8 按照耗时进行过滤

`#cost>200` (单位是`ms`)表示只有当耗时大于 200ms 时才会输出，过滤掉执行时间小于 200ms 的调用：

```shell
watch demo.MathGame primeFactors '{params, returnObj}' '#cost>200' -x 2
```

> watch / stack / trace 这个三个命令都支持 `#cost`

## 四、trace

`trace` 命令能主动搜索 `class-pattern` / `method-pattern` 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

| 参数名称            | 参数说明                             |
| ------------------- | ------------------------------------ |
| *class-pattern*     | 类名表达式匹配                       |
| *method-pattern*    | 方法名表达式匹配                     |
| *condition-express* | 条件表达式                           |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配 |
| `[n:]`              | 命令执行次数                         |
| `#cost`             | 方法执行耗时                         |

这里重点要说明的是观察表达式，观察表达式的构成主要由 ognl 表达式组成，所以你可以这样写 `"{params,returnObj}"`，只要是一个合法的 ognl 表达式，都能被正常支持。

### 4.1 注意事项

`trace` 能方便的帮助你定位和发现因 RT 高而导致的性能问题缺陷，但其每次只能跟踪一级方法的调用链路。

参考：[Trace命令的实现原理](https://github.com/alibaba/arthas/issues/597)

3.3.0 起支持使用动态 Trace 功能，不断增加新的匹配类，参考下面的示例。

### 4.2 trace 函数

trace 函数指定类的指定方法：

```shell
trace demo.MathGame run
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227204005224.png)

如果方法调用的次数很多，那么可以用 -n 参数指定捕捉结果的次数：

```shell
trace demo.MathGame run -n 1
```

### 4.3 包含 JDK 函数

默认情况下，trace 不会包含 JDK 里的函数调用，如果希望 trace JDK 里的函数，需要显式设置 `--skipJDKMethod false`：

```shell
trace --skipJDKMethod false demo.MathGame run
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227204136928.png)

### 4.4 按照调用耗时进行过滤

只会展示耗时大于 1ms 的调用路径，有助于在排查问题的时候，只关注异常情况。

```shell
trace demo.MathGame run '#cost > 1'
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227204311307.png)

- `trace` 在执行的过程中本身是会有一定的性能开销，在统计的报告中并未像 JProfiler 一样预先减去其自身的统计开销。所以这统计出来有些许的不准，渲染路径上调用的类、方法越多，性能偏差越大。但还是能让你看清一些事情的。
- 这里存在一个统计不准确的问题，就是所有方法耗时加起来可能会小于该监测方法的总耗时，这个是由于 Arthas 本身的逻辑会有一定的耗时。

### 4.5 trace 多个类或者多个函数

trace 命令只会 trace 匹配到的函数里的子调用，并不会向下 trace 多层。因为 trace 是代价比较贵的，多层 trace 可能会导致最终要 trace 的类和函数非常多。

可以用正则表匹配路径上的多个类和函数，一定程度上达到多层 trace 的效果。

```shell
trace -E com.test.ClassA|org.test.ClassB method1|method2|method3
```

### 4.6 动态 trace

打开终端1，trace 上面 demo 里的 `run` 函数，可以看到打印出 `listenerId: 1`：

```
[arthas@59161]$ trace demo.MathGame run
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 112 ms, listenerId: 1
`---ts=2020-07-09 16:48:11;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
    `---[1.389634ms] demo.MathGame:run()
        `---[0.123934ms] demo.MathGame:primeFactors() #24 [throws Exception]
 
`---ts=2020-07-09 16:48:12;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
    `---[3.716391ms] demo.MathGame:run()
        +---[3.182813ms] demo.MathGame:primeFactors() #24
        `---[0.167786ms] demo.MathGame:print() #25
```

现在想要深入子函数 `primeFactors`，可以打开一个新终端2，使用 `telnet localhost 3658` 连接上arthas，再 trace `primeFactors` 时，指定 `listenerId`。

```
[arthas@59161]$ trace demo.MathGame primeFactors --listenerId 1
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 34 ms, listenerId: 1
```

这时终端 2 打印的结果，说明已经增强了一个函数：`Affect(class count: 1 , method count: 1)`，但不再打印更多的结果。

再查看终端1，可以发现trace的结果增加了一层，打印了`primeFactors`函数里的内容：

```
`---ts=2020-07-09 16:49:29;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
    `---[0.492551ms] demo.MathGame:run()
        `---[0.113929ms] demo.MathGame:primeFactors() #24 [throws Exception]
            `---[0.061462ms] demo.MathGame:primeFactors()
                `---[0.001018ms] throw:java.lang.IllegalArgumentException() #46
 
`---ts=2020-07-09 16:49:30;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@3d4eac69
    `---[0.409446ms] demo.MathGame:run()
        +---[0.232606ms] demo.MathGame:primeFactors() #24
        |   `---[0.1294ms] demo.MathGame:primeFactors()
        `---[0.084025ms] demo.MathGame:print() #25
```

通过指定`listenerId`的方式动态 trace，可以不断深入。另外 `watch`/`tt`/`monitor` 等命令也支持类似的功能。

## 五、stack

很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多，或者你根本就不知道这个方法是从那里被执行了，此时你需要的是 stack 命令。

| 参数名称            | 参数说明                             |
| ------------------- | ------------------------------------ |
| *class-pattern*     | 类名表达式匹配                       |
| *method-pattern*    | 方法名表达式匹配                     |
| *condition-express* | 条件表达式                           |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配 |
| `[n:]`              | 执行次数限制                         |

这里重点要说明的是观察表达式，观察表达式的构成主要由 ognl 表达式组成，所以你可以这样写 `"{params,returnObj}"`，只要是一个合法的 ognl 表达式，都能被正常支持。

### 5.1 获取方法的调用路径

```shell
stack demo.MathGame primeFactors
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227205138934.png)

### 5.2 条件表达式来过滤

第 0 个参数的值小于 0，-n 表示获取 2 次

```shell
stack demo.MathGame primeFactors 'params[0]<0' -n 2
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227205230546.png)

### 5.3 按照执行时间进行过滤

过滤耗时大于0.5毫秒

```shell
stack demo.MathGame primeFactors '#cost>0.5'
```

## 六、tt

`watch` 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。

这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。

于是乎，TimeTunnel （时间隧道）命令就诞生了。

| 参数名称  | 参数说明                         |
| --------- | -------------------------------- |
| -t        | 记录某个方法在一个时间段中的调用 |
| -l        | 显示所有已经记录的列表           |
| -n 次数   | 只记录多少次                     |
| -s 表达式 | 搜索表达式                       |
| -i 索引号 | 查看指定索引号的详细调用信息     |
| -p        | 重新调用指定的索引号时间碎片     |

### 6.1 记录调用

最基本的使用来说，就是记录下当前方法的每次调用环境现场

```shell
tt -t demo.MathGame primeFactors
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227210237832.png)

| 表格字段  | 字段解释                                                     |
| --------- | ------------------------------------------------------------ |
| INDEX     | 时间片段记录编号，每一个编号代表着一次调用，后续tt还有很多命令都是基于此编号指定记录操作，非常重要。 |
| TIMESTAMP | 方法执行的本机时间，记录了这个时间片段所发生的本机时间       |
| COST(ms)  | 方法执行的耗时                                               |
| IS-RET    | 方法是否以正常返回的形式结束                                 |
| IS-EXP    | 方法是否以抛异常的形式结束                                   |
| OBJECT    | 执行对象的 `hashCode()`，注意，曾经有人误认为是对象在 JVM 中的内存地址，但很遗憾它不是，但它能帮助你简单的标记当前执行方法的类实体。 |
| CLASS     | 执行的类名                                                   |
| METHOD    | 执行的方法名                                                 |

- 条件表达式

  不知道大家是否有在使用过程中遇到以下困惑

  - Arthas 似乎很难区分出重载的方法
  - 我只需要观察特定参数，但是 tt 却全部都给我记录了下来

  条件表达式也是用 `OGNL` 来编写，核心的判断对象依然是 `Advice` 对象。除了 `tt` 命令之外，`watch`、`trace`、`stack` 命令也都支持条件表达式。

- 解决方法重载

  `tt -t *Test print params.length==1`

  通过指定参数个数的形式解决不同的方法签名，如果参数个数一样，你还可以这样写

  `tt -t *Test print 'params[1] instanceof Integer'`

- 解决指定参数

  `tt -t *Test print params[0].mobile=="13989838402"`

### 6.2 检索调用记录

当你用 `tt` 记录了一大片的时间片段之后，你希望能从中筛选出自己需要的时间片段，这个时候你就需要对现有记录进行检索。

```shell
tt -l
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227210522544.png)

筛选出异常方法的调用信息：

```shell
tt -s "isThrow==true"
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227211459017.png)

### 6.3 查看调用信息

对于具体一个时间片的信息而言，你可以通过 `-i` 参数后边跟着对应的 `INDEX` 编号查看到他的详细信息。

```shell
tt -i 1000
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227211603717.png)

### 6.4 重做一次调用

当你稍稍做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。

`tt` 命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个 `INDEX` 编号的时间片自主发起一次调用，从而解放你的沟通成本。此时你需要 `-p` 参数。通过 `--replay-times` 指定 调用次数，通过 `--replay-interval` 指定多次调用间隔(单位ms, 默认1000ms)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227211803317.png)

你会发现结果虽然一样，但调用的路径发生了变化，由原来的程序发起变成了 Arthas 自己的内部线程发起的调用了。

需要强调的点：

1. **ThreadLocal 信息丢失**

   很多框架偷偷的将一些环境变量信息塞到了发起调用线程的 ThreadLocal 中，由于调用线程发生了变化，这些 ThreadLocal 线程信息无法通过 Arthas 保存，所以这些信息将会丢失。

2. **引用的对象**

   需要强调的是，`tt` 命令是将当前环境的对象引用保存起来，但仅仅也只能保存一个引用而已。如果方法内部对入参进行了变更，或者返回的对象经过了后续的处理，那么在 `tt` 查看的时候将无法看到当时最准确的值。这也是为什么 `watch` 命令存在的意义。
