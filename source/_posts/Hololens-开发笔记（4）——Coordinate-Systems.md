---
title: HoloLens 开发笔记（4）——Coordinate Systems
categories: HoloLens
abbrlink: 75fdb477
date: 2018-11-26 11:29:15
references:
  - name: Coordinate systems
    url: https://docs.microsoft.com/zh-cn/windows/mixed-reality/coordinate-systems
  - name: Spatial anchors
    url: https://docs.microsoft.com/zh-cn/windows/mixed-reality/spatial-anchors
  - name: Microsoft HoloLens 开发上手(4)
    url: 4
---

混合现实应用的核心就是如何在现实世界中放置看起来真实的全息影像，这涉及到全息图的精确定位，无论是在现实世界还是在虚拟世界中，HoloLens 为我们提供了空间坐标系统（spatial coordinate systems）来方便几何图形的定位。

## 一、空间坐标系统

所有的三维应用程序都是使用`笛卡尔坐标系`来描述物体的位置和方向，沿着坐标系建立 X，Y，Z轴。空间坐标系以**米**为单位表示其坐标值，因此可以十分方便的渲染对象和环境。

HoloLens 采用右手笛卡尔坐标系，也就是说 X 轴正方向指向右边，Y轴正方向与重力平行且指向上方，Z轴正方向朝向你。

笛卡尔坐标系的左手和右手的区别就是 Z 轴的方向是朝向你还是远离你。将左手和右手平放均指向右方，将手指弯曲指向上方，此时大拇指的朝向就是 Z 轴的朝向。

![笛卡尔坐标系](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181126112756736.png)

## 二、坐标参考框架

在全息渲染中，有些影像需要跟随用户头部的移动而移动，有些影像在用户头部移动时需要始终保持在固定的位置上。

HoloLens 为我们提供了两种参考框架，分别是`静止参考框架`（Stationary frame of reference） 和`附加参考框架`（Attached frame of reference）。

### 2.1 附加参考框架

附加参照框架中，当用户移动或转动头部的时候，内容也会跟着走。当 HoloLens 不知道自己在哪里的时候，就只会渲染基于附加参照框架的全息图。比如程序提示用户在世界中无法找到用户时，提供一个回退界面（Fallback UI），帮助用户退出或重新启动。

### 2.2 静止参考框架

在编写游戏、VR程序时，传统做法是建立一个绝对世界坐标系（absolute world coordinate system）。在该坐标系中，可以可靠的获取任意两个物体之间的关系，只要不移动物体的位置，它们的相对位置是保持不变的。

然而在 HoloLens 中，动态传感器会随着用户的移动而不断的调整对周围世界的扫描。如果仍然采用一个绝对世界坐标系，随着用户的移动，可能就会导致物体的漂移（drift）。

例如假设 HoloLens 采用绝对世界坐标系，定义房间左侧角落C1，右侧角落为C2，$C1(0,0,0), C2(10,0,0)$，在C1、C2上分别放置一个全息图，当用户在房间移动时，动态传感器重新扫描，发现 C1 到 C2 的距离只有9.9米，这时 $C2(9.9,0,0)$ ，C2的变化就会导致 C2上的全息图的位置变化，导致物体漂移。 

HoloLens 采用空间锚（spatial anchors）来解决这个问题。HoloLens 在用户放置全息图的位置上放置空间锚，每一个锚都有一个坐标系统，当用户移动导致动态传感器重新扫描时，HoloLens 根据需要调整每个锚的位置，来确保锚上的全息图停留在现实世界的固定位置。

HoloLens 支持将空间锚持久化保存（Spatial anchor persistence），这样在相同的环境下再次启动程序时可以加载锚，实现全息图的位置记忆功能。

HoloLens 还支持空间锚分享（Spatial anchor sharing），通过将空间锚和周围环境的传感器数据从一个HoloLens 传输到另一个HoloLens。两台设备使用共享的空间锚，使得用户可以在相同位置看到一样的东西。
