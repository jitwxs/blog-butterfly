---
title: HoloLens 开发笔记（3）——Windows Device Portal
categories: HoloLens
abbrlink: b9c9d468
date: 2018-11-22 11:43:29
copyright_author: Microsoft Docs
copyright_url: https://docs.microsoft.com/en-us/windows/mixed-reality/develop/advanced-concepts/using-the-windows-device-portal
---

Windows设备控制台允许你通过 Wi-Fi 或 USB 来远程控制你的 HoloLens 设备。设备控制台是 HoloLens 上的一个 Web Server，你可以通过PC的浏览器来连接到它。设备控制台包含了很多帮助你管理、调试和优化 HoloLens 设备的工具。

## 一、设置 HoloLens 以使用 Windows Device Portal

1. 打开 HoloLens，并穿戴上
2. 使用绽开手势打开开始菜单
3. 选中设置应用，在你放置它以后会自动启动
4. 选中更新选项
5. 选中开发者选项
6. 打开开发者模式
7. 滑动页面，打开设备控制台选项

![](https://docs.microsoft.com/zh-cn/windows/mixed-reality/images/deviceportalsettings.png)

## 二、通过 Wi-Fi 连接

1. 将 HoloLens 连上Wi-Fi
2. 找到你的 IP 地址
3. 在PC浏览器上前往 `https://<你设备的IP>`
   - 浏览器会显示以下信息，“浏览器的证书存在问题”。这是因为Windows设备控制台的证书是测试证书，你现在可以忽略这个证书错误。

## 三、通过 USB 连接

1. 安装好开发工具，确保PC上已有 Visual Studio 2015 Update 1及更新版本和 Windows 10 开发者工具。这保证了USB连接性。
2. 将 HoloLens 设备通过USB连接到PC
3. 在PC浏览器上前往 `http://127.0.0.1:10080`

## 四、连接到模拟器

你也可以在模拟器上使用设备控制台。可以使用 toolbar 连接到设备控制台。点击下面这个图标： **Open Device Portal**： 打开 HoloLens 模拟器的设备控制台。

![](https://docs.microsoft.com/zh-cn/windows/mixed-reality/images/emulator-deviceportal.png)

## 五、创建用户名和密码

你首次连接到 HoloLens 上的设备控制台时，需要创建一个用户名和密码。

1. 在PC浏览器上访问 HoloLens 的IP地址，会打开一个设置页面
2. 点击**Request pin**，然后在 HoloLens 上查看生成的pin码
3. 输入设备上出现的 pin 码
4. 输入一个用户名用于连接 HoloLens，不必是微软账户或者域账号
5. 重复输入密码，密码至少要有7个字符。不必是微软账号或者域账号密码
6. 点击 Pair按钮来连接到 HoloLens

任何时候如果你想修改用户名和密码，你可以点击页面顶部Security链接访问设备安全页面，或者直接访问：`https://<YOUR_HOLOLENS_IP_ADDRESS>/devicesecurity.htm`。

![](https://docs.microsoft.com/zh-cn/windows/mixed-reality/images/windows-device-portal-credentials-reset-page-1000px.png)

## 六、安全证书

如果你在浏览器里看到证书错误提示，可以通过信任 HoloLens 设备证书来修复此问题。

每台 HoloLens 设备都会生成一个自签名的证书用于 SSL 连接。默认情况下，此证书不会被你的浏览器信任，并显示证书错误。通过下载此证书，并在PC上信任它，你就可以安全的连接到设备了。

1. 确保处在安全的网络下
2. 从设备控制台安全（Security）页面下载设备证书
   - 单击右上角图标列表中的安全链接或导航到: `https://<YOUR_HOLOLENS_IP_ADDRESS>/devicesecurity.htm`
3. 安装证书到PC上的“受信任的信任根证书发行机构（Trusted Root Certification Authority）”目录
   - 在Windows菜单中， type: Manage Computer Certificates and start the applet
   - 展开 Trusted Root Certification Authority 文件夹
   - 从操作菜单中，选择:All Tasks > Import...
   - 使用从设备门户下载的证书文件完成证书导入向导。
4. 重启浏览器

## 七、设备控制台页面

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222105548198.png)

设备管理会话起始于首页。从左边导航栏点击Home即可进入首页。

顶部工具栏提供了设备状态和一些特性内容。

- Online：指示设备是否连接到了Wi-Fi
- Power：电源，包含 Shutdown 和 Restart
- Cool：指示设备温度
- 电量信息
- Help：打开使用帮助

首页显示了以下信息：

- 设备状态：监视设备健康及报告致命错误
- Windows 信息：显示HoloLens名字和当前系统版本
- 设备名：分配一个名字给设备，改名后必须重启后才能生效。

### 7.1 3D视图

![](https://docs.microsoft.com/zh-cn/windows/mixed-reality/images/3dview-1000px.png)

使用 3D 视图页面来了解 HoloLens 如何解析周围环境。使用鼠标可以调整视图内容：

- 旋转：按住鼠标左键移动
- 平移：按住鼠标邮件移动
- 缩放：滚动鼠标滚轮
- 追踪选项：通过勾选Force visual tracking打开持续可视化追逐。勾选Pause会暂停追踪。
- 视图选项：
  - Tracking：指示可视化追踪是否激活
  - Show floor：显示一个方格平面图
  - Show frustum：显示一个视锥
  - Show stabilization plane：显示HoloLens用于稳定运动的平面
  - Show mesh：显示周围环境的表面映射网格
  - Show details：显示实时变化时，手的位置，头部转动参数，以及设备初始矢量
  - Full screen按钮：全屏模式显示3D视图，按Esc键可退出
- Surface reconstruction：点击Update按钮会显示最新的空间映射网格，有时候这个过程可能会花费一点时间。3D视图中的空间网格不会自动更新，你必须手动点击更新按钮来从设备中载入最新的网格数据。点击保存按钮可以将当前空间映射网格保存为obj文件存储到PC上。

### 7.2 混合现实捕获

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110222976.png)

使用[混合现实捕获](https://developer.microsoft.com/zh-cn/windows/holographic/using_mixed_reality_capture)可以保存来自 HoloLens 设备的媒体流。

  - Holograms：捕获全息内容到视频流。全息图像已单声道渲染，而不是立体声
  - PV camera：从摄像头捕获视频流
  - Mic Audio：捕获麦克风阵列的声音
  - App Audio：捕获当前应用的声音
  - Live preview quality：为实时预览视频选择分辨率、帧率和流速
- 点击Live preview按钮来预览当前捕捉流内容。Stop live preview按钮用于停止预览捕捉流
- 点击Record按钮来开始使用指定设置来记录混合现实流。Stop recording用于结束纪录，并保存它
- 点击Take photo按钮从捕获流里获取一张照片
- Videos and photos：显示捕获的视频和照片列表

注意：当你从设备控制台纪录或实时预览捕获流时，HoloLens 应用将不能捕获 MRC 视频或者照片

### 7.3 性能追踪

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110339519.png)

用于从 HoloLens 捕获 Windows 性能记录器（WPR）追踪内容

- Available profiles：选择 WPR 配置后点击 Start 开始性能追踪
- Custom profile：点击 Browse 从 PC 选择一个 WPR 配置文件。点击 Upload and start 开始性能捕捉

为了停止性能追踪，点击 stop。停留在此页面直到性能追踪文件下载完成。捕获到的 ETL 文件可以被 [Windows性能分析器](https://msdn.microsoft.com/en-us/library/windows/hardware/hh448170.aspx) 打开并分析。

### 7.4 进程

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110435604.png)

显示当前运行进程的细节。包括了所有系统和应用进程。

### 7.5 系统性能

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110516658.png)

显示系统实时诊断图形信息，例如使用电量、帧速和 CPU 负载。

以下是可获得的内容指标：

- SoC 电源：平均每分钟瞬时系统芯片电量利用率
- System power：平均每分钟瞬时系统电量利用率
- Frame rate：每秒帧数，每秒丢失的空白帧数以及持续丢失的帧数
- GPU：GPU 引擎利用率
- I／O：读写速度
- Network：接收到和发出的流量大小
- Memory：总内存、使用中、修改的、分页的以及不分页的内存情况

### 7.6 应用

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110633929.png)

