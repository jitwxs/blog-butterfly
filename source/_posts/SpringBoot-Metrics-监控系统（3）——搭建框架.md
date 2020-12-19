---
title: SpringBoot Metrics 监控系统（3）——搭建框架
copyright_author: Jitwxs
tags:
  - SpringBoot
  - Prometheus
  - Grafana
categories: 云原生
abbrlink: 527f9dce
date: 2020-11-15 21:06:13
related_repos:
  - name: springboot_metrics
    url: https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/springboot_metrics
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、前言

本章节开始将为大家展示如何在 SpringBoot 应用中去使用 Metrics 监控。本系列使用的 SpringBoot 版本为笔者当前的最新 RELAESE 版本 `2.4.0`，整个 SpringBoot 2 关于这边都是大同小异，所以大家不用担心版本问题。

## 二、依赖包

除了常规开发 SpringBoot Web 所需要的两个包外：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

我们还需要导入 SpringBoot 的监控端点 actuator 包和 Prometheus 包：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

另外为了便于开发，我还使用了 Lombok：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

## 三、Metrics Enums

对于刚刚入门使用的同学，经常会出现 Metrics 指标的管理混乱的情况。在代码里这处注册一个指标，那处注册一个，很混乱。因此有必要提供一个类，专门管理所有的 Metrics 指标，并统一注册、

这里我使用枚举类实现。当然你也可以根据自己需要使用其他方式实现。一共有以下两个类：

- `MetricsTypeEnum` 枚举系统支持的所有 Metrics 类型
- `MetricsEnum` 枚举所有的指标，标识了每一个指标的 name、tags、description、type 等属性

### 3.1 MetricsTypeEnum

```java
public enum MetricsTypeEnum {
    UNKNOWN,
    COUNTER,
    GAUGE,
    TIMER
    ;
}
```

### 3.2 MetricsEnum

```java
@Getter
@AllArgsConstructor
public enum MetricsEnum {
    DEFAULT("default", null, "default description", UNKNOWN),
    ;

    private final String name;

    private final String[] tags;

    private final String description;

    private final MetricsTypeEnum type;
}
```

## 四、Metrics Support

首先编写一些使用 Prometheus 的 Metrics 的工具类，一共有以下几个类：

- `MetricsRegisterConfig` 交于 Spring 容器管理，负责获取到 Prometheus 的全局实例对象
- `BaseMetricsUtil` 基础的 Metrics 工具类，抽取一些共用方法
- `CounterMetricsUtil` Counter 类型的 Metrics 工具类
- `GaugeMetricsUtil` Gauge 类型的 Metrics 工具类
- `TimerMetricsUtil` Timer 类型的 Metrics 工具类

### 4.1 MetricsRegisterConfig

```java
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MetricsRegisterConfig implements BeanPostProcessor {
    @Value("${spring.application.name}")
    private String applicationName;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof MeterRegistry) {
            MeterRegistry registry = (MeterRegistry) bean;
            registry.config().commonTags("application", applicationName);

            BaseMetricsUtil.meterRegistry = registry;
        }

        return bean;
    }
}
```

### 4.2 BaseMetricsUtil

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.util.Assert;

public class BaseMetricsUtil {
    public static MeterRegistry meterRegistry;

    protected static String CLASS_NAME = new Object() {
        public String getClassName() {
            String clazzName = this.getClass().getName();
            return clazzName.substring(0, clazzName.lastIndexOf("$"));
        }
    }.getClassName();

    public static void basicCheck(final MetricsEnum metricsEnum) {
        Assert.notNull(meterRegistry, CLASS_NAME + " meterRegistry not allow null");

        String[] tags = metricsEnum.getTags();
        if(tags != null && tags.length % 2 != 0) {
            throw new IllegalArgumentException(CLASS_NAME + "metrics tags must appear in pairs");
        }
    }
}
```

### 4.3 CounterMetricsUtil

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import io.micrometer.core.instrument.Counter;
import org.springframework.util.Assert;

import java.util.HashMap;
import java.util.Map;

public class CounterMetricsUtil extends BaseMetricsUtil {
    private static final Map<MetricsEnum, Counter> REGISTER_MAP = new HashMap<>();

    public static void register(MetricsEnum metricsEnum) {
        basicCheck(metricsEnum);
        Assert.isTrue(!REGISTER_MAP.containsKey(metricsEnum), "this metrics already register");

        Counter counter = Counter.builder(metricsEnum.getName())
                .tags(metricsEnum.getTags())
                .description(metricsEnum.getDescription())
                .register(meterRegistry);

        REGISTER_MAP.put(metricsEnum, counter);
    }

    public static void increment(MetricsEnum metricsEnum) {
        increment(metricsEnum, 1.0);
    }

    public static void increment(MetricsEnum metricsEnum, double value) {
        Counter counter = REGISTER_MAP.get(metricsEnum);
        if(counter != null) {
            counter.increment(value);
        }
    }
}
```

