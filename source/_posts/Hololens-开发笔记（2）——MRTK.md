---
title: HoloLens 开发笔记（2）——MRTK
typora-root-url: ..
categories: HoloLens
abbrlink: 51ef9785
date: 2018-11-21 21:01:06
related_repos:
  - name: MixedRealityToolkit-Unity
    url: https://github.com/Microsoft/MixedRealityToolkit-Unity
    rel: nofollow noopener noreferrer
    target: _blank
  - name: HoloLens Demo
    url: httphttps://github.com/jitwxs/blog-sample/tree/master/HoloLens
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、什么是 MRTK？

`MRTK(Mixed Reality Toolkit)` 是微软为我们提供的混合现实开发工具包，旨在帮助我们加速开发混合现实应用程序。

![](/images/posts/20181121205140695.png)

基于 Unity 的 MRTK(MixedRealityToolkit-Unity) 提供了许多 API 来加速混合现实项目的开发，包括 HoloLens 和 IHMD。

![](/images/posts/20181121205201455.png)

## 二、如何使用MRTK？

在我写这篇文章时（2018-11-21）时，最新的 Relaese 版本为 Mixed Reality Toolkit 2018.9.0 (vNext Beta) ，仍然为 Beta 版，在配合 Unity 2018 2.x 使用的时候仍然会出现 BUG，因此我使用了[HoloToolkit 2017.4.2.0](https://github.com/Microsoft/MixedRealityToolkit-Unity/releases/tag/2017.4.2.0) 版本。

![](/images/posts/20181121205219161.jpg)

请下载 `HoloToolkit-Unity` 和 `HoloToolkit-Unity-Examples` 这两个 unitypackage 文件。

使用的先提条件和具体流程参考官方文档 [GettingStarted](https://github.com/Microsoft/MixedRealityToolkit-Unity/blob/master/GettingStarted.md)，下面我简单叙述下：

1. 创建一个新的 3D Unity项目，然后点击 `Assets -> Import Package -> Custom Package…`，导入 `HoloToolkit-Unity` 

>1. 导入过程中如果你的 Unity 大于 Unity  2017.4 LTS，会提示有些地方过期，选择自动升级GoAhead即可。
>
>2. HoloToolkit-Unity-Examples 为官方提供的例子，在自己跟着文档学习的时候再导入。

![](/images/posts/20181121205234603.jpg)

2. 导入完成后，检查下菜单栏是否多出一个 `Mixed Reality Toolkit`，再检查下 Project 面板的 Assets 目录下是否有 `HoloToolkit` 目录。如果都存在，至此安装完成。

![](/images/posts/20181121205247143.jpg)

## 三、Hello World

下面开始 HelloWorld，这里演示一个 [《HoloLens 开发笔记（1）——HelloWorld》](/e42fdabb.html) 中实现的立方体，并为其添加 Cursor 效果。

1. 首先将项目设置为 MR 项目，点击 `Mixed Reality Toolkit -> Configure -> Apply Mixed Reality Scene Settings`，即可一键切换，不再需要手动设置了。

![](/images/posts/20181121205314130.jpg)

![](/images/posts/20181121205329569.jpg)

2. 删除默认的 **Main Camera**，从 `Assets -> HoloToolkit -> Input -> Prefabs` 中添加`MixedRealityCamera` 预置体，并设置该相机：

![](/images/posts/20181121205340786.jpg)

![](/images/posts/20181121205352926.jpg)

3. 添加一个立方体：

![](/images/posts/20181121205402160.jpg)

4. 从 `Assets -> HoloToolkit -> Input -> Prefabs` 中添加 `InputManager` 预置体，从 `Assets -> HoloToolkit -> Input -> Prefabs-> Cursor` 中添加 `CursorWithFeedback`预置体。Hierarchy 目录结构如下：

   > 我使用了 CursorWithFeedback ，具有反馈功能，你也可以尝试使用 Cursor 目录下的其他 Cursor

![](/images/posts/20181121205413965.jpg)

5. 点击 `InputManager` 预置体，在右边的设置中，拖拽 `CursorWithFeedback`预置体到  `SimpleSinglePointerSelector` 脚本的 Cursor 属性：

![](/images/posts/20181121205424423.jpg)

6. 运行程序，当：
   - 未碰触到 Cude：出现一个光点
   - 碰触到 Cude：出现 Cursor
   - 碰触到 Cude，且检测到手：出现一个手指
   - 碰触到 Cude，且手处于 Tap 状态：出现一个 Tap 的手指

## 四、Camera

MRTK 包下为我们提供了许多种 Camera ，它们的说明如下。

### 4.1 HoloLensCamera.prefab

Unity camera that has been customized for Holographic development.

1. Camera.Transform set to 0,0,0
2. 'Clear Flags' changed to 'Solid Color'
3. Color set to R:0, G:0, B:0, A:0 as black renders transparent in HoloLens.
4. Set the recommended near clipping plane.
5. Allows manual movement of the camera when in editor

### 4.2 MixedRealityCamera.prefab

Camera capabale of rendering for HoloLens and occluded Windows Mixed Reality enabled devices. 

MixedRealityCameraManager.cs exposes some defaults for occluded aka opaque displays Vs HoloLens. You can either use the defaults that have been set or customize them to match your application requirements.

**For HoloLens:**

1. 'Clear Flags' is set to 'Solid Color' and Color to 'Clear' as black renders transparent in HoloLens.
2. Near clip plane is set to 0.85 per comfort recommendations.
3. Quality Settings to be Fastest.

**For occluded aka opaque devices:**

1. 'Clear Flags' is set to Skybox
2. Near clip plane is set to 0.3 which is typical for VR applications.
3. Quality Settings to be Fantastic as it uses the PC GPU to render content.

### 4.3 MixedRealityCameraParent.prefab

This prefab is used when you want to enable teleporting on mixed reality enabled occluded devices. In order to prevent the MainCamera position from being overwritten in the next update we use a parent GameObject.

## 五、Cursor

Cursor 代表的就是我们的实现，它的位置就是我们所看的物体。MRTK 包中提供了以下几种 Cursor 的预制体，这几种的说明如下：

| Name                      | Description                                                  |
|:------------------------- |:------------------------------------------------------------ |
| BasicCursor.prefab        | Torus shaped basic cursor that follows the user's gaze around. |
| Cursor.prefab             | Torus shaped CursorOnHolograms when user is gazing at holograms and point light CursorOffHolograms when user is gazing away from holograms. |
| CursorWithFeedback.prefab | Torus shaped cursor that follows the user's gaze and HandDetectedFeedback asset to give feedback to user when their hand is detected in the ready state. |
| DefaultCursor.prefab      | 3D animated cursor that follows the user's gaze and uses the Unity animation system to handle its various states. This cursor imitates the HoloLens Shell cursor. |

![](/images/posts/20181218200550791.png)
