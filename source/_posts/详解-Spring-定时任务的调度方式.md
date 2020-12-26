---
title: 详解 Spring 定时任务的调度方式
categories: Java Web
tags: Spring
abbrlink: 295bff8a
date: 2020-01-15 21:44:19
icons: [fas fa-fire red]
references:
  - name: 一张图让你秒懂Spring @Scheduled定时任务的fixedRate,fixedDelay,cron执行差异
    url: https://blog.csdn.net/applebomb/article/details/52400154
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 理解 Spring 定时任务的 fixedRate 和 fixedDelay 的区别
    url: https://yanbin.blog/understand-spring-schedule-fixedrate-fixeddelay/
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

在 Spring 中，我们可以使用 `@Scheduled` 方便的进行定时任务的执行，其支持以下三种调度方式：Cron、FixedDelay、FixedRate。下面分别介绍在标准模式下和异步模式下这三种调度方式的不同。

## 一、标准模式

### 1.1 示例准备

创建一个 SpringBoot 初始程序，依赖包只需要引入 spring-boot-starter-web 即可：

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

创建类 SchedulerConfig，实现 `SchedulingConfigurer` 接口，作用是自定义 `@Scheduled` 执行的线程池配置信息。

```java
@Configuration
public class SchedulerConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        // 执行线程数
        taskScheduler.setPoolSize(10);
        taskScheduler.initialize();
        taskRegistrar.setTaskScheduler(taskScheduler);
    }
}
```

为了方便起见，示例的定时任务直接定义在主类中，如下所示。

```java
@EnableScheduling
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    public void job() throws InterruptedException {
        LocalTime start = LocalTime.now();

        System.out.println(Thread.currentThread().getName() + " start @ "  + start);
        Thread.sleep(ThreadLocalRandom.current().nextInt(8) * 1000);

        LocalTime end = LocalTime.now();
        System.out.println(Thread.currentThread().getName() + " end @ " + end
        		 + ", seconds cost " + (ChronoUnit.SECONDS.between(start, end)));
    }
}
```

- `@EnableScheduling` 注解开启 Spring 对定时任务的自动配置和扫描。

- 每次任务执行将会随机 sleep 0 ~ 7 秒钟，模拟业务耗时。

至此我们只需要给 job() 方法上方加上 `@Scheduled` 注解就可以实现定时执行了。

### 1.2 Cron

Cron 本身语法比较简单和通用，具体语法看文章[《详解 Cron 表达式》](/482391a0.html)。以每隔 5 秒执行为例，给 job() 方法上方加上注解：

```java
@Scheduled(cron = "0/5 * * * * ? ")
```

程序运行结果如下：

```
ThreadPoolTaskScheduler-1 start @ 14:51:05.004
ThreadPoolTaskScheduler-1 end @ 14:51:08.008, seconds cost 3
ThreadPoolTaskScheduler-2 start @ 14:51:10.003
ThreadPoolTaskScheduler-2 end @ 14:51:16.007, seconds cost 6
ThreadPoolTaskScheduler-1 start @ 14:51:20.004
ThreadPoolTaskScheduler-1 end @ 14:51:23.008, seconds cost 3
...
```

首次执行时间为 14:51:05，本次执行耗时 3 秒。

第二次计划执行时间为 14:51:10，按时执行，本次执行耗时 6 秒。

第三次计划执行时间为 14:51:15，由于第二次执行直到 14:51:16 才执行完毕，因此本次不执行，等到下次 14:51:20 执行。

**由此可见**，Cron 方式严格按照执行周期执行，如果到了下次执行周期时，而本次执行未执行完毕，直接跳过。

### 1.3 FixedDelay

以程序启动后延时 1 秒执行，后续间隔 5 秒执行为例，给 job() 方法上方加上注解：

```java
@Scheduled(initialDelay = 1000, fixedDelay = 5000)
```

程序运行结果如下：

```
ThreadPoolTaskScheduler-3 start @ 15:00:12.106
ThreadPoolTaskScheduler-3 end @ 15:00:16.109, seconds cost 4
ThreadPoolTaskScheduler-2 start @ 15:00:21.113
ThreadPoolTaskScheduler-2 end @ 15:00:28.117, seconds cost 7
ThreadPoolTaskScheduler-4 start @ 15:00:33.121
ThreadPoolTaskScheduler-4 end @ 15:00:36.126, seconds cost 3
...
```

