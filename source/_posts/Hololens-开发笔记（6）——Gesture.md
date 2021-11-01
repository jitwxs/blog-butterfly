---
title: HoloLens 开发笔记（6）——Gesture
categories: HoloLens
abbrlink: 714f9373
date: 2018-11-30 12:30:10
related_repos:
  - name: HoloLens Demo
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
---

本篇文章来学习 HoloLens 的基础开发之手势操作。

创建一个新的 Unity 项目 GestureDemo，初始化项目：

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

## 一、Navigation

新建一个脚本 `CubeNavigation.cs`，并将其添加到 Cube 上。

```csharp
using UnityEngine;
using HoloToolkit.Unity.InputModule;

public class CubeNavigation : MonoBehaviour,INavigationHandler {
    [Tooltip("旋转速度")]
    public float RotationSensitivity = 10.0f;

    public void OnNavigationCanceled(NavigationEventData eventData)
    {
        InputManager.Instance.PopModalInputHandler();
    }

    public void OnNavigationCompleted(NavigationEventData eventData)
    {
        InputManager.Instance.PopModalInputHandler();
    }

    public void OnNavigationStarted(NavigationEventData eventData)
    {
        InputManager.Instance.PushModalInputHandler(gameObject);
    }

    public void OnNavigationUpdated(NavigationEventData eventData)
    {
        // 计算旋转值，其中：eventData的CumulativeDelta返回手势导航差值，值域[-1, 1]
        float rotationFactor = eventData.CumulativeDelta.x * RotationSensitivity;
        transform.Rotate(new Vector3(0, -1 * rotationFactor, 0));
    }
}
```

当我们将 Cursor 移到 Cube 上，按下手势，并缓慢向左或向右移动时（即导航手势），Cube 会跟随绕 Y 轴旋转。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018121822170786.png)

可以看到实现导航手势主要是实现 `INavigationHandler` 接口，在 `OnNavigationUpdated()` 方法中改变 Cube 的 Rotate。

## 二、Hand Guidance

下面实现一个**当手势快要超出检测范围时，给出提示**的效果。

在 Hierarchy 创建一个空的 gameObject 并重命名为 `HandGuidanceManager`，为其添加 MRTK 的 `HandGuidance.cs` 脚本。

设置该脚本的 `Cursor` 属性为 Hierarchy  中的 CursorWithFeedback，设置其 `HandGuidanceIndicator` 属性为 MRTK 中的 `HeadsUpDirectionIndicatorPointer`。

`HandGuidanceThreshold` 属性含义是：当开始显示手动导航指示器时。1不在视图中，0在视图中居中。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218222822445.png)

> 和上一章[《HoloLens 开发笔记（5）——Gaze》](/aa63820b.html) 中一样，因为官方提供的 HeadsUpDirectionIndicatorPointer 实际效果实在是太丑了，因此我实际使用的是 [MR Input 210: Gaze Chapter 4 - Directional indicator](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-210#chapter-4---directional-indicator) 中提供的方向指示器。

这里使用 Unity 的运行按钮无法查看效果，将程序在真机中运行，结果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181222122748992.png)

## 三、Manipulation

下面来实现 Cube 的拖拽（Manipulation）移动。

新建一个脚本 `CubeManipulation.cs`，并将其添加到 Cube 上。

```csharp
using HoloToolkit.Unity.InputModule;
using UnityEngine;

public class CubeManipulation : MonoBehaviour, IManipulationHandler
{
    // Cube移动前的位置
    private Vector3 OriginPosition;

    public void OnManipulationCanceled(ManipulationEventData eventData)
    {
        InputManager.Instance.PopModalInputHandler();
    }

    public void OnManipulationCompleted(ManipulationEventData eventData)
    {
        InputManager.Instance.PopModalInputHandler();
    }

    public void OnManipulationStarted(ManipulationEventData eventData)
    {
        InputManager.Instance.PushModalInputHandler(gameObject);
        // 开始移动前，保存Cube原始位置
        OriginPosition = transform.position;
    }

    public void OnManipulationUpdated(ManipulationEventData eventData)
    {
        transform.position = OriginPosition + eventData.CumulativeDelta;
    }
}
```

运行程序，选中并点击 Cube ，尝试拖拽它。为了更好的体现拖拽的效果，可以先不加载 `CubeNavigation.cs` 脚本，不然你会发现 Cube 一边旋转一边被拖拽。
