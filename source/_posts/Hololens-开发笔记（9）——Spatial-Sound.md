---
title: HoloLens 开发笔记（9）——Spatial Sound
categories: HoloLens
abbrlink: 8990e5f7
date: 2018-12-12 13:02:17
related_repos:
  - name: HoloLens Demo
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
    rel: nofollow noopener noreferrer
    target: _blank
---

在前面我们学习了HoloLens的基础部分，包括 `Gaze`、`Gesure`、`Voice`、`Audio Souce` 等学习，下面开始进阶部分。

进阶部分包含 `Spatial Sound`、`World Anchor`、`Spatial Mapping`、`Sharing`、`Spectator View` 等内容，欢迎大家一起交流学习。在开始本文学习前，请确保已经学习了基础部分的内容。

创建一个新的 Unity 项目 SpatialSoundDemo，初始化项目：

1. 导入 MRTK 包

2. 应用项目设置为 MR 项目

3. 使用 `HoloLensCamera` 替代默认相机

4. 添加 `CursorWithFeedback`

5. 添加 `InputManager`

6. 设置 InputManager 的 `SimpleSinglePointerSelector` 脚本的 Cursor 属性为添加的 CursorWithFeedback

7. 添加一个 Cube，位置如下

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218201741787.png)

最终 Hierarchy 结构如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218221721133.png)

## 一、Spatial Sound

开启 Unity 的空间声音设置，在设置菜单中 `Edit/Audio/Spatializer` 启用 `Microsoft HRTF` 拓展。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181119143038972.jpg)

为 Cube  添加一个 Audio Souce 组件，配置如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018121915375780.png)

每一项具体的含义不再赘述了，可以参考：[《HoloLens 开发笔记（8）——Audio Sound》](/602f136c.html)。使用 Unity 或在真机中运行程序，通过改变和 Cube 的远近，感受声音的变化。

## 二、Sound Occlusion

下面来演示下当 Cube 被其他物体遮挡后，声音能够发生变化。

1. 给 Cube 添加 MRTK 包中的 `Audio Emitter.cs` 脚本 ，使用默认参数即可。
2. 新建一个 Sphere，为其添加 MRTK 包中的 `Audio Occluder.cs` 脚本，使用默认参数即可。

使用 Unity 运行程序，在 Scene 中通过改变 Sphere 是否遮挡住 Cube，来感受声音的变化。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219154433368.png)
