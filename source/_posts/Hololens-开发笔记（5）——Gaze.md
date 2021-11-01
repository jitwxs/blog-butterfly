---
title: HoloLens 开发笔记（5）——Gaze
categories: HoloLens
abbrlink: aa63820b
date: 2018-11-28 11:53:40
related_repos:
  - name: HoloLens Demo
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
---

在前面我们粗略了解了 HoloLens，包括 `Hello World`、`MRTK`、`Windows Device Portal`、`坐标系统`。

下面开始 HoloLens 基础部分的学习，包括 `Gaze`、`Gesure`、`Voice`、`Audio Souce` 等。本篇文章来学习 HoloLens 的基础开发之凝视操作。

创建一个新的 Unity 项目 GazeDemo，导入 MRTK 工具包，并将项目应用为 MR 项目。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201811/20181121205314130.png)

删除掉默认的相机，添加 MRTK 的 `HoloLensCamera` 到 Hierarchy 中。添加 MRTK 中的 `CursorWithFeedback` 到 Hierarchy  中。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218201235184.png)

## 一、凝视操作

从 MRTK 中拖拽 `InputManager` 到 Hierarchy 中。这是一个十分重要的 Manager，它将管理我们的凝视等输入事件。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218201814467.png)

设置 InputManager 的 `SimpleSinglePointerSelector` 脚本的 Cursor 属性为添加的 CursorWithFeedback。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018121821571321.png)

随后添加一个 Cube 到 Hierarchy 中，来实现凝视 Cube 5 秒将其隐藏掉的效果。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218201741787.png)

新建脚本 `CubeCommand`，并将其添加到 Cube 上。

```csharp
using UnityEngine;
using HoloToolkit.Unity.InputModule;
using System;

public class CubeCommand : MonoBehaviour, IFocusable {
    // 延迟多少秒后触发
    public double duration = 5.0;
    private DateTime? startDate;

    void Update () {
        if (startDate.HasValue)
        {
            TimeSpan span = DateTime.Now - startDate.Value;

            if (span.TotalSeconds > duration)
            {
                HideCube();
            }
        }
    }

    void HideCube()
    {
        startDate = null;
        gameObject.GetComponent<MeshRenderer>().enabled = false;
    }

    public void OnFocusEnter()
    {
        startDate = DateTime.Now;
    }

    public void OnFocusExit()
    {
        startDate = null;
        if (!gameObject.GetComponent<MeshRenderer>().enabled)
        {
            gameObject.GetComponent<MeshRenderer>().enabled = true;
        }
    }
}
```

运行程序，当我们将 Cursor 停留 Cube 5秒后，Cube 消失，当我们移开 Cursor 时，Cube 恢复。

总结下代码：

1. 导包：`using HoloToolkit.Unity.InputModule;`
2. 实现接口 `IFocusable`，当 Cursor 进入物体时，调用 `OnFocusEnter()` 方法，当 Cursor 移出物体时，调用 `OnFocusExit()` 方法。
3. 在 `Update()` 方法中计算凝视的时长，当达到 duration 时，将物体隐藏。

## 二、Directional indicator

假如我们为 Cube 添加了方向指示器（Directional indicator），当我们的视野中看不见该 Cube 时，方向指示器会指示出它的位置。实现步骤如下：

1. 为 Cube 添加 MRTK 包中的 `DirectionIndicator.cs` 脚本。
2. 选中该脚本，设置 `Cursor` 属性为 Hierarchy 中的 CursorWithFeedback；设置`DirectionIndicatorObject` 属性为 MRTK 包中的 `HeadsUpDirectionIndicatorPointer`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218203714810.png)

具体属性含义如下：

| 属性名                   | 含义                                                         |
|:------------------------ |:------------------------------------------------------------ |
| Cursor                   | 该物体在场景中被当作光标，方向指示器会显示在这个物体旁边     |
| DirectionIndicatorObject | 方向指示器物体，该物体会一直指向附加该脚本的对象。           |
| DirectionIndicatorColor  | 方向指示器的颜色（方向指示器材质里的Shader必须要有“_TintColor”属性，否则颜色不会变） |
| VisibilitySafeFactor     | 范围[-0.3,0.3] ，当物体在摄像机视锥的某个百分比范围中，方向指示器才会显示。（例如此值为0时，当物体完全离开摄像机视锥之后方向指示器才会显示；此值为0.1时，物体在视锥范围的90%之外，方向指示器才会显示；此值为-0.1时，物体在视锥范围的110%之外 ，方向指示器才会显示） |
| MetersFromCursor         | 方向指示器从原中心到它面向方向(forward)的一个偏移值。        |

