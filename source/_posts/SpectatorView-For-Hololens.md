---
title: SpectatorView For HoloLens
categories: HoloLens
abbrlink: c93fa54c
date: 2018-11-22 00:34:47
copyright_author: Microsoft Docs
copyright_url: https://docs.microsoft.com/en-us/windows/mixed-reality/design/spectator-view
---

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191212004034809.jpg)

戴上 HoloLens 时，我们通常会忘记这样一件事：没有戴上它的人无法体验到我们所能体验到的神奇感受。 SpectatorView（三方视角，旁观视图）允许其他人通过2D屏幕看到 HoloLens 用户在其世界里看到的东西。有了旁观视图，就可以通过快速且经济的方式在移动设备中录制高清全息影像。 有了它，还可以通过摄像机对全息影像进行专业质量的录制。

下表显示了不同的旁观视图功能。 请选择最符合你的视频录制需求的选项：

|              |           移动版     |                    摄像机                  |
| :---------------- | :------------------: | ------------------------------------------------ |
|      高清质量      |        全高清        | 专业质量摄影（取决于摄像机）                     |
|    相机移动轻松    |          ✔           | ✔                                                |
|    第三人称视图    |          ✔           | ✔                                                |
| 可以流式传输到屏幕 |          ✔           | ✔                                                |
|       可移植       |          ✔           |                                                  |
|        无线        |          ✔           |                                                  |
|    其他必需硬件    | Android 手机、iPhone | HoloLens + 支架 + 三脚架 + 摄像机 + 电脑 + Unity |
|      硬件投资      |          低          | 高                                               |
|       跨平台       |     Android、iOS     |                                                  |
|     同步的内容     |          ✔           | ✔                                                |
| 运行时设置持续时间 |         即时         | 慢                                               |

**重要资源：**

