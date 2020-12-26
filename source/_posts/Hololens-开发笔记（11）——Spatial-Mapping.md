---
title: HoloLens 开发笔记（11）——Spatial Mapping
categories: HoloLens
abbrlink: fd886217
date: 2019-01-25 17:40:36
copyright_author: Jitwxs
---

HoloLens 作为一款混合现实设备，其与传统 VR/AR 设备最大的区别是，能够和现实世界进行交互。

以一个立方体为例，当我们没有使用 `Spatial Mapping` 时，我们只能在空间中移动它，而不能把它放置在现实世界的物体上，例如放置在一个椅子上。当我们使用了 Spatial Mapping 后，HoloLens 会先扫描出所在房间的三维信息，扫描完毕后你就可以将物体放置在扫描后的空间物体上。

创建一个新的 Unity 项目 SpatialDemo，初始化项目：

1. 导入 MRTK 包
2. 应用项目设置为 MR 项目
3. 使用 `HoloLensCamera` 替代默认相机
4. 添加 `CursorWithFeedback`
5. 创建一个空 GameObject，名为 `Manager`，为其添加子 gameObject： `InputManager`
6. 设置 InputManager 的 `SimpleSinglePointerSelector` 脚本的 Cursor 属性为添加的 CursorWithFeedback
7. 添加一个 Cube，位置如下
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218201741787.png)

最终 Hierarchy 结构如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219161856957.png)

## 一、Spatial Mapping

（1）添加 MRTK 工具包下的 `SpatialMapping` 预制体到 Manager 对象下。

修改 Spatial Mapping Manager 的 `Surface Material` 属性值为 MRTK 包中的 `SpatialUnderstandingSurface`，其他参数使用默认值即可，该属性为空间扫描时所使用的材质。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219163624113.png)

（2）在 Manager 下新建一个 GameObject，名为 `SpatialProcessing`。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219163748595.png)

（3）为 SpatialProcessing 添加以下两个 MRTK 包中的脚本：

- `SurfaceMeshesToPlanes.cs`
- `RemoveSurfaceVertices.cs`

（4）新建脚本 `SpatialProcessing.cs`，并将其添加到 SpatialProcessing 上。

