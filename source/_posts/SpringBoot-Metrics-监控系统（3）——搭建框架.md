---
title: SpringBoot Metrics 监控系统（3）——搭建框架
tags:
  - SpringBoot
  - Prometheus
  - Grafana
categories: 云原生
abbrlink: 527f9dce
date: 2020-11-15 21:06:13
related_repos:
  - name: metrics-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/springboot-sample/metrics-sample
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

对于刚刚入门使用的同学，经常会出现 Metrics 指标的管理混乱的情况。在代码里这处注册一个指标，那处注册一个，很混乱。因此有必要提供一个类，专门管理所有的 Metrics 指标并统一注册。这里我使用枚举类实现，当然你也可以根据自己需要使用其他方式实现。

### 3.1 IMetricsEnum

Metrics 监控指标枚举，定义了指标的类型和名称。

```java
public interface IMetricsEnum {
    enum Type {GAUGE, COUNTER, TIMER}

    String getName();

    Type getType();

    String getDesc();

    default IMetricsTagEnum createVirtualMetricsTagEnum(String[] tags) {
        IMetricsEnum iMetricsEnum = this;
        return new IMetricsTagEnum() {
            @Override
            public IMetricsEnum getMetricsEnum() {
                return iMetricsEnum;
            }

            @Override
            public String[] getTags() {
                return tags;
            }
        };
    }
}
```

### 3.2 IMetricsTagEnum

负责维护所有的 Metrics 指标，是实际生效的 Metrics 指标。相较于 IMetricsEnum，需要额外指定 tag 属性。

```java
public interface IMetricsTagEnum {
    String FUNCTION = "function";

    IMetricsEnum getMetricsEnum();

    String[] getTags();
}
```

## 四、Metrics Support

首先编写一些使用 Prometheus 的 Metrics 的工具类，一共有以下几个类：

- `MetricsRegisterConfig` 交于 Spring 容器管理，负责获取到 Prometheus 的全局实例对象
- `BaseMetricsUtil` 基础的 Metrics 工具类，抽取一些共用方法
- `MetricsUtil` Metrics 注册和各个类型指标值记录的工具类

### 4.1 MetricsRegisterConfig

```java
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
@Slf4j
public class BaseMetricsUtil {
    public static MeterRegistry meterRegistry;

    protected static String CLASS_NAME = new Object() {
        public String getClassName() {
            String clazzName = this.getClass().getName();
            return clazzName.substring(0, clazzName.lastIndexOf("$"));
        }
    }.getClassName();

    public static boolean basicCheck(final IMetricsTagEnum metricsTagEnum) {
        if (meterRegistry == null) {
            log.warn("metrics registry is null,class={}", CLASS_NAME);
            return false;
        }

        final String[] tags = metricsTagEnum.getTags();

        if (tags != null && tags.length % 2 != 0) {
            log.error("metrics count error,class={},tags={}", CLASS_NAME, tags);
            return false;
        }

        return true;
    }
}
```

### 4.3 MetricsUtil

