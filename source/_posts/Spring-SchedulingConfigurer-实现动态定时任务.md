---
title: Spring SchedulingConfigurer 实现动态定时任务
tags:
  - SchedulingConfigurer
  - 定时任务
categories:
  - Java Web
  - SpringBoot
abbrlink: e4d53ddb
date: 2021-03-27 23:23:41
related_repos:
  - name: dynamic-schedule-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/springboot-sample/dynamic-schedule-sample
---

## 一、前言

大家在日常工作中，一定使用过 Spring 的 `@Scheduled` 注解吧，通过该注解可以非常方便的帮助我们实现任务的定时执行。

但是该注解是不支持运行时动态修改执行间隔的，不知道你在业务中有没有这些需求和痛点：

- 在服务运行时能够动态修改定时任务的执行频率和执行开关，而无需重启服务和修改代码
- 能够基于配置，在不同环境/机器上，实现定时任务执行频率的差异化

这些都可以通过 Spring 的 `SchedulingConfigurer` 注解来实现。

这个注解其实大家并不陌生，如果有使用过 @Scheduled 的话，因为 @Scheduled 默认是单线程执行的，因此如果存在多个任务同时触发，可能触发阻塞。使用 SchedulingConfigurer 可以配置用于执行 @Scheduled 的线程池，来避免这个问题。

```java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        //设定一个长度10的定时任务线程池
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(10));
    }
}
```

但其实这个接口，还可以实现动态定时任务的功能，下面来演示如何实现。

## 二、功能实现

> 后续定义的类开头的 `DS` 是 `Dynamic Schedule` 的缩写。

使用到的依赖，除了 Spring 外，还包括：

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
</dependency>

<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <scope>provided</scope>
    <version>1.18.18</version>
</dependency>
```

### 2.1 @EnableScheduling

首先需要开启 `@EnableScheduling` 注解，直接在启动类添加即可：

```java
@EnableScheduling
@SpringBootApplication
public class DSApplication {
    public static void main(String[] args) {
        SpringApplication.run(DSApplication.class, args);
    }
}
```

### 2.2 IDSTaskInfo

定义一个任务信息的接口，后续所有用于动态调整的任务信息对象，都需要实现该接口。

- `id`：该任务信息的唯一 ID，用于唯一标识一个任务
- `cron`：该任务执行的 cron 表达式。
- `isValid`：任务开关
- `isChange`：用于标识任务参数是否发生了改变

```java
public interface IDSTaskInfo {
    /**
     * 任务 ID
     */
    long getId();

    /**
     * 任务执行 cron 表达式
     */
    String getCron();

    /**
     * 任务是否有效
     */
    boolean isValid();