```csharp
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.

using System.Collections.Generic;
using UnityEngine;

namespace HoloToolkit.Unity.SpatialMapping.Tests
{
    public class SpatialProcessing : Singleton<SpatialProcessing>
    {
        [Tooltip("How much time (in seconds) that the SurfaceObserver will run after being started; used when 'Limit Scanning By Time' is checked.")]
        public float scanTime = 30.0f;

        [Tooltip("Material to use when rendering Spatial Mapping meshes while the observer is running.")]
        public Material defaultMaterial;

        [Tooltip("Optional Material to use when rendering Spatial Mapping meshes after the observer has been stopped.")]
        public Material secondaryMaterial;

        [Tooltip("结束处理所需要的最小floor数量")]
        public uint minimumFloors = 1;

        /// <summary>
        /// Indicates if processing of the surface meshes is complete.
        /// </summary>
        private bool meshesProcessed = false;

        /// <summary>
        /// GameObject initialization.
        /// </summary>
        private void Start()
        {
            // Update surfaceObserver and storedMeshes to use the same material during scanning.
            SpatialMappingManager.Instance.SetSurfaceMaterial(defaultMaterial);

            // Register for the MakePlanesComplete event.
            SurfaceMeshesToPlanes.Instance.MakePlanesComplete += SurfaceMeshesToPlanes_MakePlanesComplete;
        }

        /// <summary>
        /// Called once per frame.
        /// </summary>
        private void Update()
        {
            // Check to see if the spatial mapping data has been processed yet.
            if (!meshesProcessed)
            {
                // Check to see if enough scanning time has passed
                // since starting the observer.
                if ((Time.unscaledTime - SpatialMappingManager.Instance.StartTime) < scanTime)
                {
                    // If we have a limited scanning time, then we should wait until
                    // enough time has passed before processing the mesh.
                }
                else
                {
                    // The user should be done scanning their environment,
                    // so start processing the spatial mapping data...

                    if (SpatialMappingManager.Instance.IsObserverRunning())
                    {
                        // Stop the observer.
                        SpatialMappingManager.Instance.StopObserver();
                    }

                    // Call CreatePlanes() to generate planes.
                    CreatePlanes();

                    // Set meshesProcessed to true.
                    meshesProcessed = true;
                }
            }
        }

        /// <summary>
        /// Handler for the SurfaceMeshesToPlanes MakePlanesComplete event.
        /// </summary>
        /// <param name="source">Source of the event.</param>
        /// <param name="args">Args for the event.</param>
        private void SurfaceMeshesToPlanes_MakePlanesComplete(object source, System.EventArgs args)
        {
            // Collection of floor planes that we can use to set horizontal items on.
            List<GameObject> floors = new List<GameObject>();
            floors = SurfaceMeshesToPlanes.Instance.GetActivePlanes(PlaneTypes.Floor);

            // Check to see if we have enough floors (minimumFloors) to start processing.
            if (floors.Count >= minimumFloors)
            {
                // Reduce our triangle count by removing any triangles
                // from SpatialMapping meshes that intersect with active planes.
                RemoveVertices(SurfaceMeshesToPlanes.Instance.ActivePlanes);

                // After scanning is over, switch to the secondary (occlusion) material.
                SpatialMappingManager.Instance.SetSurfaceMaterial(secondaryMaterial);
            }
            else
            {
                // Re-enter scanning mode so the user can find more surfaces before processing.
                SpatialMappingManager.Instance.StartObserver();

                // Re-process spatial data after scanning completes.
                meshesProcessed = false;
            }
        }

        /// <summary>
        /// Creates planes from the spatial mapping surfaces.
        /// </summary>
        private void CreatePlanes()
        {
            // Generate planes based on the spatial map.
            SurfaceMeshesToPlanes surfaceToPlanes = SurfaceMeshesToPlanes.Instance;
            if (surfaceToPlanes != null && surfaceToPlanes.enabled)
            {
                surfaceToPlanes.MakePlanes();
            }
        }

        /// <summary>
        /// Removes triangles from the spatial mapping surfaces.
        /// </summary>
        /// <param name="boundingObjects"></param>
        private void RemoveVertices(IEnumerable<GameObject> boundingObjects)
        {
            RemoveSurfaceVertices removeVerts = RemoveSurfaceVertices.Instance;
            if (removeVerts != null && removeVerts.enabled)
            {
                removeVerts.RemoveSurfaceVerticesWithinBounds(boundingObjects);
            }
        }

        /// <summary>
        /// Called when the GameObject is unloaded.
        /// </summary>
        protected override void OnDestroy()
        {
            if (SurfaceMeshesToPlanes.Instance != null)
            {
                SurfaceMeshesToPlanes.Instance.MakePlanesComplete -= SurfaceMeshesToPlanes_MakePlanesComplete;
            }

            base.OnDestroy();
        }
    }
}
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219163851771.png)

- `Surface Meshes To Planes` 脚本能够**将扫描的网格转换为实体**。
  - **Draw Planes** 为需要转换的类型。
  - **Destory Planes** 为需要丢弃的类型。
  - 我这里这两个参数都使用了默认值，即保留了 Wall、Floor、Ceiling、Table 类型的网格数据。 
- `Remove Surface Vertices` 脚本能够**把与实体重合的网格删除**。
- `SpatialProcessing` 脚本用于**处理网格数据**。
  - **Scan Time** : 扫描过多少秒开始转换
  - **Default Material**: 扫描时使用的材质，这里使用 MRTK 包中的 `WireframeBlue`。
  - **secondaryMaterial**: 停止扫描时使用的材质，这里使用 MRTK 包中的 `Occlusion`，注意路径是 `HoloToolKit/SpatialMapping/Materials/Occlusion.mat`。
  - **minimumFloors**: 结束处理所需要的最小 floor 数量。

（5）为 Cube 添加 MRTK 包下的 `TapToPlace.cs` 脚本。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219165758785.png)

（6）使用真机运行程序，不要忘记添加 `SpatialPerception` 权限：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219163517509.png)

程序启动后，会先扫描空间信息：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219170922809.png)

当扫描结束后，我们就可以把 Cube 放在实际的物体上，比如墙壁上：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219171009803.png)

## 二、Spatial UnderStanding

不知道你在运行上面的程序时，有没有尝试过，在扫描结束后你走动到之前没有扫描到的地方，这时候就无法将Cube放置在实际的物体上了。

这也很好理解，程序在启动的一段时间内扫描空间数据，扫描结束后将其转换为（房屋）模型，你实际上放到的是在（房屋）模型上（不信你先扫描一个椅子，扫描结束后将椅子移走，Cube 只能放在椅子原来的位置上）。而我们之前没有扫描到的地方，自然没有（房屋）模型，因此无法放置。

HoloLens 为我们提供了 `Spatial UnderStanding` 的功能，能够让 HoloLens 实时扫描空间数据，实时更新（房屋）模型。当然这样会占用较大的 CPU 资源。

MRTK 工具包为我们提供了 `SpatialUnderstanding`，直接将其拖入 Manager 下即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219171936148.png)

重新运行程序，我们发现是在实时扫描的，扫描到的部分被蓝色网格所覆盖。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219172601972.png)

查看下开启 SpatialUnderstanding 的 CPU 使用情况：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219172628898.png)

## 三、Anchor

如果我们查看 Cube 上的 `TapToPlace` 脚本的源码的话，我们会发现它内部调用了 WorldAnchorManager 来实现锚点的管理。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219172808951.png)

因此理论上我们给任一 GameObject 添加上 `WorldAnchorManager` 脚本，就能够实现锚点管理。但是遗憾的是，不知道是不是我打开姿势不对，还是什么原因，即时添加了 `WorldAnchorManager` 脚本，仍然无法实现锚点的效果，有实现的小伙伴可以留言告诉我下。

因此，我只能放弃使用官方提供的 `WorldAnchorManager`，使用在[《HoloLens 开发笔记（10）——World Anchor》](/c957c529.html) 中的方法，自己实现锚点效果。

使用如下代码覆盖 `TapToPlace`  脚本即可：

```csharp
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.