```java

@Slf4j
public class MetricsUtil extends BaseMetricsUtil {
    private static final Map<IMetricsTagEnum, MetricsWrapper> METRICS_MAP = new HashMap<>();

    static {
        ThreadPoolUtil.newScheduledExecutor(1, "metrics-manager-thread-pool").scheduleWithFixedDelay(() -> METRICS_MAP.values().stream()
                .filter(e -> e.getType() != IMetricsEnum.Type.COUNTER).filter(e -> TimeUtils.diffMs(e.getLastTime()) > 10_000)
                .forEach(e -> e.recordTimerOrGauge(0L)), 15, 15, TimeUnit.SECONDS);
    }

    /**
     * For IMetricsTagEnum
     */
    public static <T extends IMetricsTagEnum> void init(Class<T> clazz) {
        try {
            Method method = clazz.getDeclaredMethod("values");
            simpleRegister((T[]) method.invoke(null));
        } catch (final Exception e) {
            log.error("metrics gauge error,class={}", clazz.getSimpleName(), e);
        }
    }

    /**
     * For RingBuffer
     */
    public static void registerGauge(final IMetricsTagEnum metricsTagEnum,
                                     final Supplier<Number> supplier) {
        if (!basicCheck(metricsTagEnum)) {
            return;
        }

        Gauge.builder(metricsTagEnum.getMetricsEnum().getName(), supplier)
                .tags(metricsTagEnum.getTags())
                .description(metricsTagEnum.getMetricsEnum().getDesc())
                .register(meterRegistry);
    }

    /**
     * For Common
     */
    public static void simpleRegister(IMetricsTagEnum... metricsTagEnums) {
        if (metricsTagEnums != null && metricsTagEnums.length > 0) {
            for (IMetricsTagEnum metricsTagEnum : metricsTagEnums) {
                if (!basicCheck(metricsTagEnum)) {
                    continue;
                }

                final IMetricsEnum.Type type = metricsTagEnum.getMetricsEnum().getType();

                if (type == IMetricsEnum.Type.GAUGE) {
                    Gauge.builder(metricsTagEnum.getMetricsEnum().getName(), METRICS_MAP, m -> (long) m.get(metricsTagEnum).getMetrics())
                            .tags(metricsTagEnum.getTags())
                            .description(metricsTagEnum.getMetricsEnum().getDesc())
                            .register(meterRegistry);

                    METRICS_MAP.put(metricsTagEnum, MetricsWrapper.newInstance(type, 0L));
                } else if (type == IMetricsEnum.Type.COUNTER) {
                    final Counter cnt = Counter.builder(metricsTagEnum.getMetricsEnum().getName())
                            .tags(metricsTagEnum.getTags())
                            .description(metricsTagEnum.getMetricsEnum().getDesc())
                            .register(meterRegistry);

                    METRICS_MAP.put(metricsTagEnum, MetricsWrapper.newInstance(type, cnt));
                } else if (type == IMetricsEnum.Type.TIMER) {
                    final Timer timer = Timer.builder(metricsTagEnum.getMetricsEnum().getName())
                            .tags(metricsTagEnum.getTags())
                            .description(metricsTagEnum.getMetricsEnum().getDesc())
                            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
                            .register(meterRegistry);

                    METRICS_MAP.put(metricsTagEnum, MetricsWrapper.newInstance(type, timer));
                }
            }
        }
    }

    /**
     * For Counter
     */
    public static void recordCounter(IMetricsTagEnum metricsTagEnum) {
        recordCounter(metricsTagEnum, 1.0D);
    }

    /**
     * For Counter
     */
    public static void recordCounter(IMetricsTagEnum metricsTagEnum, double size) {
        if (METRICS_MAP.containsKey(metricsTagEnum)) {
            METRICS_MAP.get(metricsTagEnum).recordCounter(size);
        }
    }

    /**
     * For Timer、Gauge
     */
    public static void recordTimerOrGauge(final IMetricsTagEnum metricsTagEnum, final long value) {
        if (METRICS_MAP.containsKey(metricsTagEnum)) {
            METRICS_MAP.get(metricsTagEnum).recordTimerOrGauge(value);
        }
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class MetricsWrapper {
        private IMetricsEnum.Type type;

        private Object metrics;

        private long lastTime = TimeUtils.nowMs();

        public static MetricsWrapper newInstance(IMetricsEnum.Type type, Object metrics) {
            return new MetricsWrapper(type, metrics, TimeUtils.nowMs());
        }

        private void recordCounter(final double value) {
            ((Counter) this.metrics).increment(value);
            this.lastTime = TimeUtils.nowMs();
        }

        public void recordTimerOrGauge(final long value) {
            if (this.type == IMetricsEnum.Type.TIMER) {
                ((Timer) this.metrics).record(value, TimeUnit.MILLISECONDS);
            } else if (this.type == IMetricsEnum.Type.GAUGE) {
                this.metrics = value;
            }
            this.lastTime = TimeUtils.nowMs();
        }
    }
}
```

## 五、applicaiton.yaml

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
    name: metrics-sample
```

## 六、结语

至此完成 SpringBoot Metrics 监控系统框架的搭建，下一节将开始演示指标的使用。