- [**GitHub 上的旁观视图**](https://github.com/microsoft/MixedReality-SpectatorView)
- [**旁观视图文档**](https://microsoft.github.io/MixedReality-SpectatorView/README.html)
- [**旁观视图示例**](https://github.com/microsoft/MixedReality-SpectatorView/tree/master/samples)

## 一、第三人称视角（Preview）

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdmlld3Bob25lZGVtby5qcGc?x-oss-process=image/format,png)

### 1.1 使用案例

- **拍摄高清全息影像**：使用SpectatorView (Preview)，你可以使用Iphone记录下混合现实的体验。记录下完整的高清图像并对全息影像应用反锯齿甚至是阴影，这是一种具有性价比且快速的方法来捕捉全息影像。

- **现场演示**：免费的通过你的Iphone或者Ipad直接以流的形式将混合现实体验推送到Apple TV。

- **与客人分享体验**：让没有HoloLens的用户直接通过他们的手机或平板来体验全息影像。

### 1.2 当前功能

- 网络能够自动发现添加到会话中的手机。

- 自动会话处理，一边将用户添加到正确的会话中。

- 全息影像的空间同步，让每个人看见同一个地方的全息影像。

- IOS支持（支持ARKit的设备）。

- 多个IOS客户。

- 视频录制 + 全息影像 + 环境音效 + 全息音效。

- 分享sheet以便你可以保存视频、发送右键，或者分享到其他支持的应用中。

**Note：**

SpetatorView (Preview) 的代码不能被用在 SpectatorView Pro 上，我们推荐在需要录制全息影像时在一个新的项目上实现它。

<center><iframe width="581" height="327" src="https://www.youtube.com/embed/tiXA9CW8iAs" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

### 1.3 许可证

- [OpenCV - (3-clause BSD License)](https://opencv.org/license.html)

- [Unity ARKit - (MIT License)](https://bitbucket.org/Unity-Technologies/unity-arkit-plugin/src/3691df77caca2095d632c5e72ff4ffa68ced111f/LICENSES/MIT_LICENSE?at=default&fileviewer=file-view-default)

## 二、如何设置第三人称视角（Preview）

### 2.1 必需品

- SpectatorView插件需要的OpenCV库在这里可以找到：[点击这里](https://github.com/Microsoft/MixedRealityToolkit/tree/master/SpectatorViewPlugin)。关于编译本地插件的详细信息在下面可以找到。从被生成的二进制你将需要：

   - opencv_aruco341.dll
   - opencv_calib3d341.dll
   - opencv_core341.dll
   - opencv_features2d341.dll
   - opencv_flann341.dll
   - opencv_imgproc341.dll
   - zlib1.dll
   - SpectatorViewPlugin.dll

- SpectatorView 使用Unity网络来进行网络发现和空间同步。这意味着应用程序期间的所有交互都需要在设备间进行同步。

- Unity 2017.2.1p2 或之后版本

- 硬件
   - 一个HoloLens
   - Win10 PC
   - 兼容 ARKit 的设备 (iPhone 6s +/ iPad Pro 2016 +/ iPad 2017 +)，运行IOS 11 + 系统

- Apple开发账户，免费的或者付费的，[点击这里](<https://developer.apple.com/>)

-  Visual Studio 2017

- **可选的：** UnityARKitPlugin。在MixedRealityToolkit-Unity项目中已经包含了这个插件所需要的组件。整个ARKit 插件可以从这里下载：[点击这里](<https://assetstore.unity.com/packages/essentials/tutorial-projects/unity-arkit-plugin-92515>)。

### 2.2 构建第三人称视角本地插件

跟着步骤来生成需要的文件，[点击这里](<https://github.com/Microsoft/MixedRealityToolkit/blob/master/SpectatorViewPlugin/README.md>)。

### 2.3 项目设置

1. 准备好你的场景，确保场景内的所有 gameobjects 都包含在一个根 gameobjects 下。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdmlld3Bob25ld29ybGRyb290Mi5wbmc?x-oss-process=image/format,png)

2. 添加SpectatorView prefab到你的场景下，[点击这里](https://github.com/Microsoft/MixedRealityToolkit-Unity/blob/master/Assets/HoloToolkit-Preview/SpectatorView/Prefabs/SpectatorView.prefab)。

3. 添加Add the SpectatorViewNetworking prefab到你的场景下，[点击这里](https://github.com/Microsoft/MixedRealityToolkit-Unity/blob/master/Assets/HoloToolkit-Preview/SpectatorView/Prefabs/SpectatorViewNetworking.prefab)。

4. 选择SpectatorViewNetworking gameobjects 链接到SpectatorViewNetworkingManager 组件上，如果没有该组件在运行时将寻找必要的脚本。
     -  标记检测 HoloLens -> `SpectatorView/HoloLens/MarkerDetection`
     -  标记生成 3D -> `SpectatorView/IPhone/SyncMarker/3DMarker`

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdmlld3Bob25lbmV0d29ya2Rpc2NvdmVyeS5wbmc?x-oss-process=image/format,png)

### 2.4 网络连接你的应用

- SpectatorView 使用Unity Networking来实现网络并且为你管理所有主机 - 客户端连接。

- 任何应用程序特定的数据都必须由你同步和实现。使用例如：SyncVars, NetworkTransform, NetworkBehavior。

- 关于更多Unity Networking的信息和教程，[点击这里](<https://unity3d.com/learn/tutorials/s/multiplayer-networking>)。

### 2.5 每个平台的构建（HoloLens 或 IOS）

- 当在iOS上构建时，请确保删除MRTK的GLTF组件，因为它还不兼容这个平台。

- 在SpectatorView prefab 的顶层有一个名为“Platform Switcher”的组件。选择您想要构建的平台。
     -  如果选择“HoloLens"，你应该会看到SpectatorView prefab中的Iphone prefab 下面的所有gameobject 都变成非活动，而"HoloLens"下所有的gameobjects 变为活动的。
     -  这可能需要一点时间，取决于你从平台上的项目中选择添加或者移除HoloToolkit。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdmlld3Bob25lc3dpdGNoZXIucG5n?x-oss-process=image/format,png)

- 确保你使用相同的Unity编辑器实例来构建应用程序的所有版本（构建时Unity没有紧密的统一），因为Unity有一个未解决的问题。

### 2.6 运行你的应用

一旦你已经在Iphone和HoloLens上构建并部署了应用的一个版本，你应该能够连接它们。

1. 确保这两个设备在同一Wifi网络下。

2. 在这两个设备上启动应用程序，没有特定的顺序。在Iphone上启动应用的过程中应该会触发Holoens摄像头打开并开始拍照。

3. 一旦Iphone应用启动，它将寻找诸如地板、桌子之类的表面(surfaces )。当表面找到时，你应该看到一个类似于下图的标记，将这个标记展示给HoloLens。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdmlld3Bob25lbWFya2VyLnBuZw?x-oss-process=image/format,png)

4. 一旦HoloLens已经检测到了标记，标记就会消失，两个设备应该已经连接并且空间同步了。

### 2.7 视频录制

1. 要从Iphone上捕捉并且保存视频，点击并按住屏幕1秒，这将打开录制菜单。

2. 点击红色的录制按钮，在开始录制屏幕前启动一个倒计时。

3. 要完成录制，点击并按住屏幕1秒，然后点击停止按钮。

4. 一旦录制的视频载入完成，一个蓝色的预览按钮就会出现，点击就可以观看录制的视频。

5. 打开分享sheet并且选择保存为相机胶卷。

### 2.8 示例场景

这里有一个场景例子，[点击这里](https://github.com/Microsoft/MixedRealityToolkit-Unity/blob/master/Assets/HoloToolkit-Examples/SpectatorView/Scenes/SpectatorViewExample.unity)。

### 2.9 故障排除

（1）**Unity编辑器不能连接到HoloLens设备托管的会话**

- 可能由于Windwos防火墙
- 前往Windows防火墙选项
- 允许应用程序或功能通过Windows防火墙
- 为Unity编辑器列表上所有实例，域，私有和公开。
- 然后返回Windows防火墙选项
- 点击高级设置
- 点击入站规则
- 所有Unity编辑器实例应该有一个绿色标记
- 对于没有绿色标记的Unity编辑器实例：
  - 双击它
  - 在动作对话框中选择允许连接
  - 点击OK
  - 重启Unity编辑器

（2）**Iphone运行时屏幕显示"Locating Floor..."，但是没有显示AR标记**

- Iphone正寻找一个水平的表面，因此尝试将Iphone的摄像头只向地面或桌子，这将帮助 ARKit 寻找到HoloLens空间同步所必需的表面。

（3）**当Iphone试图加入时，HoloLens相机不能自动打开**

- 确保HoloLens和Iphone在同一个Wifi网络下。

（4）**当在移动设备上运行SpectatorView应用时，运行其他SpectatorView应用的HoloLens打开了他们摄像头**

- 前往 `NewDeviceDiscovery` 组件，并且修改 Broadcast Key 和 Broadcast port 为两个唯一的值。
- 转到 `SpectatorViewDiscovery` 并且更改 Broadcast Key 和 Broadcast port 为另一组唯一的数字。

（5）**HoloLens不会与Mac上的Unity编辑器连接**

这是一个跟Unity版本相关的已知问题，构建到Iphone上仍然可以正常工作和连接。

（6）**HoloLens相机打开但是不能扫描标记**

确保你构建的应用的所有版本使用一个相同的Unity编辑器实例（构建时Unity没有紧密的统一），这是由于Unity一个未知的问题。

## 三、第三人称视角Pro

![SpectatorView Pro setup](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdGF0b3J2aWV3LTMwMHB4LnBuZw?x-oss-process=image/format)

使用 SpectatorView Pro 涉及下面四个组件：

1. 一款专门为用户观看而开发的应用，这是基于[shared experiences in mixed reality](https://docs.microsoft.com/zh-cn/windows/mixed-reality/shared-experiences-in-mixed-reality)。

2. 使用该应用的用户戴上HoloLens。

3. 提供第三人称视角视频的是一台观众视角相机。

4. 运行共享体验应用的并且将全息影像合称为观众视角视频的桌面PC。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9ob2xvbGVuc3NwZWN0YXRvcnZpZXctNTAwcHguanBn?x-oss-process=image/format,png)

### 3.1 使用案例

<center><iframe width="581" height="327" src="https://www.youtube.com/embed/DgIHjxoPy_c" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

![SpectatorView Pro photo capture scenario example](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9mYWxsLTM1MHB4LmpwZw?x-oss-process=image/format)

![SpectatorView Pro video capture scenario example](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdGF0b3J2aWV3dmlkZW8uZ2lm)

有三个关键场景可以很好的应用这项技术：

1. **照片捕获**：使用这项技术，你可以捕获高分辨率的全息影像像。这项图片可以用于在营销活动中作为橱窗内容，发送给你的潜在客户，甚至于将你的应用提交到Windows商店。你可以决定使用哪一款相机用于拍摄这些照片，因此你可能更喜欢高质量的单反相机。

2. **现场演示**：当相机的位置保持稳定或可控时，三方视角是现场演示的首选途径。因为你可以使用高质量的摄像机，你也可以为大屏幕制作高质量的图片。这也适用于在屏幕上实时演示，也许是为了让那些热切的参与者也可以排队等待参与。

3. **视频捕获**：视频是与许多人分享全息应用体验的最佳故事讲述机制。三方视角允许你选择最适合你想要展示应用程序的相机、镜头和框架，它让你根据你现有的视频硬件来控制视频质量。

### 3.2 视频捕获技术的比较

`混合现实捕捉（MCR）`提供了HoloLens用户从第一人称视角所看到的视频合成。三方视角(Spectator view)从第三人称视角生成视频，允许观察者以全息影像的形式看到戴着HoloLens设备用户的环境。根据你选择的相机，三方视角可以产生更高分辨率和更好质量的图像相对于HoloLens自己相机所使用的MCR图像。因此，三方视角更适合作为Windwos商店的应用展示图片、营销视频或对观众放映现场视图。

![SpectatorView Pro professional camera used in Microsoft keynote presentations](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdGF0b3Itdmlldy1wcm9mZXNzaW9uYWwtcmVkLWNhbWVyYS0zMDBweC5qcGc?x-oss-process=image/format)

从2015年1月微软HoloLens首次发布产品时起，三方视角已经是微软HoloLens向观众展现其体验的重要部分。专业设置需要很高的要求和昂贵的价格来支持。例如，相机使用 genlock 信号来确保与 HoloLens 跟踪系统协调的精确定时。在此设置中，在保持全息影像稳定的同时，移动三方视角相机是可能的，该全息影像与直接用HoloLens的人的体验相匹配。

开源版本的三方视图放弃了移动摄像头的能力，以大幅降低整体设置的成本。这个项目使用一个固定安装在HoloLens眼镜上的外部相机来拍摄你的全息Unity项目上的高清图片和视频。**在现场演示时，相机应该保持在一个固定的位置。** 移动相机可能导致全息影像的抖动和漂移。这是因为视频帧的校时和PC上全息影像的渲染可能不完全同步。因此，保持相机的稳定或限制移动将带来和佩戴HoloLens的人相近的结果。

为了让你的应用准备好三方视角，你需要构建一个[共享体验](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-240)的应用并确保它可以在HoloLens和Unity编辑器上运行。这款应用的桌面版将会有更多的组件内置合成的全息影像。

## 四、如何设置第三人称视角Pro

### 4.1 必需品

![Hardware shopping list](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdGF0b3J2aWV3cmlnLTM1MHB4LmpwZw?x-oss-process=image/format)

以下是推荐的硬件清单，你也可以实验其他兼容的硬件。

| 硬件组建 | 建议 |
|:--- |:--- |
| 一台PC，用于全息开发和全息模拟器 |  |
| 具有HDMI输出的相机或照片捕捉SDK | 对于图片和视频捕捉，我们已经测试了[Canon EOS 5D Mark III](https://www.amazon.com/Canon-Frame-Full-HD-Digital-Camera/dp/B007FGYZFI/ref=sr_1_3?s=photo&ie=UTF8&qid=1480537693&sr=1-3&keywords=Canon+5D+Mark+III)相机。<br/>对于现场演示，我们已经测试了[Blackmagic Design Production Camera 4K](https://www.amazon.com/Blackmagic-Design-Production-Camera-Mount/dp/B00CWLSHYG/ref=sr_1_1?s=photo&ie=UTF8&qid=1480537790&sr=1-1&keywords=blackmagic+design+production+camera+4k)。注意，任何带HDMI的相机（例如GoPro）都应该可以工作。<br/>我们的许多视频都使用了[Canon EF 14mm f/2.8L II USM Ultra-Wide Angle Fixed Lens](https://www.amazon.com/Canon-Ultra-Wide-Angle-Digital-Cameras/dp/B000V5P94Q)，但是你应该选择一个适合你的相机镜头。 |
| 采集卡，为你的PC从相机得到色彩帧校准并预览你的合成场景。 | 我们已经测试了 [Blackmagic Design Intensity Pro 4K capture card](https://www.amazon.com/dp/B00U3QNP7Q)。 |
| 线缆 | 使用[HDMI to Mini HDMI](https://www.amazon.com/AmazonBasics-High-Speed-Mini-HDMI-HDMI-Cable/dp/B014I8UHXE?ie=UTF8&psc=1&redirect=true&ref_=oh_aui_detailpage_o03_s00) 连接相机与采集卡，确保你购买的HDMI线能够匹配你的相机。(例如GoPro通过 [Micro HDMI](https://www.amazon.com/dp/B014I8U33I/ref=twister_B0198TA40O?_encoding=UTF8&psc=1)输出) <br/>[HDMI cable](https://www.amazon.com/dp/B014I8TC4E/ref=twister_B016I3XG0S?_encoding=UTF8&th=1) 用于在显示器或电视上查看视频。 |
| 把你的全息眼镜和相机连接起来，更多细节可以在OSS项目README中找到。 | |
| 3D打印适配器将全息支架连接到摄像机的热靴上，更多细节可以在OSS项目README中找到。 | |
| 使用热靴将相机和HoloLens Mount连接起来 | [Hotshoe Fastener](https://www.amazon.com/Fotasy-SCX2-Adapter-Premier-Cleaning/dp/B00HPAPFNU/ref=redir_mobile_desktop?ie=UTF8&psc=1&ref_=yo_ii_img) |
| 各种螺母、螺栓和工具 | [1/4-20" Nuts](https://www.amazon.com/Hillman-Group-150003-20-Inch-100-Pack/dp/B000BPEPNW/ref=redir_mobile_desktop?ie=UTF8&psc=1&ref_=yo_ii_img)，[1/4-20" x 3/4" Bolts](https://www.amazon.com/Hard-Find-Fastener-014973100032-4-20-Inch/dp/B004S6RZPK/ref=redir_mobile_desktop?ie=UTF8&psc=1&ref_=yo_ii_img)，[7/16 Nut Driver](https://www.amazon.com/Klein-Tools-630-7-Cushion-Grip-Hollow-Shank/dp/B000BPG4CW/ref=sr_1_1?ie=UTF8&qid=1479853212&sr=8-1)，[T15 Torx](https://www.amazon.com/Stanley-60-011-Standard-Torx-Screwdriver/dp/B000KFXDWW/ref=sr_1_1?ie=UTF8&qid=1479853303&sr=8-1)，[T7 Torx](https://www.amazon.com/SE-7542ST-6-Piece-Professional-Screwdriver/dp/B000ST3K3W/ref=sr_1_1?ie=UTF8&qid=1479853479&sr=8-1) |

下面是需要的软件组件。

1. 从 [GitHub project for spectator view](https://github.com/Microsoft/HoloLensCompanionKit/tree/master/SpectatorView)下载软件。

2. [Blackmagic Capture Card SDK](https://www.blackmagicdesign.com/support)，在“最新下载”中搜索桌面视频开发者SDK。

3. [Blackmagic Desktop Video Runtime](https://www.blackmagicdesign.com/support)，在“最新下载”中搜索桌面视频软件更新，确保版本号与SDK版本匹配。

4. [OpenCV 3.1](https://opencv.org/releases.html)，用于在没有Blackmagic捕获卡的情况下进行校准或视频捕获。

5. [Canon SDK](https://www.usa.canon.com/internet/portal/us/home/explore/solutions-services/digital-camera-sdk-information) (可选)，如果你用的是佳能相机，并且可以使用佳能SDK，你可以把相机固定在你的电脑上以获得更高分辨率的图像。

6. 使用Unity作为全息应用开发，支持的版本可以在OSS项目中找到。

7. Visual Studio 2015 及更高版本。

### 4.2 建立自己的第三人称视角相机

**注意与免责声明：** 当等该HoloLens硬件（包括但不限于HoloLens for “三方视角”）时，应该始终遵守基本的安全预防措施。您有责任遵循所有指示，并按照指示使用工具。你可以购买或许可你的HoloLens限时保修或没有保修。请阅读您适用的HoloLens许可协议或使用和销售条款，以了解您的保修选择，[点击这里](http://microsoft.com/microsoft-hololens/en-us/order-now)。

<center class="video"><iframe width="591" height="333" src="https://www.youtube.com/embed/aKX8UMejtWc" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

#### 4.2.1 设备组装

![Assembled SpectatorView Pro rig with HoloLens and DSLR camera](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9hc3NlbWJseS5naWY)

- 使用T7螺丝起子从HoloLens上取下头带，一旦螺丝松了，用另一边的回形针把它们戳出来。

- 用一个小的平头螺丝起子拆卸HoloLens防护罩内侧前部的螺钉帽。

- 使用T15螺丝起子从HoloLens支架上拆卸小的 torx 螺栓，以拆卸U形和钩形附件。

- 将HoloLens放在支架上，在护目镜内侧将露出的孔与支架前部的挤压件排成一线。HoloLens的手臂应该被固定在支架底部的钉子上。

- 重新连接U形和钩形附件，以将HoloLens固定到支架上。

- 将热靴紧固件安装在相机的热靴上。

- 将 mount 适配器连接到热靴紧固件。

- 旋转适配器，使窄边面向前，并平行于相机镜头。

- 使用7/16螺母驱动将适配器固定在1/4"螺母处。

- 把支架放在适配器上，这样 HoloLens 防护罩的前部就会尽可能靠近相机镜头的前部。

- 使用7/16螺母驱动器，用4 1/4"螺母和螺栓连接支架。

#### 4.2.2 PC设置

- 从软件组件部分安装软件。

- 将采集卡添加到主板上的一个打开的PCIe插槽中。

- 从你的相机插入HDMI电缆到采集卡上的外部HDMI插槽(HDMI- in)。

- 将HDMI线缆从采集卡上的中心HDMI插口(hdmiout)插到一个可选的预览显示器上。

#### 4.2.3 相机设置

- 将你的相机设置为视频模式，这样它就可以输出1920x1080的分辨率，而不是3:4的分辨率。

- 找到你的相机的HDMI设置并**启用镜像**或**双显示器**。

- 将输出分辨率设置为1080P。

- 关闭**屏幕显示上的实时视图**，这样屏幕上的覆盖就不会出现在合成画面中。

- 打开相机的**实时视图**。

- 如果使用**佳能SDK**并想使用flash单元，关闭静默LV拍摄。

- 将HDMI线从相机插入到采集卡上的外部HDMI插槽(HDMI- in)。

#### 4.2.4 校准

在设置好观景设备后，你必须校准相机的位置和旋转偏移量到HoloLens上。

- 打开Calibration\Calibration.sln下的Calibration Visual Studio 解决方案。

- 在这个解决方案中，你会找到为第三方源的inc位置创建宏的dependencies.props文件。

- 使用您安装的OpenCV 3.1、Blackmagic SDK和Canon SDK的位置更新此文件(如果适用)

![Dependency locations snapshot in Visual Studio](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9kZXBlbmRlbmNpZXMtNjAwcHgucG5n?x-oss-process=image/format)

- 打印标准模式Calibration\CalibrationPatterns\2_66_grid_FULL.png文件到平坦坚硬的表面上。

- 通过USB把你的HoloLens连接你的电脑。

- 使用HoloLens的设备门户凭证更新**stdafx.h**中的预处理器定义**HOLOLENS_USER**和**HOLOLENS_PW**。

- 将你的相机通过HDMI连接到你的采集卡上，并打开它。

- 运行 Calibration  解决方案。

- 把棋盘上的图案像这样移动！

![Calibrating the SpectatorView Pro rig](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9jYWxpYnJhdGlvbi5naWY)

- 当看到棋盘时，就会自动拍照。在进入下一个姿势之前，在HoloLens的遮罩上寻找白光。

- 完成后，在Calibration 应用的焦点上按`Enter`键，创建 **CalibrationData.txt** 文件。

- 这个文件将被保存到`Documents\CalibrationFiles\CalibrationData.txt`中。

- 检查这个文件，看看你的校准数据是否准确：

  - **单反**`RMS`应该接近0。
  - **HoloLens`RMS`**应该接近0。

  - **立体**`RMS`可能是20~50，这是可以接受的因为这两个摄像头的视野可能不一样。

  - **平移**是从HoloLens相机到附加相机镜头的距离，在数米内。

  - **旋转**应该接近一致。

  - **DSLR_fov** y值应该接近从您的镜头焦距和任何相机主体裁剪因子预期的垂直视场。

- 如果以上任何一个值看起来没有意义，请重新校准。

- 拷贝这个文件到你的Unity项目的`Assets`目录下。

### 4.3 渲染

compositor 是一个Unity扩展，在Unity编辑器中作为窗口运行。为了实现它，首先需要构建Compositor Visual Studio解决方案。

- 打开`Compositor\Compositor.sln`下的 Compositor Visual Studio 解决方案。

- 更新依赖项，和上面的校准解决方案有相同的标准。如果你遵循校准步骤，这个文件就已经更新了。

- 将整个解决方案构建为Release版本，并且与你的Unity版本体系结构相匹配。如果有疑问，请同时构建x86和x64。

- 如果你为x64构建了解决方案，那么还应该将SpatialPerceptionHelper项目构建为x86，因为它将在HoloLens上运行。

- 如果正在运行你的应用程序，则关闭Unity。如果 DLLs 在运行时发生更改，则需要重新启动Unity。

- 运行 `Compositor\CopyDLL.cmd` 来复制构建的 DLLs 从这个解决方案到你的Unity项目中。该脚本将复制 DLLs 到所包括的样本项目。一旦你建立了自己的项目，你也可以用命令行参数运行CopyDLL，并指向项目的 Assets 目录进行复制。

- 运行这个示例Unity应用。

### 4.4 Unity应用

compositor作为Unity编辑器中的窗口运行。所包含的样例项目已经设置好了所有内容，一旦 compositor 的DLLs被复制，就会有三方视角。

三方视角需要应用程序以 [shared experience](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-240)方式来运行，这意味着在HoloLens上发生的任何应用状态变化都需要联网来更新在Unity中运行的应用。

如果从一个新的Unity项目开始，你需要先做一些设置:

- 从示例程序复制**Assets/Addons/HolographicCameraRig**到你的项目中。

- 将最新的`MixedRealityToolkit`添加到你的项目中，包括**Sharing**, **csc.rsp**, **gmcs.rsp** 和 **smcs.rsp**。

- 添加你的**CalibrationData.txt**文件到你的 Assets 目录。

![files-400px](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9maWxlcy00MDBweC5wbmc?x-oss-process=image/format)

- 添加 **HolographicCameraRig\Prefabs\SpectatorViewManager** 预制到你的场景并填写一下字段：

  - **HolographicCameraManager**应使用 HolographicCameraRig 预制目录中的预制程序填充 HolographicCameraManager 。

  - **Anchor**应使用 HolographicCameraRig 预制目录中的预制程序填充Anchor。

  - **Sharing**应使用 MixedRealityToolkit 预制目录中的预制程序填充Sharing。

  - 注意: 如果这些预制件已经存在于您的项目层次结构中，那么将使用现有的预制件而不是这些预制件。

  - 三方视角 IP 应该是连接到观众观看设备的HoloLens的IP。

  - **Sharing Service IP**应该是正在运行MixedRealityToolkit SharingService的PC的IP。

  - 可选:如果多个三方视角机器连接到多个PC，应将本地计算机IP设置为PC，三方视角机器将与之通信。

![spectatorviewmanager-500px](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9zcGVjdGF0b3J2aWV3bWFuYWdlci01MDBweC5wbmc?x-oss-process=image/format)

- 启动 MixedRealityToolkit 的共享服务（**Sharing Service**）

- 将应用程序作为D3D UWP构建并部署到连接到三方视图平台的HoloLens上。

- 将应用程序部署到体验中的任何其他HoloLens设备上。

- 选中`edit/project settings/player`下的 `Run In Background` 复选框。

![run_in_bg](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9ydW4taW4tYmctNDAwcHgucG5n?x-oss-process=image/format)

- 在`Spectator View/Compositor`下启动合成窗口

![compositor-500px](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kb2NzLm1pY3Jvc29mdC5jb20vemgtY24vd2luZG93cy9taXhlZC1yZWFsaXR5L2ltYWdlcy9jb21wb3NpdG9yLTUwMHB4LnBuZw?x-oss-process=image/format)

- 此窗口允许你：

  - 启动视频录制

  - 拍照

  - 改变全息影像不透明度

  - 更改帧偏移量(调整颜色时间戳以考虑采集卡延迟)

  - 打开保存采集到的目录

  - 从三方视角摄像机请求空间映射数据(如果您的项目中存在一个SpatialMappingManager)

  - 分别查看场景的合成视图以及色彩、全息图和字母通道

  - 打开你的相机

  - 在Unity中按下Play

  - 当相机被移动时，在Unity中的全息影响应该是他们在真实世界中相对于你的相机 color feed 的位置。