using System.Collections.Generic;
using UnityEngine;
using HoloToolkit.Unity.InputModule;
using UnityEngine.XR.WSA.Persistence;
using System.Linq;
using UnityEngine.XR.WSA;

namespace HoloToolkit.Unity.SpatialMapping
{
    /// <summary>
    /// The TapToPlace class is a basic way to enable users to move objects 
    /// and place them on real world surfaces.
    /// Put this script on the object you want to be able to move. 
    /// Users will be able to tap objects, gaze elsewhere, and perform the tap gesture again to place.
    /// This script is used in conjunction with GazeManager, WorldAnchorManager, and SpatialMappingManager.
    /// </summary>
    [RequireComponent(typeof(Collider))]
    [RequireComponent(typeof(Interpolator))]
    public class TapToPlace : MonoBehaviour, IInputClickHandler
    {
        [Tooltip("Distance from camera to keep the object while placing it.")]
        public float DefaultGazeDistance = 2.0f;

        [Tooltip("Place parent on tap instead of current game object.")]
        public bool PlaceParentOnTap;

        [Tooltip("Specify the parent game object to be moved on tap, if the immediate parent is not desired.")]
        public GameObject ParentGameObjectToPlace;

        /// <summary>
        /// Keeps track of if the user is moving the object or not.
        /// Setting this to true will enable the user to move and place the object in the scene.
        /// Useful when you want to place an object immediately.
        /// </summary>
        [Tooltip("Setting this to true will enable the user to move and place the object in the scene without needing to tap on the object. Useful when you want to place an object immediately.")]
        public bool IsBeingPlaced;

        [Tooltip("Setting this to true will allow this behavior to control the DrawMesh property on the spatial mapping.")]
        public bool AllowMeshVisualizationControl = true;