管理安装在 HoloLens 上的应用。

- Installed apps：移除和开始应用
- Running apps：列出当前正在运行的应用
- Install app：从电脑上选择应用包来安装
- Dependency：添加安装包依赖项
- Deploy：部署应用和其依赖项到 HoloLens

### 7.7 应用崩溃纪录页面 

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018122211081481.png)

这个页面允许你收集旁加载应用的崩溃日志。为每一个你想收集崩溃日志的应用选中 Crash Dump，然后返回此页面收集崩溃日志。dump 文件可以使用 Visual Studio 打开来 [调试](https://msdn.microsoft.com/en-us/library/d5zhxt22.aspx)。

### 7.8 文件浏览器

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222110909672.png)

使用文件资源管理器浏览、上传和下载文件。您可以在 Documents 文件夹、Pictures 文件夹和本地存储文件夹中处理从 Visual Studio 或设备门户部署的应用程序的文件。

### 7.9 Kiosk模式

开启 Kiosk 模式后，会限制用户启动新应用或者改变正在运行应用的能力。Bloom 手势和 Cortana 也将不能使用，环境中放置的其他应用也不会被显示。

选中 Enable Kiosk Mode 来使 HoloLens 进入 kiosk 模式。从 Startup app 里选择一个应用。点击 Save 来保存设定。

