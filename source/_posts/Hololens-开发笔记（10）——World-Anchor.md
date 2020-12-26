---
title: HoloLens 开发笔记（10）——World Anchor
categories: HoloLens
abbrlink: c957c529
date: 2018-12-16 13:06:18
related_repos:
  - name: HoloLens
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
    rel: nofollow noopener noreferrer
    target: _blank
references:
  - name: Spatial anchors
    url: https://docs.microsoft.com/zh-cn/windows/mixed-reality/spatial-anchors
    rel: nofollow noopener noreferrer
    target: _blank
  - name: HoloLens入门之空间锚与场景保持
    url: https://blog.csdn.net/sun_t89/article/details/52432807
    rel: nofollow noopener noreferrer
    target: _blank
  - name: HoloLens开发手记 - Unity之World Anchor空间锚
    url: https://www.cnblogs.com/mantgh/p/5578590.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: HoloLens anchor使用与共享
    url: https://blog.csdn.net/fdbvm/article/details/79324718
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

HoloLens 实现全息体验的一个特性就是**场景保持**。当用户离开场景或关闭应用时，场景中的全息图会被保存在所放置的位置，当用户回到场景或重新打开应用时，能够准确的还原之前场景内的全息内容。

`World Anchor（空间锚）`提供了一种能够将物体保留在特定位置和旋转状态上的方法，以此来保证全息对象的稳定性（即静止参考框架），也通过它来实现场景保持。

`WorldAnchorStore` 是实现空间锚特性的关键 API，为了能够真正保持一个全息对象，通常为根 GameObject 添加空间锚，同时对其子 GameObject 也附上具有相对位置偏移的空间锚组件。

## 一、相关 API

添加命名空间：

```csharp
using UnityEngine.XR.WSA;
using UnityEngine.XR.WSA.Persistence;
```

**（1）为物体添加空间锚**

```csharp
WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
```

**（2）销毁物体上的空间锚**

当物体被添加空间锚后，该物体不能够再移动。

单纯的销毁空间锚，不需要移动物体：

```csharp
Destroy(gameObject.GetComponent<WorldAnchor>());
```

需要移动物体，使用 DestroyImmediate 来销毁空间锚：

```csharp
DestroyImmediate(gameObject.GetComponent<WorldAnchor>());
```

**（3）移动已经添加空间锚的物体**

之前说过物体被添加空间锚后无法移动，因此步骤如下：

1. 销毁空间锚
2. 移动物体
3. 重新添加空间锚

```csharp
DestroyImmediate(gameObject.GetComponent<WorldAnchor>());
gameObject.transform.position = new Vector3(0, 0, 2);
WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
```

**（4）读取已保存的所有空间锚**

通过调用 WorldAnchorStore.GetAsync() 来加载所有保存的空间锚。

```csharp
void Start () {
    WorldAnchorStore.GetAsync(AnchorStoreReady);
}

private void AnchorStoreReady(WorldAnchorStore store)
{
    // 读取所有已保存的空间锚
    WorldAnchorStore anchorStore = store;
    string[] ids = anchorStore.GetAllIds();
}
```

**（5）保存空间锚**

```csharp
/**
 * 返回是否保存成功
 * @Param anchorName: 保存的锚点名
 * @Param anchor: 物体上的锚点组件
 */
bool saved = anchorStore.Save(anchorName, anchor);
```

**（6）加载已保存的空间锚到物体上**

```csharp
/**
 * 当加载成功时返回锚点对象
 * @Param anchorName: 保存的锚点名
 * @Param gameObject: 被添加空间锚的目标对象
 */
WorldAnchor anchor = anchorStore.Load(anchorName, gameObject);
```

**（7）删除已保存的空间锚**

```csharp
/**
 * 返回是否删除成功
 * @Param anchorName: 删除的锚点名
 */
bool deleted = anchorStore.Delete(anchorName);
```

**（8）OnTrackingChanged 事件**

当我们为物体添加空间锚的情况下，有些情况空间锚会被立即定位到，即：

```csharp
WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
// anchor.isLocated == true
```

但是有些情况下不会被立即定位到，我们可以为空间锚绑定 OnTrackingChanged 事件，当它定位成功后，再继续后面的逻辑。

```csharp
anchor.OnTrackingChanged += Anchor_OnTrackingChanged;
```

例如，我们需要为物体添加空间锚，等到被定位后将其保存起来，那么代码大概如下：

```csharp
void OnSelect() {
    WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();
    if(anchor.isLocated) {
        anchorStore.Save("测试锚点名", anchor);
    } else {
        anchor.OnTrackingChanged += Anchor_OnTrackingChanged;
    }
}

void Anchor_OnTrackingChanged(WorldAnchor self, bool located) {
    if(located) {
        anchorStore.Save("测试锚点名", self);
        // 取消事件监听
        self.OnTrackingChanged -= Anchor_OnTrackingChanged;
    }
}
```

## 二、示例程序

使用 MRTK 初始化一个 HoloLens 应用：

1. 删除默认相机，使用 HoloToolkit / Input / Prefabs / HoloLensCamera 替代
2. 添加 HoloToolkit / Input / Prefabs / Cursor / CursorWithFeedback
3. 添加 HoloToolkit / Input / Prefabs / InputManager，设置其 Simple Single Pointer Selector 的 Cursor 为上一步的 Cursor。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181204145735150.png)

