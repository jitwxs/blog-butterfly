---
title: SpringBoot Metrics 监控系统（2）——快速起步
tags:
  - SpringBoot
  - Prometheus
  - Grafana
categories: 云原生
copyright_author: Jitwxs
abbrlink: 1e8d61b4
date: 2020-11-14 16:45:49
---

## 一、Docker

首先需要安装 Docker，已经安装的朋友直接跳过该节即可。

Docker 最近新出了 Docker Desktop，可以对容器和镜像可视化管理，还是很不错的。访问[官网](https://www.docker.com/get-started)

下载即可，这里我使用 Windows 平台进行安装。【最好采用科学上网，否则速度会很感人】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114164928.png)

如果安装完毕后打开报下图的错，需要在更新下 WSL2，[点此下载](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114165548.png)

启动成功后如下图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114170114.png)

## 二、Prometheus + Grafana

打开 PowerShell 或其他命令行工具，下载镜像：

```bash
docker pull prom/prometheus
```

下载完毕后，提前准备好配置文件，新建配置文件 `prometheus.yml`，内容如下。将其保存到任一位置即可，并记录下它的绝对路径。我这里把它放置在了桌面，因此它的绝对路径是：`C:\Users\Jitwxs\Desktop\prometheus.yml`。

```yaml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
```
执行命令启动容器【注意命令中的绝对路径，需要替换成你自己实际的路径】：

```bash
docker run  -d -p 9090:9090 -v C:\Users\Jitwxs\Desktop\prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

执行完毕后，打开 Docker 客户端，可以看到 Prometheus 已经启动成功，在 `Mounts` 中可以看到配置文件也被挂载到容器内。 

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114184154.png)

打开浏览器，输入 `http://127.0.0.1:9090` 即可登录 Prometheus，选择 `Status -> Configuration` 也能看到我们的配置文件。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114184330.png)

至此 Prometheus 的起步就完成了，下面来启动 Grafana 把。

## 三、Grafana

下载镜像：

```bash
docker pull grafana/grafana
```

启动容器：

```bash
docker run -d -p 3000:3000 --name=grafana grafana/grafana
```

执行完毕后，打开 Docker 客户端，可以看到 Grafana 已经启动成功。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114190856.png)

打开浏览器，输入 `http://127.0.0.1:3000`，即可登录 Grafana 首页，使用默认用户名密码 admin 登录即可。

首先配置下 Grafana 的数据源，点击右侧⚙图标，选择 `Data Sources` 选项卡，然后选择 右侧的 Prometheus。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114191115.png)

并做以下配置：

- URL: `http://127.0.0.1:9090`
- Access: `Browser`

然后点击下方 `Save & Test` 按钮，弹出 `Data source is working` 的绿色提示即可。

【注意：如果弹出的是红色的`HTTP Error Bad Gateway`，尝试将 Access 改为 `Server` 试试看】

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114191345.png)

至此完成 Grafana 的起步。

## 四、入门实例

下面展示一个简单的心跳监测的功能，Prometheus 服务自身也会对外暴露一些 Metrics，下面来实现如何在 Grafana 上侦听 Prometheus 服务是否存活。

我们在 Prometheus 的首页上，搜索一个名为 `up` 的 metrics，搜索结果中有一条，它有两个 label，通过 `instance` 这个 label 得知这条记录就是 prometheus 提供的。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114233624.png)

切换到 Grafana 上，点击左侧的加号，点击 Dashboard，会创建一个 Dashboard。然后我们选择 `Add new panel`，新建一个面板。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114233824.png)

在下面的 PromQL 表达式上输入刚刚的 metrics，然后将图表从默认的 `Graph` 改为 `Stat`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234006.png)

对于 `up` 这个 metrics，可能的取值有 1 和不存在。我们可以通过 Value Mapping 功能，将取值和实际的展示的内容作对应。同时对于不同的取值，也可以指定配色。

如下图所示，我将 1 映射为 UP，将 null 映射为 DOWN。同时，将 1 颜色指定为绿色，0 指定为红色。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234417.png)

另外图表下方的 Graph 展示了这个数值在不同时间的变化情况，假设我们只想关心当前的，不想关心这个服务什么时候 UP 什么时候 DOWN，可以将其隐藏掉。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234700.png)

至此这个 Panel 就制作好了，返回到 Dashboard 后形如下图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234720.png)

让我们尝试关闭掉 prometheus 服务

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234820.png)

点击 Grafana Dashboard 右上角的刷新后，发现变成了 DOWN。将 Prometheus 服务恢复后，又恢复正常。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20201114234911.png)