运行程序，你会发现一个巨大的三角箭头指示着 Cube 的方向：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218204119264.png)

MRTK 默认提供的 DirectionIndicatorObject 实在是太丑了。我这里使用 [MR Input 210: Gaze Chapter 4 - Directional indicator](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-210#chapter-4---directional-indicator) 中提供的方向指示器，效果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218204447980.png)

## 三、Billboarding

如果为 Cube 添加了广告牌（Billboarding）效果，他就会永远的面朝你，即使你尝试走到它的后面（应用在 Cube 上没啥意义，可以使用 Text 来测试）。

为一个 gameObject 添加广告牌效果十分简单，只要为其添加 MRTK 包下的 `Billboard.cs` 脚本，并设置它的 `PivotAxis` 属性为 `Y` 即可，即绕着 Y 轴实现广告牌。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218204944734.png)

## 四、Tag-Along

不知道你还记不记得 HoloLens 的 Bloom 手势呼出主菜单的效果，主菜单是跟随你移动并且使终面朝你。这就是**广告牌 + 平滑追踪**的联合实现。

为要平滑追踪（Tag-Along）的物体添加 MRTK 包下的 `Tagalong.cs` 脚本即可。

这里我添加了一个 3D Text，并为其添加了广告牌和平滑追踪的效果，运行效果如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018121820595640.png)

## 五、Cursor 底层实现

> 代码来源于：[MR Basics 101: Complete project with device: Chapter 2 - Gaze](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-101#chapter-2---gaze)

我们可以看下 Cursor 的效果是如何实现的，创建一个 Cursor ，它其实就是一个圆圈，为它附加脚本 `WorldCursor` ：

```csharp
using UnityEngine;

public class WorldCursor : MonoBehaviour
{
    private MeshRenderer meshRenderer;

    // Use this for initialization
    void Start()
    {
        // Grab the mesh renderer that's on the same object as this script.
        meshRenderer = this.gameObject.GetComponentInChildren<MeshRenderer>();
    }

    // Update is called once per frame
    void Update()
    {
        // Do a raycast into the world based on the user's
        // head position and orientation.
        var headPosition = Camera.main.transform.position;
        var gazeDirection = Camera.main.transform.forward;

        RaycastHit hitInfo;

        if (Physics.Raycast(headPosition, gazeDirection, out hitInfo))
        {
            // If the raycast hit a hologram...
            // Display the cursor mesh.
            meshRenderer.enabled = true;

            // Move the cursor to the point where the raycast hit.
            this.transform.position = hitInfo.point;

            // Rotate the cursor to hug the surface of the hologram.
            this.transform.rotation = Quaternion.FromToRotation(Vector3.up, hitInfo.normal);
        }
        else
        {
            // If the raycast did not hit a hologram, hide the cursor mesh.
            meshRenderer.enabled = false;
        }
    }
}
```

Unity 是没有明确的 Gaze 的 API，在 HoloLens 中，是通过射线检测来实现的。当视线接触到了碰撞物时，展示出一个⭕；当视线没有接触到碰撞物时，隐藏⭕。

（1）获取其及其子对象中的 MeshRenderer

```csharp
meshRenderer = this.gameObject.GetComponentInChildren<MeshRenderer>();
```

- GetComponent<T>()：获取对象中指定类型的控件（脚本）
- GetComponentInChildren<T>()：获取对象的子对象中指定类型的控件，如果父对象中有，优先获取父对象的
- GetComponents<T>()、GetComponentsInChildren<T>()：返回数组

Mesh Filter（网格过滤器）用于从资源中读取网格信息（Mesh） ，读取信息之后可以传递给 Mesh Renderer（网格渲染器 ），将接受到的网格信息渲染出来。

（2）获取主相机的位置和朝向

```csharp
var headPosition = Camera.main.transform.position;
var gazeDirection = Camera.main.transform.forward;
```

（3）调用射线检测函数，从 headPosition 射向 gazeDirection ，当碰撞到物体时，返回true，并将碰撞信息放入 hitInfo 。

```csharp
Physics.Raycast(headPosition, gazeDirection, out hitInfo)
```

（4）如果碰撞到物体，显示 meshRenderer，并将位置移动到碰撞的位置。将⭕旋转，从（0,1,0)到碰撞点的法线。

```csharp
meshRenderer.enabled = true;
this.transform.position = hitInfo.point;
this.transform.rotation = Quaternion.FromToRotation(Vector3.up, hitInfo.normal);
```

（5）如果没有碰撞到物体，隐藏 meshRenderer。
