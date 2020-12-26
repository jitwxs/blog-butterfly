---
title: HoloLens 开发笔记（1）——HelloWorld
categories: HoloLens
abbrlink: e42fdabb
date: 2018-11-19 15:38:31
copyright_author: Jitwxs
---

HoloLens是微软在2015年推出的一款混合现实（MR）眼镜，官方网站[点击这里](https://www.microsoft.com/zh-cn/hololens)。这个眼镜具体能干啥，文字表述总是太过乏力，下面给出一些相关视频，帮助大家感受下。

1. [Microsoft HoloLens: Build 2016 Keynote](https://www.youtube.com/watch?v=mM1P41qeVGs)
2. [TED - A futuristic vision of the age of holograms](https://www.ted.com/talks/alex_kipman_the_dawn_of_the_age_of_holograms)
3. [科技美学 - 微软 HoloLens 开发者体验 混合现实](https://www.bilibili.com/video/av6187892)
4. [没玩过微软的这个产品还敢说你了解高科技？——微软HoloLens](https://www.bilibili.com/video/av6455205)

## 一、环境搭建

| 名称          | 环境                                                      |
|:------------- |:--------------------------------------------------------- |
| 操作系统      | Win10 专业版 1703【请确保**至少保证为Win10**，推荐1703+】 |
| Visual Studio | Visual Studio Community 2017                              |
| Unity         | Unity 2018.2                                              |

首先请熟悉HoloLens的基本操作，可以参考说明书，或者文章开头的三方体验视频，或者自己百度啊啥的，然后在PC上搭建VS + Unity的开发环境。

## 二、Mixed Reality Unity 项目

本章将演示一个简单的使用Unity实现的混合现实应用，基于教程：[MR Basics 100: Getting started with Unity](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-100)

### 2.1 创建项目

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153119561.png)

1. 点击右上角New按钮，然后你会看到如图的设置项
2. 请指定项目名和项目位置
3. 请确保选择3D模式
4. 点击创建项目按钮

### 2.2 设置相机

1. 进入主页面后，选中最左侧的`Main Camera`，或者点击中间窗口的相机图标，进入到相机的设置。
2. 点击右侧的`Inspector`，进入到相机的详细设置。

3. 将相机移动到坐标轴的起始位置，将`position`的坐标轴改为**(X:0 Y:0 Z:0)**。
4. 将`Clear Flags`从SkyBox（天空盒）更改为**Solid Color**（纯色）。
5. 将下面的`Background`颜色修改为黑色，即RGBA为**(0, 0, 0, 0)**。
6. 修改Clipping Planes的 `Near` 的值为 **0.85**，防止当用户接近一个对象时，对象被渲染到离用户眼镜太近的位置。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153145584.png)

>在 HoloLens 应用中，摄像机位置就是用户的头部的位置，摄像机就代表了用户的眼镜。
>
>当场景中存在多个摄像机时，Unity会使用 `MainCamera` 标签来确定使用哪个摄像机来进行立体世界渲染。

### 2.3 切换项目平台

点击菜单栏 `File-> Build Setting`，或者快捷键 `Ctr + Shift + B` ，点击`Universal Windows Platform`平台，然后点击左下角的`Switch Platform`。

如果该平台没有安装，选择右边的`open download`按钮，下载对应的平台插件。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153200480.png)

### 2.4 更改项目设置

点击菜单栏 `Edit -> Project Settings -> Quality`，在右侧的`Inspector`中，点击windows图标下的Default的下拉框，如图所示，将值修改为`Very Low`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222142923568.png)

点击菜单栏`Edit -> Project Settings -> Player`，选中`Windwos`图标选项卡，选中`XR Settings`，勾选`Virtual Reality Supported`，会看到下面有一项**Windows Mixed Reality**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153230991.png)

选中`Other Settings`，将`Scripting Backend`的值修改为**.NET**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153254780.png)

### 2.5 添加立方体

1. 在左侧项目列表空白处，右击选择`3D Object -> Crub`新建一个立方体。
2. 修改立方体坐标为**(X:0 Y:0 Z:2)**，更改旋转为**(X:45 Y:45 Z:45)**，将立方体缩放为**0.25**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153443757.png)