### 4.4 GaugeMetricsUtil

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import io.micrometer.core.instrument.Gauge;
import org.springframework.util.Assert;

import java.util.HashMap;
import java.util.Map;

public class GaugeMetricsUtil extends BaseMetricsUtil {
    private static final Map<MetricsEnum, Double> REGISTER_MAP = new HashMap<>();

    public static void register(MetricsEnum metricsEnum) {
        basicCheck(metricsEnum);
        Assert.isTrue(!REGISTER_MAP.containsKey(metricsEnum), "this metrics already register");

        Gauge.builder(metricsEnum.getName(), REGISTER_MAP, e -> e.get(metricsEnum))
                .tags(metricsEnum.getTags())
                .description(metricsEnum.getDescription())
                .register(meterRegistry);

        REGISTER_MAP.put(metricsEnum, 0D);
    }

    public static void gauge(MetricsEnum metricsEnum, double value) {
        REGISTER_MAP.put(metricsEnum, value);
    }
}
```

### 4.5 TimerMetricsUtil

```java
import java.util.concurrent.TimeUnit;

import com.github.jitwxs.metrics.enums.MetricsEnum;
import io.micrometer.core.instrument.Timer;
import org.springframework.util.Assert;

import java.util.HashMap;
import java.util.Map;

public class TimerMetricsUtil extends BaseMetricsUtil {
    private static final Map<MetricsEnum, Timer> REGISTER_MAP = new HashMap<>();

    public static void register(MetricsEnum metricsEnum) {
        basicCheck(metricsEnum);
        Assert.isTrue(!REGISTER_MAP.containsKey(metricsEnum), "this metrics already register");

        Timer timer = Timer.builder(metricsEnum.getName())
                .tags(metricsEnum.getTags())
                .description(metricsEnum.getDescription())
                .publishPercentiles(0.5, 0.8, 0.95, 0.99) // 指定百分位数
                .register(meterRegistry);

        REGISTER_MAP.put(metricsEnum, timer);
    }

    public static void record(MetricsEnum metricsEnum, long time, TimeUnit unit) {
        Timer timer = REGISTER_MAP.get(metricsEnum);
        if(timer != null) {
            timer.record(time, unit);
        }
    }
}
```

## 五、程序配置

### 5.1 applicaiton.yaml

编辑程序的配置文件，主要是 `management` 相关的配置。

- `management.server.port` 指定暴露的监控端点
- `management.endpoints.web.base-path` SpringBoot 程序监控默认的根路径是 `/actuator`，我嫌它麻烦，给改成 `/` 了
- `management.endpoints.web.exposure.include` SpringBoot 默认情况会将所有信息都暴露出去，这里我改成只暴露一部分，主要用的就是那个 `prometheus`【生产环境这些内容要么要加权限控制，要么尽量减少暴露部分，减少泄露信息的可能】

```yaml
management:
  server:
    port: 7002
  endpoints:
    web:
      base-path: /
      exposure:
        include: health, info, prometheus
spring:
  application:
    name: springboot_metrics

server:
  port: 8080
```

### 5.2 注册 MetricsEnum

前面提到我将所有的指标都注册到了 `MetricsEnum` 中，因此当服务启动时，我需要将它们都注册到 Metrics 里面。

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import com.github.jitwxs.metrics.support.CounterMetricsUtil;
import com.github.jitwxs.metrics.support.GaugeMetricsUtil;
import com.github.jitwxs.metrics.support.TimerMetricsUtil;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component
public class MetricsApplicationBoot implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 注册指标
        for (MetricsEnum metricsEnum : MetricsEnum.values()) {
            switch (metricsEnum.getType()) {
                case COUNTER:
                    CounterMetricsUtil.register(metricsEnum);
                    break;
                case GAUGE:
                    GaugeMetricsUtil.register(metricsEnum);
                    break;
                case TIMER:
                    TimerMetricsUtil.register(metricsEnum);
                    break;
                default:
                    break;
            }
        }
    }
}
```

### 5.3 开启定时任务注解

另外后续我将通过定时任务的方式，去模拟 Metrics 数据的产生，因此不要忘记在启动类上方加上注解：`@EnableScheduling`。

## 六、结语

至此完成 SpringBoot Metrics 监控系统框架的搭建，整个项目的文件结构如下图所示，另外访问 `http://127.0.0.1:7002/prometheus` 应当得到形如下图的数据。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201115213022.png)