    /**
     * 判断任务是否发生变化
     */
    boolean isChange(IDSTaskInfo oldTaskInfo);
}
```

### 2.3 DSContainer

顾名思义，是存放 IDSTaskInfo 的容器。

具有以下成员变量：

- `scheduleMap`：用于暂存 IDSTaskInfo 和实际任务 ScheduledTask 的映射关系。其中：
  - task_id：作为主键，确保一个 IDSTaskInfo 只会被注册进一次
  - T：暂存当初注册时的 IDSTaskInfo，用于跟最新的 IDSTaskInfo 比较参数是否发生变化
  - ScheduledTask：暂存当初注册时生成的任务，如果需要取消任务的话，需要拿到该对象
  - Semaphore：确保每个任务实际执行时只有一个线程执行，不会产生并发问题
- `taskRegistrar`：Spring 的任务注册管理器，用于注册任务到 Spring 容器中
- `name`：调用方提供的类名

具有以下成员方法：

- `void checkTask(final T taskInfo, final TriggerTask triggerTask)`：检查 IDSTaskInfo，判断是否需要注册/取消任务。具体的逻辑包括：
  - 如果任务已经注册：
    - 如果任务无效：则取消任务
    - 如果任务有效：
      - 如果任务配置发生了变化：则取消任务并重新注册任务
  - 如果任务没有注册：
    - 如果任务有效：则注册任务
- `Semaphore getSemaphore()`：获取信号量属性。

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.tuple.Pair;
import org.springframework.scheduling.config.ScheduledTask;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.scheduling.config.TriggerTask;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Semaphore;

/**
 * 存放 IDSTaskInfo 容器
 * @author jitwxs
 * @date 2021年03月27日 16:29
 */
@Slf4j
public class DSContainer<T extends IDSTaskInfo> {
    /**
     * IDSTaskInfo和真实任务的关联关系
     *
     * <task_id, <Task, <Scheduled, Semaphore>>>
     */
    private final Map<Long, Pair<T, Pair<ScheduledTask, Semaphore>>> scheduleMap = new ConcurrentHashMap<>();

    private final ScheduledTaskRegistrar taskRegistrar;

    private final String name;

    public DSContainer(ScheduledTaskRegistrar scheduledTaskRegistrar, final String name) {
        this.taskRegistrar = scheduledTaskRegistrar;
        this.name = name;
    }

    /**
     * 注册任务
     * @param taskInfo 任务信息
     * @param triggerTask 任务的触发规则
     */
    public void checkTask(final T taskInfo, final TriggerTask triggerTask) {
        final long taskId = taskInfo.getId();

        if (scheduleMap.containsKey(taskId)) {
            if (taskInfo.isValid()) {
                final T oldTaskInfo = scheduleMap.get(taskId).getLeft();

                if(oldTaskInfo.isChange(taskInfo)) {
                    log.info("DSContainer will register {} again because task config change, taskId: {}", name, taskId);
                    cancelTask(taskId);
                    registerTask(taskInfo, triggerTask);
                }
            } else {
                log.info("DSContainer will cancelTask {} because task not valid, taskId: {}", name, taskId);
                cancelTask(taskId);
            }
        } else {
            if (taskInfo.isValid()) {
                log.info("DSContainer will register {} task, taskId: {}", name, taskId);
                registerTask(taskInfo, triggerTask);
            }
        }
    }

    /**
     * 获取 Semaphore，确保任务不会被多个线程同时执行
     */
    public Semaphore getSemaphore(final long taskId) {
        return this.scheduleMap.get(taskId).getRight().getRight();
    }

    private void registerTask(final T taskInfo, final TriggerTask triggerTask) {
        final ScheduledTask latestTask = taskRegistrar.scheduleTriggerTask(triggerTask);
        this.scheduleMap.put(taskInfo.getId(), Pair.of(taskInfo, Pair.of(latestTask, new Semaphore(1))));
    }

    private void cancelTask(final long taskId) {
        final Pair<T, Pair<ScheduledTask, Semaphore>> pair = this.scheduleMap.remove(taskId);
        if (pair != null) {
            pair.getRight().getLeft().cancel();
        }
    }
}
```

### 2.4 AbstractDSHandler

下面定义实际的动态线程池处理方法，这里采用抽象类实现，将共用逻辑封装起来，方便扩展。

具有以下抽象方法：

- `List<T> listTaskInfo()`：获取所有的任务信息。
- `void doProcess(T taskInfo)`：实现实际执行任务的业务逻辑。

具有以下公共方法：

- `void configureTasks(ScheduledTaskRegistrar taskRegistrar)`：创建 DSContainer 对象，并创建一个单线程的任务定时执行，调用 scheduleTask() 方法处理实际逻辑。
- `void scheduleTask()`：首先加载所有任务信息，然后基于 cron 表达式生成 TriggerTask 对象，调用 checkTask() 方法确认是否需要注册/取消任务。当达到执行时间时，调用 execute() 方法，执行任务逻辑。
- `void execute(final T taskInfo)`：获取信号量，成功后执行任务逻辑。

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections4.CollectionUtils;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.scheduling.config.TriggerTask;
import org.springframework.scheduling.support.CronTrigger;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * 抽象 Dynamic Schedule 实现，基于 SchedulingConfigurer 实现
 * @author jitwxs
 * @date 2021年03月27日 16:41
 */