4. 添加一个 Cube，它的 Position 为 (X:0, Y:0, Z:4), Scale 为 (X:0.25, Y:0.25, Z:0.25)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181204145754692.png)

5. 编写脚本 CubeCommand 并将其添加到 Cube 上。

```csharp
using UnityEngine;
using HoloToolkit.Unity.InputModule;
using UnityEngine.XR.WSA;
using UnityEngine.XR.WSA.Persistence;
using System.Linq;

public class CubeCommand : MonoBehaviour, IInputClickHandler {
    // 被保存的锚点名
    public string ObjectAnchorStoreName;

    WorldAnchorStore anchorStore;

    // 是否可被移动
    bool HasMove = false;
    
    void Start ()
    {
        WorldAnchorStore.GetAsync(AnchorStoreReady);
    }

    private void AnchorStoreReady(WorldAnchorStore store)
    {
        anchorStore = store;

        if (anchorStore.GetAllIds().Contains(ObjectAnchorStoreName))
        {
            anchorStore.Load(ObjectAnchorStoreName, gameObject);
        }
    }
    
    void Update ()
    {
        // 如果立方体可移动，更新其位置
        if (HasMove)
        {
            gameObject.transform.position = Camera.main.transform.position + Camera.main.transform.forward * 2;
        }
	}

    public void OnInputClicked(InputClickedEventData eventData)
    {
        if (anchorStore == null)
        {
            return;
        }

        if(HasMove)
        {
            // 当物体处于可移动，且再次被点击后
            WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();

            if (anchor.isLocated)
            {
                anchorStore.Save(ObjectAnchorStoreName, anchor);
            }
            else
            {
                anchor.OnTrackingChanged += Anchor_OnTrackingChanged;
            }
        }
        else
        {
            // 当物体处于不可移动，且再次被点击后
            WorldAnchor anchor = gameObject.GetComponent<WorldAnchor>();
            if(anchor != null)
            {
                DestroyImmediate(anchor);
            }

            if (anchorStore.GetAllIds().Contains(ObjectAnchorStoreName))
            {
                anchorStore.Delete(ObjectAnchorStoreName);
            }
        }

        HasMove = !HasMove;
    }

    void Anchor_OnTrackingChanged(WorldAnchor self, bool located)
    {
        if (located)
        {
            anchorStore.Save(ObjectAnchorStoreName, self);
            // 取消事件监听
            self.OnTrackingChanged -= Anchor_OnTrackingChanged;
        }
    }
}
```

6. 运行程序

初始位置位于靠近屋顶：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181204145857992.png)

通过点击事件，将其拖拽到地上：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181204145907467.png)

关闭程序，重新打开后，物体仍然停留在地上。

## 三、锚点共享

锚点可以在多个设备间共享，来使得不同设备可以使用相同的空间位置，可以通过 `WorldAnchorTransferBatch`将锚点信息导出为byte数组，在另外一台设备中加载这个数组并重新还原出锚点信息。

（1）锚点导出方

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.XR.WSA;
using UnityEngine.XR.WSA.Sharing;

public class ExportAnchorScript : MonoBehaviour {
    public string exportingAnchorName;
    private List<byte> exportingAnchorBytes = new List<byte>();

    void Start ()
    {
        WorldAnchorTransferBatch transferBatch = new WorldAnchorTransferBatch();
        transferBatch.AddWorldAnchor(exportingAnchorName, transform.GetComponent<WorldAnchor>());
        WorldAnchorTransferBatch.ExportAsync(transferBatch, OnExportDataAvailable, OnExportComplete);
	}

    private void OnExportDataAvailable(byte[] data)
    {
        exportingAnchorBytes.AddRange(data);
    }

    private void OnExportComplete(SerializationCompletionReason completionReason)
    {
        if (completionReason == SerializationCompletionReason.Succeeded)
        {
            Debug.Log("share anchor complete");
        }
        else
        {

        }
    }
}
```

（2）锚点导入方

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.XR.WSA;
using UnityEngine.XR.WSA.Sharing;

public class ImportAnchorScript : MonoBehaviour {
    public string exportingAnchorName;
    // 导入的目标对象
    public GameObject targetObject;
    int retryCount = 5;

    private List<byte> exportingAnchorBytes = new List<byte>();
    
    void Start ()
    {
        WorldAnchorTransferBatch.ImportAsync(exportingAnchorBytes.ToArray(), OnImportComplete);
    }

    private void OnImportComplete(SerializationCompletionReason completionReason, WorldAnchorTransferBatch deserializedTransferBatch)
    {
        if (completionReason != SerializationCompletionReason.Succeeded)
        {
            Debug.Log("Failed to import: " + completionReason.ToString());
            if (retryCount > 0)
            {
                retryCount--;
                WorldAnchorTransferBatch.ImportAsync(exportingAnchorBytes.ToArray(), OnImportComplete);
            }
            return;
        }

        string[] ids = deserializedTransferBatch.GetAllIds();
        Debug.Log("load anchor count " + ids.Length);
        foreach (string id in ids)
        {
            Debug.Log("load anchor " + id);
            if (targetObject != null && id.Equals(exportingAnchorName))
            {
                Debug.Log("find anchor form share");
                if (targetObject.GetComponent<WorldAnchor>() == null)
                {
                    targetObject.AddComponent<WorldAnchor>();
                }

                deserializedTransferBatch.LockObject(id, targetObject);
                return;
            }
        }
    }
}
```
