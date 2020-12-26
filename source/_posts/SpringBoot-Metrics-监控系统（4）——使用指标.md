---
title: SpringBoot Metrics 监控系统（4）——使用指标
copyright_author: Jitwxs
tags:
  - SpringBoot
  - Prometheus
  - Grafana
categories: 云原生
abbrlink: dad9ec18
date: 2020-11-15 21:34:35
related_repos:
  - name: springboot_metrics
    url: https://github.com/jitwxs/blog-sample/tree/master/SpringBoot/springboot_metrics
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、前言

在上一章节，我们已经完成了 SpringBoot Metrics 程序的框架搭建。在本章节中，我们将在程序中进行 Metrics 埋点，并能够被 Prometheus 采集到，且最终在 Grafana 中展示出来。

## 二、Metrics 埋点

### 2.1 Counter

先来介绍下最简单的 `Counter` 类型，它是不断递增的一种数据结构，你可以将其理解为**计数器**。

假设我们想要统计两部视频的阅读量，首先我们需要在 MetricsEnum 中注册下：

```java
READ_COUNT_1("read_count", new String[]{"video_name", "法外狂徒张三"}, "阅读量统计", COUNTER),
READ_COUNT_2("read_count", new String[]{"video_name", "不讲武德年轻人"}, "阅读量统计", COUNTER),
```

先给大家解释下这个枚举类的几个参数把：

- `name` Metrics 指标的名字
- `tags` 该 Metrics 对应的 label（在不同系统中叫法不一样，在 java 中就是叫 tag），必须成对出现，其实就是一组组键值对。
- `description` Metrics 指标的描述
- `type` 自定义的 Metrics 的类型，用于初始化的时候去 Prometheus 注册哪种 Metrics

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115214053.png)

有小伙伴可能会说，你这两个 metrics 怎么 name 都是一样的啊。这个其实没有关系，因为能够通过后面的 tag 来区分开。

举个例子：所有视频的阅读量统计，使用的都是同一个 metrics，即 `read_count`，如果我想要得到每一部视频的阅读量，我只需要筛选的时候加上 `video_name = xxx` 即可。可以把 metricsName 理解为一个 Map 对象，而其中每一组 tag 就是这个 Map 里面的一个 K-V。

解释完代码后，咱们来写一个定时任务，去模拟阅读量的递增：

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import com.github.jitwxs.metrics.support.CounterMetricsUtil;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.Random;

@Service
public class MockReadCountScheduler {

    @Scheduled(initialDelay = 100, fixedDelay = 1000)
    public void mockReadCount() {
        try {
            double value1 = new Random().nextDouble();
            Thread.sleep(new Random().nextInt(1000));
            CounterMetricsUtil.increment(MetricsEnum.READ_COUNT_1, value1);

            double value2 = new Random().nextDouble();
            Thread.sleep(new Random().nextInt(1000));
            CounterMetricsUtil.increment(MetricsEnum.READ_COUNT_2, value2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

尝试启动程序，访问 `http://127.0.0.1:7002/prometheus` ，你应该能够看到这两个 metrics 已经出现在其中了，并且随着每次刷新，其数值都会自增。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115220055.png)

### 2.2 Gauge

让我们关闭程序，继续学习 Gauge 吧。Gauge 官方名称叫做测量仪，Prometheus 每隔一段时间（称为采集间隔）就会访问一次 Metrics 端口，访问的这一刻这个值是多少就是多少。

因此使用 Gauge 的指标的数值可增可减，没有先后关系。方便用于统计系统的实时信息，例如访客数、每分钟阅读量等等。

假设我们想要统计系统此时的访客数，首先我们需要在 MetricsEnum 中注册下：

```java
VISITOR_SIZE("visitor_size", null, "系统访问量", GAUGE),
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115215612.png)

咱们来写一个定时任务，去模拟实时的访客数：

```java
@Service
public class MockVisitorSizeScheduler {
    @Scheduled(initialDelay = 100, fixedDelay = 1500)
    public void mockVisitorSize() {
        int visitorSize = new Random().nextInt(100);

        GaugeMetricsUtil.gauge(MetricsEnum.VISITOR_SIZE, visitorSize);
    }
}
```

运行程序后，一样能够找到这个指标。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115220132.png)

### 2.3 Timer

Timer 类型的指标，适合用于统计一些耗时，也能够方便的进行百分位统计。

这里初学者包括我刚刚用的时候，就会习惯用 Gauge 去做耗时统计而不是用 Timer 类型。Gauge 类型的特点是测量仪，Promethues 默认的采集时间是 60s，那么使用 Gauge 统计耗时其实就是统计每隔 15s 你的耗时。这其实就是一个非常粗粒度的采集，60s 间隔内的耗时 Prometheus 都没采集到。

而 Timer 不一样，它有一个显著特点是**客户端计算**。Prometheus 你不是 60s 采集我一次嘛，没关系，我在客户端内记录下你调用 Timer 的次数（count）、总和（sum）、最大值（max）。这样 Prometheus 采集到的就不同于 Gauge 的粗粒度的、单独的一个数据，而是一个最小粒度的数据。

解释完 Timer 和 Gauge 的区别后，开始我们的例子吧。

假设我们想要统计系统某接口的响应时间，首先我们需要在 MetricsEnum 中注册下：

```java
PORTAL_REQUEST_TIME("portal_request_time", new String[]{"url", "/userInfo"}, "前端请求耗时", TIME),
```

咱们来写一个定时任务，去模拟这个接口的响应时间变化：

```java
import com.github.jitwxs.metrics.enums.MetricsEnum;
import com.github.jitwxs.metrics.support.TimerMetricsUtil;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@Service
public class MockRequestTimeScheduler {
    @Scheduled(initialDelay = 100, fixedDelay = 1500)
    public void mockVisitorSize() {
        TimerMetricsUtil.record(MetricsEnum.PORTAL_REQUEST_TIME, new Random().nextInt(1000), TimeUnit.MILLISECONDS);
    }
}
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115233933.png)

## 四、连接 Prometheus

我们需要编辑 prometheus 配置文件，我试了下 Docker 的 Promethues 配置变更不支持热部署，所以需要先通过 Docker Dashboard 将 Prometheus 容器关闭。

然后编辑 `prometheus.yml` 文件，增加一个节点：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115225008.png)

这里多了一个 `metrics_path`，是因为 prometheus 默认采集的 `/metrics` 路径，而我们是用的 `/prometheus` 路径，所以要显式声明一下。或者修改程序的配置文件，加入如下两行也是一样的效果：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115223930.png)

【注：这里有个坑，因为我的 Prometheus 和 Grafana 都是用容器启动的，所以它们之间用 localhost 是可以连接的。但是我的 SpringBoot 应用跑在容器外，Prometheus 连接 SpringBoot 应用就必须用我物理机的 IP，不能用 localhost】

将 Prometheus 重新启动后，进入 `Status -> Targets` 等我们的服务变成 UP 就连接完毕了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115225332.png)

## 五、Grafana

下面我们尝试在 Grafana 中将我们的指标渲染出来。

### 5.1 Counter

这张图表将展示视频阅读量在 5 分钟内的增速：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115230117.png)

### 5.2 Gauge

这张图表将展示系统当前的访客数信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115230737.png)

### 5.3 Timer

这张图表将展示接口的平均耗时信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115231305.png)

这张图表将展示接口的百分位耗时信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202011/20201115234750.png)

## 六、结尾

至此完成了在程序中进行 Metrics 埋点，并结合 Prometheus 和 Grafana，在大盘中可视化展示出来。Grafana 的可配置项很多，还需要大家自己去尝试和摸索。