首次执行时间为 15:00:12，本次执行耗时 4 秒。

第二次计划执行时间为第一次执行结束时间 + 5 秒，即 15:00:21，本次执行耗时 6 秒。

第三次计划执行时间为第二次执行结束时间 + 5 秒，即 15:00:33。

**由此可见**，fixedDelay 将在上一次执行完毕后，等待指定时间后再进行下次执行，任务的执行间隔是固定的，不受执行时长的影响。

### 1.4 FixedRate

以程序启动后延时 1 秒执行，后续间隔 5 秒执行为例，给 job() 方法上方加上注解：

```java
@Scheduled(initialDelay = 1000, fixedRate = 5000)
```

程序运行结果如下：

```
ThreadPoolTaskScheduler-3 start @ 15:21:18.883
ThreadPoolTaskScheduler-3 end @ 15:21:18.883, seconds cost 0
ThreadPoolTaskScheduler-2 start @ 15:21:23.883
ThreadPoolTaskScheduler-2 end @ 15:21:29.887, seconds cost 6
ThreadPoolTaskScheduler-4 start @ 15:21:29.888
ThreadPoolTaskScheduler-4 end @ 15:21:31.892, seconds cost 2
ThreadPoolTaskScheduler-1 start @ 15:21:33.880
ThreadPoolTaskScheduler-1 end @ 15:21:35.882, seconds cost 2
...
```

和 fixedDelay 不同，fixedRate 在任务启动后，根据首次执行开始的时间将后续要执行的任务进行**预先编排**，以上面输出为例。首次执行开始时间为 15:21:18，那么编排如下：

| 编排执行顺序 | 编排执行时间 |
| ------------ | ------------ |
| T1           | 15:21:18     |
| T2           | 15:21:23     |
| T3           | 15:21:28     |
| T4           | 15:21:33     |

T1 执行时间为15:21:18，本次执行耗时 0 秒。

T2 计划执行时间为 15:21:23，由于 T1 到 15:21:18 就执行结束了，因此等待到该时刻后执行即可。本次执行耗时 6 秒。

T3 计划执行时间为 15:21:28，但由于 T2 直到 15:21:29 才执行结束，比预计执行时间还要晚。因此直接执行。本次执行耗时 2 秒。

T4 计划执行时间为 15:21:33，由于 T3 到 15:21:31 就执行结束了，因此等待到该时刻后执行即可。本次执行耗时 2 秒。

**由此可见**，fixedRate 在上次任务执行结束后，根据编排时间表决定是直接执行还是等待执行下次任务。如果预先编排的时间晚于上次执行时间 + fixedRate值，则等待到预定执行时间，否则立即执行。

### 1.5 总结

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202001/20200119214455363.png)

- **Cron：**  在预计的执行时间上，上次执行还未结束，就跳过。
- **fixedDelay：** 总是在上次执行完毕后，等待 n 秒后执行下次。
- **fixedRate：** 上次执行完毕后，比较编排时间和当前时间。如果超过编排时间，立即执行；如果未到编排时间，等待到编排时间。

| 调度方式   | 一言以蔽之     |
| ---------- | -------------- |
| cron       | 一旦错过就不在 |
| fixedDelay | 老死不相往来   |
| fixedRate  | 欠的总是要还的 |

## 二、异步模式

Spring 提供 `@Async` 注解实现了方法的异步执行，使用该注解的方法会在独立的线程中执行。也就是说使用了该注解的方法，就不受上文定义的 `SchedulerConfig` 的管理了。

### 2.1 示例准备

修改主类，新增 `@EnableAsync` 注解，同时给 job() 方法添加 `@Async` 注解，这样该方法就会以异步方式执行。

```java
@EnableAsync
@EnableScheduling
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Async
    public void job() throws InterruptedException {
        LocalTime start = LocalTime.now();

        System.out.println(Thread.currentThread().getName() + " start @ "  + start);
        Thread.sleep(ThreadLocalRandom.current().nextInt(8) * 1000);

        LocalTime end = LocalTime.now();
        System.out.println(Thread.currentThread().getName() + " end @ " + end + ", seconds cost " + (ChronoUnit.SECONDS.between(start, end)));
    }
}
```

### 2.1 Cron

给 job() 方法上方加上注解：

