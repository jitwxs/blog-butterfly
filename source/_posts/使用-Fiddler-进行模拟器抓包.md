---
title: 使用 Fiddler 进行模拟器抓包
tags: 抓包
categories: 测试
abbrlink: f6ed356a
date: 2020-10-25 16:52:52
---

`Fiddler` 是一款比较常用的网络抓包工具，本文记录下使用其完成对模拟器中请求进行抓包的过程。

[点击链接](https://www.telerik.com/download/fiddler) 官网下载 Fiddler，可能需要翻墙。下载完毕后，常规安装并登录后打开应用。

1. 点击右上角⚙图标，进入设置页面

2. 选择 `HTTPS` 选项卡，点击 `Trust root certficarte` ，获取 root 权限。获取完毕后，勾选下方复选框，以支持对 HTTPS 协议的请求。
3. 展开 `Advanced Settings`，将 `.crt` 证书导出至桌面，后续需要在模拟器中安装。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025165959.png)

4. 选择 `Connections` 选项卡，可以指定监听端口，直接使用默认的 `8866` 即可。
5. 勾选下方 `Allow remote computers to connect`，我们的模拟器才能连接上来。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025170340.png)

至此完成 Fiddler 侧设置，下面安装模拟器。随便选择一家模拟器即可，我这里使用夜神模拟器进行演示。

1. 确认模拟器开启了 `root`，我这里默认就已经是开启状态了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025170851.png)

2. 进入系统 `WLAN` 设置中，鼠标左键长按已连接的 wifi，在弹出菜单中选择 `修改网络`。
3. 展开 `高级选项`，将 `代理` 设置为手动，在下方填入代理服务器的IP和端口号。
   - IP 为运行模拟器的宿主机 IP（使用 `ipconfig` 或 `ifconfig` 命令查看）
   - 端口为 Fiddler 的监听端口，即 8866

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025171321.png)

设置完毕后，尝试在模拟器中打开浏览器，如果一直弹出如下弹窗，我们还需要设置下证书。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025171731.png)

4. 基本所有模拟器都支持宿主机与模拟器之间的文件拷贝，我们需要将前文中导出到桌面的 `.crt` 文件拷贝到模拟器中。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025171951.png)

5. 进入 `设置 -> 安全 -> 从 SD 卡安装`，选择上一步中拷入的文件，设置好名字后，点击确认即可。（会提示你给模拟器设置一个密码，正常设置即可）

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025172228.png)

6. 如果安装完毕后，进入浏览器还是弹证书有问题的框，一直点“确定”忽略即可。

至此完成了模拟器侧的配置，在 Fiddler 首页选择 `Live Traffic` 选项卡，将右侧的滑动按钮点开，即启动了抓包。此时在模拟器中发起网络请求就可以显示出来了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202010/20201025172727.png)

当完成抓包后，记得将  `Live Traffic` 选项卡右侧的滑动按钮关闭，否则可能会影响你正常访问浏览器。