@Slf4j
public abstract class AbstractDSHandler<T extends IDSTaskInfo> implements SchedulingConfigurer {

    private DSContainer<T> dsContainer;
    
    private final String CLASS_NAME = getClass().getSimpleName();

    /**
     * 获取所有的任务信息
     */
    protected abstract List<T> listTaskInfo();

    /**
     * 做具体的任务逻辑
     *
     * <p/> 该方法执行时位于跟 SpringBoot @Scheduled 注解相同的线程池内。如果内部仍需要开子线程池执行，请务必同步等待子线程池执行完毕，否则可能会影响预期效果。
     */
    protected abstract void doProcess(T taskInfo) throws Throwable;

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        dsContainer = new DSContainer<>(taskRegistrar, CLASS_NAME);
        // 每隔 100ms 调度一次，用于读取所有任务
        taskRegistrar.addFixedDelayTask(this::scheduleTask, 1000);
    }

    /**
     * 调度任务，加载所有任务并注册
     */
    private void scheduleTask() {
        CollectionUtils.emptyIfNull(listTaskInfo()).forEach(taskInfo ->
                dsContainer.checkTask(taskInfo, new TriggerTask(() ->
                        this.execute(taskInfo), triggerContext -> new CronTrigger(taskInfo.getCron()).nextExecutionTime(triggerContext)
                ))
        );
    }

    private void execute(final T taskInfo) {
        final long taskId = taskInfo.getId();

        try {
            Semaphore semaphore = dsContainer.getSemaphore(taskId);
            if (Objects.isNull(semaphore)) {
                log.error("{} semaphore is null, taskId: {}", CLASS_NAME, taskId);
                return;
            }
            if (semaphore.tryAcquire(3, TimeUnit.SECONDS)) {
                try {
                    doProcess(taskInfo);
                } catch (Throwable throwable) {
                    log.error("{} doProcess error, taskId: {}", CLASS_NAME, taskId, throwable);
                } finally {
                    semaphore.release();
                }
            } else {
                log.warn("{} too many executor, taskId: {}", CLASS_NAME, taskId);
            }
        } catch (InterruptedException e) {
            log.warn("{} interruptedException error, taskId: {}", CLASS_NAME, taskId);
        } catch (Exception e) {
            log.error("{} execute error, taskId: {}", CLASS_NAME, taskId, e);
        }
    }
}
```

## 三、快速测试

至此就完成了动态任务的框架搭建，下面让我们来快速测试下。为了尽量减少其他技术带来的复杂度，本次测试不涉及数据库和真实的定时任务，完全采用模拟实现。

### 3.1 模拟定时任务

为了模拟一个定时任务，我定义了一个 `foo()` 方法，其中只输出一句话。后续我将通过定时调用该方法，来模拟定时任务。

```java
import lombok.extern.slf4j.Slf4j;

import java.time.LocalTime;

@Slf4j
public class SchedulerTest {
    public void foo() {
        log.info("{} Execute com.github.jitwxs.sample.ds.test.SchedulerTest#foo", LocalTime.now());
    }
}
```

### 3.2 实现 IDSTaskInfo

首先定义 IDSTaskInfo，我这里想通过反射来实现调用 `foo()` 方法，因此 `reference` 表示的是要调用方法的全路径。另外我实现了 `isChange()` 方法，只要 cron、isValid、reference 发生了变动，就认为该任务的配置发生了改变。

```java
import com.github.jitwxs.sample.ds.config.IDSTaskInfo;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class SchedulerTestTaskInfo implements IDSTaskInfo {
    private long id;

    private String cron;

    private boolean isValid;

    private String reference;

    @Override
    public boolean isChange(IDSTaskInfo oldTaskInfo) {
        if(oldTaskInfo instanceof SchedulerTestTaskInfo) {
            final SchedulerTestTaskInfo obj = (SchedulerTestTaskInfo) oldTaskInfo;
            return !this.cron.equals(obj.cron) || this.isValid != obj.isValid || !this.reference.equals(obj.getReference());
        } else {
            throw new IllegalArgumentException("Not Support SchedulerTestTaskInfo type");
        }
    }
}
```

### 3.3 实现 AbstractDSHandler

有几个需要关注的：

（1）`listTaskInfo()` 返回值我使用了 volatile 变量，便于我修改它，模拟任务信息数据的改变。

（2）`doProcess()` 方法中，读取到 reference 后，使用反射进行调用，模拟定时任务的执行。

（3）额外实现了 `ApplicationListener` 接口，当服务启动后，每隔一段时间修改下任务信息，模拟业务中调整配置。

- 服务启动后，foo() 定时任务将每 10s 执行一次。
- 10s 后，将 foo() 定时任务执行周期从每 10s 执行调整为 1s 执行。
- 10s 后，关闭 foo() 定时任务执行。
- 10s 后，开启 foo() 定时任务执行。

```java
import com.github.jitwxs.sample.ds.config.AbstractDSHandler;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