        [Tooltip("Should the center of the Collider be used instead of the gameObjects world transform.")]
        public bool UseColliderCenter;

        private Interpolator interpolator;

        WorldAnchorStore AnchorStore;

        string ObjectAnchorStoreName;

        /// <summary>
        /// The default ignore raycast layer built into unity.
        /// </summary>
        private const int IgnoreRaycastLayer = 2;

        private Dictionary<GameObject, int> layerCache = new Dictionary<GameObject, int>();
        private Vector3 PlacementPosOffset;

        protected virtual void Start()
        {
            WorldAnchorStore.GetAsync(AnchorStoreReady);
            ObjectAnchorStoreName = gameObject.name;

            if (PlaceParentOnTap)
            {
                ParentGameObjectToPlace = GetParentToPlace();
                PlaceParentOnTap = ParentGameObjectToPlace != null;
            }

            interpolator = EnsureInterpolator();

            if (IsBeingPlaced)
            {
                StartPlacing();
            }
            else // If we are not starting out with actively placing the object, give it a World Anchor
            {
                AttachWorldAnchor();
            }
        }

        private void AnchorStoreReady(WorldAnchorStore store)
        {
            AnchorStore = store;

            if (AnchorStore.GetAllIds().Contains(ObjectAnchorStoreName))
            {
                AnchorStore.Load(ObjectAnchorStoreName, gameObject);
            }
        }

        private void OnEnable()
        {
            Bounds bounds = transform.GetColliderBounds();
            PlacementPosOffset = transform.position - bounds.center;
        }

        /// <summary>
        /// Returns the predefined GameObject or the immediate parent when it exists
        /// </summary>
        /// <returns></returns>
        private GameObject GetParentToPlace()
        {
            if (ParentGameObjectToPlace)
            {
                return ParentGameObjectToPlace;
            }

            return gameObject.transform.parent ? gameObject.transform.parent.gameObject : null;
        }

        /// <summary>
        /// Ensures an interpolator on either the parent or on the GameObject itself and returns it.
        /// </summary>
        private Interpolator EnsureInterpolator()
        {
            var interpolatorHolder = PlaceParentOnTap ? ParentGameObjectToPlace : gameObject;
            return interpolatorHolder.EnsureComponent<Interpolator>();
        }

        protected virtual void Update()
        {
            if (!IsBeingPlaced) { return; }
            Transform cameraTransform = CameraCache.Main.transform;

            Vector3 placementPosition = GetPlacementPosition(cameraTransform.position, cameraTransform.forward, DefaultGazeDistance);

            if (UseColliderCenter)
            {
                placementPosition += PlacementPosOffset;
            }

            // Here is where you might consider adding intelligence
            // to how the object is placed.  For example, consider
            // placing based on the bottom of the object's
            // collider so it sits properly on surfaces.

            if (PlaceParentOnTap)
            {
                placementPosition = ParentGameObjectToPlace.transform.position + (placementPosition - gameObject.transform.position);
            }

            // update the placement to match the user's gaze.
            interpolator.SetTargetPosition(placementPosition);

            // Rotate this object to face the user.
            interpolator.SetTargetRotation(Quaternion.Euler(0, cameraTransform.localEulerAngles.y, 0));
        }

        public virtual void OnInputClicked(InputClickedEventData eventData)
        {
            // On each tap gesture, toggle whether the user is in placing mode.
            IsBeingPlaced = !IsBeingPlaced;
            HandlePlacement();
            eventData.Use();
        }

        private void HandlePlacement()
        {
            if (IsBeingPlaced)
            {
                StartPlacing();
            }
            else
            {
                StopPlacing();
            }
        }
        private void StartPlacing()
        {
            var layerCacheTarget = PlaceParentOnTap ? ParentGameObjectToPlace : gameObject;
            layerCacheTarget.SetLayerRecursively(IgnoreRaycastLayer, out layerCache);
            InputManager.Instance.PushModalInputHandler(gameObject);

            ToggleSpatialMesh();
            RemoveWorldAnchor();
        }