至此整个项目结构如图所示：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153454671.png)

## 三、运行程序

### 3.1 HoloLens设置

戴上HoloLens眼镜，在HoloLens的Windos Store商店中，下载应用[Holographic Remoting Player](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holographic-remoting-player)，并进入，此时应该出现HoloLens的IP地址信息，形如：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153509315.png)

确保你的PC与HoloLens在同一网络下。

> 也可以使用USB连接，参考官方：[原文链接](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-100#for-other-mixed-reality-supported-headsets)：
>
> ### For other mixed reality supported headsets
>
> 1. Connect the headset to your development PC using the USB cable and the HDMI or display port cable.
> 2. Launch the **Mixed Reality Portal** and ensure you have completed the first run experience.
> 3. From Unity, you can now press the Play button.

### 3.2 从Unity部署

点击菜单栏的 `Window -> XR -> Holographic Emulation`，在弹出框选中 `Remote to Device`，填入HoloLens的IP地址，点击连接。

连接成功后，点击启动图标，启动程序，再点击下关闭程序，如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153521912.png)

当程序启动后，你就能通过HoloLens看见一个立方体了！Hello World！

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153542305.jpg)

### 3.3 从Visual Studio构建并部署到设备

现在我们准备将项目编译到Visual Studio并部署到目标设备。

#### 3.3.1 导出为 VS 解决方案

（1）点击菜单栏`File->Build Settings`，确保当前项目已经被添加到`Scenes In Build`，如果没有，请点击`Add Open Scenes`按钮。

（2）修改`Target Device`为HoloLens；`Build Type`为D3D；`SDK`选择**Latest installed**即可，我这里指定了10.0的版本；VS版本我制定了VS 2017；勾选下方的**Unity C# Projects**按钮，点击Build按钮。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153604307.png)

（3） 在弹出的目录中新建文件夹APP。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153614670.png)

（4）进入到App文件夹下，点击选择文件夹按钮。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153624287.png)

（5）Build完成后，点击App目录下的.sln文件，在VS中打开项目。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153640563.png)

#### 3.3.2 运行解决方案

>具体参考官方：https://docs.microsoft.com/zh-cn/windows/mixed-reality/using-visual-studio

点击右上方选择为`Release`，平台选择`x86`，选择`远程计算机`，然后会弹出一个如图的远程连接对话框，在其中填入HoloLens的IP地址，点击`选择`按钮。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/2018111915365271.png)

点击菜单栏`调试->开始执行（不调试）`按钮，或按快捷键 `Ctrl + F5`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153702626.png)

如果没有异常，经过一段漫长的等待（取决于网速、电脑配置，我用了3分钟吧），在HoloLens中就可以看到立方体了。

## 四、问题

**（1）关于textmeshpro的错误**

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153713228.png)

请将刚刚创建的App文件夹整个删掉，在Unity中，点击菜单栏`Window > Package Manager`，移除**TextMesh Pro**，如图所示，然后重新导出解决方案即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119153722513.png)

> 解决方案来源：https://github.com/MicrosoftDocs/mixed-reality/issues/557

**（2）要求输入Pin码**

>参考官方，[原文链接](https://docs.microsoft.com/zh-cn/windows/mixed-reality/using-visual-studio#pairing-your-device-hololens)
><br/>The first time you deploy an app from Visual Studio to your HoloLens, you will be prompted for a PIN. On the HoloLens, generate a PIN by launching the Settings app, go to **Update > For Developers** and tap on **Pair**. A PIN will be displayed on your HoloLens; type this PIN in Visual Studio. After pairing is complete, tap **Done** on your HoloLens to dismiss the dialog. This PC is now paired with the HoloLens and you will be able to deploy apps automatically. Repeat these steps for every subsequent PC that is used to deploy apps to your HoloLens.
><br/>To un-pair your HoloLens from all computers it was paired with, launch the **Settings** app, go to **Update > For Developers** and tap on **Clear**.