/**
 * @author jitwxs
 * @date 2021年03月27日 21:54
 */
@Component
public class SchedulerTestDSHandler extends AbstractDSHandler<SchedulerTestTaskInfo> implements ApplicationListener {
    public volatile List<SchedulerTestTaskInfo> taskInfoList = Collections.singletonList(
            SchedulerTestTaskInfo.builder()
                    .id(1)
                    .cron("0/10 * * * * ? ")
                    .isValid(true)
                    .reference("com.github.jitwxs.sample.ds.test.SchedulerTest#foo")
                    .build()
    );

    @Override
    protected List<SchedulerTestTaskInfo> listTaskInfo() {
        return taskInfoList;
    }

    @Override
    protected void doProcess(SchedulerTestTaskInfo taskInfo) throws Throwable {
        final String reference = taskInfo.getReference();
        final String[] split = reference.split("#");
        if(split.length != 2) {
            return;
        }

       try {
           final Class<?> clazz = Class.forName(split[0]);
           final Method method = clazz.getMethod(split[1]);
           method.invoke(clazz.newInstance());
       } catch (Exception e) {
           e.printStackTrace();
       }
    }

    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        Executors.newScheduledThreadPool(1).scheduleAtFixedRate(() -> {
            LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(10));

            // setting 1 seconds execute
            taskInfoList = Collections.singletonList(
                    SchedulerTestTaskInfo.builder()
                            .id(1)
                            .cron("0/1 * * * * ? ")
                            .isValid(true)
                            .reference("com.github.jitwxs.sample.ds.test.SchedulerTest#foo")
                            .build()
            );

            LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(10));

            // setting not valid
            taskInfoList = Collections.singletonList(
                    SchedulerTestTaskInfo.builder()
                            .id(1)
                            .cron("0/1 * * * * ? ")
                            .isValid(false)
                            .reference("com.github.jitwxs.sample.ds.test.SchedulerTest#foo")
                            .build()
            );

            LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(10));

            // setting valid
            taskInfoList = Collections.singletonList(
                    SchedulerTestTaskInfo.builder()
                            .id(1)
                            .cron("0/1 * * * * ? ")
                            .isValid(true)
                            .reference("com.github.jitwxs.sample.ds.test.SchedulerTest#foo")
                            .build()
            );
        }, 12, 86400, TimeUnit.SECONDS);
    }
}
```

### 3.4 运行程序

整个应用包结构如下：

![包结构](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts//202103/20210328003410.png)

运行程序后，在控制台可以观测到如下输出：

![运行结果](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts//202103/20210327232032.png)

## 四、后记

以上完成了动态定时任务的介绍，你能够根据本篇文章，实现以下需求吗：

- 本文基于 cron 表达式实现了频率控制，你能改用 fixedDelay 或 fixedRate 实现吗？
- 基于数据库/配置文件/配置中心，实现对服务中定时任务的动态频率调整和任务的启停。
- 开发一个数据表历史数据清理功能，能够动态配置要清理的表、清理的规则、清理的周期。
- 开发一个数据表异常数据告警功能，能够动态配置要扫描的表、告警的规则、扫描的周期。