        private void StopPlacing()
        {
            var layerCacheTarget = PlaceParentOnTap ? ParentGameObjectToPlace : gameObject;
            layerCacheTarget.ApplyLayerCacheRecursively(layerCache);
            InputManager.Instance.PopModalInputHandler();

            ToggleSpatialMesh();
            AttachWorldAnchor();
        }

        private void AttachWorldAnchor()
        {
            WorldAnchor anchor = gameObject.AddComponent<WorldAnchor>();

            if (anchor.isLocated)
            {
                AnchorStore.Save(ObjectAnchorStoreName, anchor);
            }
            else
            {
                anchor.OnTrackingChanged += Anchor_OnTrackingChanged;
            }
        }

        void Anchor_OnTrackingChanged(WorldAnchor self, bool located)
        {
            if (located)
            {
                AnchorStore.Save(ObjectAnchorStoreName, self);
                // 取消事件监听
                self.OnTrackingChanged -= Anchor_OnTrackingChanged;
            }
        }

        private void RemoveWorldAnchor()
        {
            WorldAnchor anchor = gameObject.GetComponent<WorldAnchor>();
            if (anchor != null)
            {
                DestroyImmediate(anchor);
            }

            if (AnchorStore.GetAllIds().Contains(ObjectAnchorStoreName))
            {
                AnchorStore.Delete(ObjectAnchorStoreName);
            }
        }

        /// <summary>
        /// If the user is in placing mode, display the spatial mapping mesh.
        /// </summary>
        private void ToggleSpatialMesh()
        {
            if (SpatialMappingManager.Instance != null && AllowMeshVisualizationControl)
            {
                SpatialMappingManager.Instance.DrawVisualMeshes = IsBeingPlaced;
            }
        }

        /// <summary>
        /// If we're using the spatial mapping, check to see if we got a hit, else use the gaze position.
        /// </summary>
        /// <returns>Placement position in front of the user</returns>
        private static Vector3 GetPlacementPosition(Vector3 headPosition, Vector3 gazeDirection, float defaultGazeDistance)
        {
            RaycastHit hitInfo;
            if (SpatialMappingRaycast(headPosition, gazeDirection, out hitInfo))
            {
                return hitInfo.point;
            }
            return GetGazePlacementPosition(headPosition, gazeDirection, defaultGazeDistance);
        }

        /// <summary>
        /// Does a raycast on the spatial mapping layer to try to find a hit.
        /// </summary>
        /// <param name="origin">Origin of the raycast</param>
        /// <param name="direction">Direction of the raycast</param>
        /// <param name="spatialMapHit">Result of the raycast when a hit occurred</param>
        /// <returns>Whether it found a hit or not</returns>
        private static bool SpatialMappingRaycast(Vector3 origin, Vector3 direction, out RaycastHit spatialMapHit)
        {
            if (SpatialMappingManager.Instance != null)
            {
                RaycastHit hitInfo;
                if (Physics.Raycast(origin, direction, out hitInfo, 30.0f, SpatialMappingManager.Instance.LayerMask))
                {
                    spatialMapHit = hitInfo;
                    return true;
                }
            }
            spatialMapHit = new RaycastHit();
            return false;
        }

        /// <summary>
        /// Get placement position either from GazeManager hit or in front of the user as backup
        /// </summary>
        /// <param name="headPosition">Position of the users head</param>
        /// <param name="gazeDirection">Gaze direction of the user</param>
        /// <param name="defaultGazeDistance">Default placement distance in front of the user</param>
        /// <returns>Placement position in front of the user</returns>
        private static Vector3 GetGazePlacementPosition(Vector3 headPosition, Vector3 gazeDirection, float defaultGazeDistance)
        {
            if (GazeManager.Instance.HitObject != null)
            {
                return GazeManager.Instance.HitPosition;
            }
            return headPosition + gazeDirection * defaultGazeDistance;
        }
    }
}
```

运行程序，将 Cube 放置在椅子上，重新运行程序，Cube 会被还原到椅子上。

>PS：注意去除 `is Being Placed` 选项，不然程序每次启动 Cube 都会处于可移动状态。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219174316587.png)