注意：即使 Kiosk 模式没有开启，应用也会在 HoloLens 启动时运行。选择 None 则没有应用会开机启动。

### 7.10 日志

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018122211094710.png)

管理HoloLens上的Windows实时事件追踪（ETW）。

选中Hide providers以仅显示事件列表

- Registered providers：选择 ETW 提供者和追踪级别。追踪级别会是以下其中之一：
  1. Abnormal exit or termination 异常退出和终止
  2. Servere errors 严重错误
  3. Warnings 警告
  4. Non-error Warnings 无错误警告
  5. Detailed trace 详细追踪

点击Enable按钮开始追踪。被追踪者将会被添加到 Enable Providers 下拉框。

- Custom Providers：选择一个自定义 ETW 来源喝追踪级别。通过 GUID 来标志提供者。GUID 不要包含括号
- Enable Providers：启动的 ETW 提供者来源
- Providers history：显示当前会话中被选中的 ETW 提供者
- Events：从选中的提供者以列表形式列出 ETW 事件
- Filters：允许你筛选通过 ID、关键词、级别、提供者名字、任务名字或文本收集的 ETW 事件

###  7.11 仿真

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222111137751.png)

允许你纪录和回放用于测试的输入数据。

- Capture room：用于下载一个包含用户周边环境空间映射网格数据的仿真房间文件，点击 Save 可以保存到本地计算机。房间文件可以导入到 HoloLens 模拟器使用。
- Recording：选中用于纪录的流，命名纪录后，开始进行纪录。在你的 HoloLens 上操作，然后点击 Stop 按钮将数据保存为 .xef 文件到 PC 上。此文件可以被 HoloLens 模拟器使用。
- Playback：点击 Upload recording 按钮从 PC 上选择一个 xef 文件，然后发送数据到 HoloLens 上。
- Control mode：从下拉框选择 Default或者Simulation，点击 Set 按钮在 HoloLens 上启用此模式。选中“Simulation”，将会禁用 HoloLens 上真实的传感器，而使用上传的模拟数据。如果启用 Simulation 模式，HoloLens 将不会响应真实用户直到切换回Default模式。

### 7.12 网络

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/201812221112573.png)

管理 HoloLens 上的 Wi-Fi 连接。

### 7.13 虚拟输入

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222111408270.png)

从远程机器发送键盘输入到 HoloLens 上。

点击 Virtual Keyboard 下方区域来放松键盘点击数据到 HoloLens 。在Input text中输入内容，然后点击Send按钮来发送内容到当前应用。

## 八、Device Portal REST API's

设备控制台里的所有内容都是基于 [REST API](https://developer.microsoft.com/zh-cn/windows/holographic/device_portal_api_reference) 制作的，你可以利用它们通过编程来自定义访问数据和控制你的设备。