```java
@Scheduled(cron = "0/5 * * * * ? ")
```

程序运行结果如下：

```
task-1 start @ 15:59:35.006
task-1 end @ 15:59:38.011, seconds cost 3
task-2 start @ 15:59:40.005
task-3 start @ 15:59:45.004
task-3 end @ 15:59:46.006, seconds cost 1
task-2 end @ 15:59:47.008, seconds cost 7
task-4 start @ 15:59:50.004
task-4 end @ 15:59:55.005, seconds cost 5
...
```

首次执行时间为 15:59:35，本次执行耗时 3 秒。

第二次计划执行时间为 15:59:40，按时执行，本次执行耗时 7 秒。

第三次计划执行时间为 15:59:45，由于第二次执行直到 15:59:47 才执行完毕，因此另开线程执行。

**由此可见**，异步模式下的 Cron 方式严格按照执行周期执行，如果到了下次执行周期时，而本次执行未执行完毕，从线程池中取新的线程去执行。

### 2.3 FixedDelay

以程序启动后延时 1 秒执行，后续间隔 5 秒执行为例，给 job() 方法上方加上注解：

```java
@Scheduled(initialDelay = 1000, fixedDelay = 5000)
```

程序运行结果如下：

```
task-1 start @ 16:14:45.557
task-1 end @ 16:14:45.558, seconds cost 0
task-2 start @ 16:14:50.551
task-3 start @ 16:14:55.556
task-2 end @ 16:14:57.555, seconds cost 7
task-3 end @ 16:15:00.559, seconds cost 5
...
```

首次执行时间为 16:14:45，本次执行耗时 0 秒。

第二次计划执行时间为第一次执行开始时间 + 5 秒，即 16:14:50，本次执行耗时 7 秒。

第三次计划执行时间为第二次执行开始时间 + 5 秒，即 16:14:55。由于第二次执行直到 16:14:57 才执行完毕，因此另开线程执行。

**由此可见**，fixedDelay 将在上一次执行开始后，等待指定时间后再进行下次执行，如果到了下次执行周期时，而本次执行未执行完毕，从线程池中取新的线程去执行。

> 注意，异步模式下的 fixedDelay 感觉变成了 fixedRate 的感觉，竟然是根据任务开始时间而不是任务结束时间了？？？

### 2.4 FixedRate

以程序启动后延时 1 秒执行，后续间隔 5 秒执行为例，给 job() 方法上方加上注解：

```java
@Scheduled(initialDelay = 1000, fixedRate = 5000)
```

程序运行结果如下：

```
task-1 start @ 16:21:27.164
task-1 end @ 16:21:29.168, seconds cost 2
task-2 start @ 16:21:32.150
task-3 start @ 16:21:37.149
task-2 end @ 16:21:38.150, seconds cost 6
task-4 start @ 16:21:42.151
task-4 end @ 16:21:43.155, seconds cost 1
task-3 end @ 16:21:44.150, seconds cost 7
...
```

还是根据标准模式一样，根据首次执行时间，进行任务编排：

| 编排执行顺序 | 编排执行时间 |
| ------------ | ------------ |
| T1           | 16:21:27     |
| T2           | 16:21:32     |
| T3           | 16:21:37     |
| T4           | 16:21:42     |

T1 执行时间为 16:21:27，本次执行耗时 2 秒。

T2 计划执行时间为 16:21:32，由于 T1 到 16:21:29 就执行结束了，因此等待到该时刻后执行即可。本次执行耗时 6 秒。

T3 计划执行时间为 16:21:37，但由于 T2 直到 16:21:38 才执行结束，因此另开线程执行。本次执行耗时 7 秒。

T4 计划执行时间为 16:21:42，但由于 T3 直到 16:21:44 才执行结束，因此另开线程执行。本次执行耗时 1 秒。

**由此可见**，fixedRate 在上次任务执行结束后，根据编排时间表决定是直接执行还是等待执行下次任务。如果预先编排的时间晚于上次执行时间 + fixedRate 值，则等待到预定执行时间，否则另开线程执行。

### 2.5 总结

首先说明下定时任务的异步模式用的很少，@Async 主要还是给非定时任务用的比较多。定时任务在该模式下，三种方式的调度逻辑一致，即**本次执行时，上次执行是否完毕，如果完毕直接执行；如果未完毕，新开线程执行。**
