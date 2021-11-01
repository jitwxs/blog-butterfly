---
title: HoloLens 开发笔记（7）——Voice
categories: HoloLens
abbrlink: 2541026f
date: 2018-12-03 12:41:39
related_repos:
  - name: HoloLens Demo
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
---

本篇文章来学习 HoloLens 的基础开发之语音操作。

创建一个新的 Unity 项目 VoiceDemo，初始化项目：

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

## 一、语音控制

在 Hierarchy 创建一个空的 gameObject 并重命名为 `SpeechManager`，为其添加 MRTK 的 `SpeechInputSource.cs` 脚本，为其添加两个关键字，分别是 start rotate 和 stop rotate。

> 1. 后面对应的 `Key Shortcut`，字面意思是对应的键盘按键，但是我在实际程序中并没有体会到有啥用，所以就随便设了两个值，有知道的同学可以留言告诉我下。
> 2. HoloLens 当前已经支持中文语音，但需要手动安装刷机，[参考官方](https://docs.microsoft.com/en-us/hololens/hololens-install-localized)。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218225322927.png)

| 属性                       | 描述                                                         |
|:-------------------------- |:------------------------------------------------------------ |
| PersistentKeywords         | Keyword 在所有场景中都是持久的，此语音输入源实例在加载新场景时不会被销毁 |
| RecognizerStart            | 是否在启动时激活识别器                                       |
| recognitionConfidenceLevel | Keyword 识别器的置信度                                       |

新建一个脚本 `CubRotate.cs`，并将其添加到 Cube 上。

```csharp
using UnityEngine;

public class CubRotate : MonoBehaviour {
    bool HasRotate = false;
    
	void Update () {
        if(HasRotate)
        {
            transform.Rotate(Vector3.up);
        }
    }

    public void StartRotate()
    {
        HasRotate = true;
    }

    public void StopRotate()
    {
        HasRotate = false;
    }
}
```

这个脚本十分简单，调用 `StartRotate()` 方法就能够使 Cube 开始旋转，调用 `StopRotate()` 方法使 Cube 停止旋转。

为 Cube 添加 MRTK 中的 `SpeechInputHandler.cs` 脚本，根据名字就可以看出和 SpeechInputSource.cs 脚本有关系，该脚本用于处理语音输入。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/2018121823033761.png)

| 属性               | 描述                                                         |
|:------------------ |:------------------------------------------------------------ |
| PersistentKeywords | Keyword 在所有场景中都是持久的，此语音输入源实例在加载新场景时不会被销毁 |
| IsGlobalListener   | 确定该处理程序是否是一个全局侦听器，而不是连接到特定的GameObject。 |

在该脚本中，我们添加了两个要处理的关键字，也就是在 SpeechInputSource.cs 中设置的 **start rotate** 和 **stop rotate**。在对应的 `Response()` 中调用了 Cube 的 CubeRotate.StartRotate() 和 CubeRotate.StopRotate() 方法。

运行程序，因为使用到了语音，所以必须使用真机运行，在运行前，不要忘记添加 Microphone 的权限。在 `Edit/Project Settings/Player/Publishing Settings/Capabilities`中勾选 Microphone 。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181218231002396.png)

部署到真机上，通过说 start rotate 和 stop rotate，来观察 Cube 的旋转和停止。

## 二、操纵麦克风

下面实现一个在耳机中播放麦克风录入的声音，并能够根据声音音量调整 Cube 的大小。

新建一个脚本 `CubeMic.cs`，并将其添加到 Cube 上。

```csharp
using HoloToolkit.Unity.InputModule;
using UnityEngine;

public class CubeMic : MonoBehaviour {
    // Cube原始大小
    private Vector3 origScale;

    // 当前麦克风"音量"
    private float averageAmplitude = 0;

    void Start()
    {
        // 保存Cube原始大小
        origScale = transform.localScale;
        // 设置麦克风音量
        MicStream.MicSetGain(10);
        // 开启麦克风
        MicStream.MicStartStream(false, false);
    }

    // 声音过滤
    private void OnAudioFilterRead(float[] buffer, int numChannels)
    {
        // 将麦克风输入到声音过滤管线中，将麦克风的声音从耳机播放出来
        MicStream.MicGetFrame(buffer, buffer.Length, numChannels);

        // 计算麦克风"音量"大小
        float sumOfValues = 0;
        for (int i = 0; i < buffer.Length; i++)
        {
            sumOfValues += Mathf.Abs(buffer[i]);
        }
        averageAmplitude = sumOfValues / buffer.Length;
    }

    void Update()
    {
        // 根据"音量"调整Cube大小
        transform.localScale = origScale * (1 + averageAmplitude * 10);
    }
}
```

总结一下代码：

- **MicStream** ：

  HoloToolkit提供的麦克风操作类，详细的用法可参考工具包中的MicStreamDemo类

- **OnAudioFilterRead** ：

  Unity引擎提供的声音滤波函数，具体原理可参考官方文档《OnAudioFilterRead》

- **MicStream.MicGetFrame(…)** ：

  这个方法可以获取到麦克风的帧数据(float[])，可以在类似 OnAudioFilterRead 或者 Update 等高频事件中调用并获取。因为获取到的是麦克风最小数据单元，使用起来非常灵活。我们可以在 OnAudioFilterRead 中播放，也可以使用Socke实现远程通话。

为 Cube 添加一个 `Audio Souce` 组件，用于播放声音，它的属性使用默认值即可。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812/20181219142042993.png)

在真机中运行程序，我们能够听见麦克风的声音，并且 Cube 根据音量大小发生改变。

## 三、设置合理的关键字

- 不要使用单音节词，避免被系统忽略。例如使用 Play Video 替代 Play。也要注意不要音节过多，增加用户使用成本。
- 不要使用系统预置语音，防止歧义，例如 Select、Remove等。
- 避免押韵的语音，例如使用 Show Store 替代 Show More。

## 四、语音的底层实现

使用 MRTK 工具包，我们只需要点点鼠标就能够实现语音的处理，有兴趣的同学可以了解下它源码的实现。

> 本节代码来源于：[MR Basics 101: Complete project with device: Chapter 4 - Voice](https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-101#chapter-4---voice)

### 4.1 SpeechManager 

```csharp
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.Windows.Speech;

public class SpeechManager : MonoBehaviour
{
    KeywordRecognizer keywordRecognizer = null;
    Dictionary<string, System.Action> keywords = new Dictionary<string, System.Action>();

    // Use this for initialization
    void Start()
    {
        keywords.Add("Reset world", () =>
        {
            // Call the OnReset method on every descendant object.
            this.BroadcastMessage("OnReset");
        });

        keywords.Add("Drop Sphere", () =>
        {
            var focusObject = GazeGestureManager.Instance.FocusedObject;
            if (focusObject != null)
            {
                // Call the OnDrop method on just the focused object.
                focusObject.SendMessage("OnDrop", SendMessageOptions.DontRequireReceiver);
            }
        });

        // Tell the KeywordRecognizer about our keywords.
        keywordRecognizer = new KeywordRecognizer(keywords.Keys.ToArray());

        // Register a callback for the KeywordRecognizer and start recognizing!
        keywordRecognizer.OnPhraseRecognized += KeywordRecognizer_OnPhraseRecognized;
        keywordRecognizer.Start();
    }

    private void KeywordRecognizer_OnPhraseRecognized(PhraseRecognizedEventArgs args)
    {
        System.Action keywordAction;
        if (keywords.TryGetValue(args.text, out keywordAction))
        {
            keywordAction.Invoke();
        }
    }
}
```

（1）创建一个 `Dictionary<string, System.Action>` 的集合，向集合中添加`Reset world`和`Drop Sphere`：

```csharp
keywords.Add("Reset world", () =>
{
    this.BroadcastMessage("OnReset");
});

keywords.Add("Drop Sphere", () =>
{
    var focusObject = GazeGestureManager.Instance.FocusedObject;
    if (focusObject != null)
    {
        focusObject.SendMessage("OnDrop", SendMessageOptions.DontRequireReceiver);
    }
});
```

在 Drop Sphere 中，如果凝聚对象非空的话，向其推送 OnDrop 消息。

（2）初始化一个语音识别器，将集合key值数组传入。

```csharp
keywordRecognizer = new KeywordRecognizer(keywords.Keys.ToArray());
```

（3）注册回调并启动识别器。

```csharp
keywordRecognizer.OnPhraseRecognized += KeywordRecognizer_OnPhraseRecognized;
keywordRecognizer.Start();
```

```csharp
private void KeywordRecognizer_OnPhraseRecognized(PhraseRecognizedEventArgs args)
{
    System.Action keywordAction;
    if (keywords.TryGetValue(args.text, out keywordAction))
    {
        keywordAction.Invoke();
    }
}
```

keywords.TryGetValue(args.text, out keywordAction) 获取用户的语音，如果存在于集合中，返回true，并将捆绑的Action存入 keywordAction，调用 Invoke() 反射执行集合中对应的方法。

### 4.2 SphereCommands

```csharp
using UnityEngine;

public class SphereCommands : MonoBehaviour
{
    Vector3 originalPosition;

    void Start()
    {
        // 获取启动时球的初始位置
        originalPosition = this.transform.localPosition;
    }

    // Called by GazeGestureManager when the user performs a Select gesture
    void OnSelect()
    {
        // If the sphere has no Rigidbody component, add one to enable physics.
        if (!this.GetComponent<Rigidbody>())
        {
            var rigidbody = this.gameObject.AddComponent<Rigidbody>();
            rigidbody.collisionDetectionMode = CollisionDetectionMode.Continuous;
        }
    }

    // Called by SpeechManager when the user says the "Reset world" command
    void OnReset()
    {
        // If the sphere has a Rigidbody component, remove it to disable physics.
        var rigidbody = this.GetComponent<Rigidbody>();
        if (rigidbody != null)
        {
            rigidbody.isKinematic = true;
            Destroy(rigidbody);
        }

        // Put the sphere back into its original local position.
        this.transform.localPosition = originalPosition;
    }

    // Called by SpeechManager when the user says the "Drop sphere" command
    void OnDrop()
    {
        // Just do the same logic as a Select gesture.
        OnSelect();
    }
}
```

（1）启动时获取球的初始位置。

```csharp
originalPosition = this.transform.localPosition;
```

（2）`OnDrop()` 方法中直接调用 `OnSelect()` 方法。

（3）`OnReset()` 方法中，获取刚体组件，如果存在，开启动力学开关，并将其销毁。

```csharp
if (rigidbody != null)
{
    rigidbody.isKinematic = true;
    Destroy(rigidbody);
}
```

（4）将球位置替换为初始位置。

```csharp
this.transform.localPosition = originalPosition;
```

### 4.3 SendMessage

介绍下上面代码中使用到的 `SendMessage` 函数。

**（1）SendMessage...**

- `SendMessage`

```csharp
public void SendMessage(string methodName, object value = null, SendMessageOptions options = SendMessageOptions.RequireReceiver);
```

调用一个对象的methodName函数（公有 or 私有均可），后面跟一个可选参数（函数入参）。

- `SendMessageUpwards`

```csharp
public void SendMessageUpwards(string methodName, object value = null, SendMessageOptions options = SendMessageOptions.RequireReceiver);
```

类似于 SendMessage ，但是它不仅会向当前对象推送消息，也会向这个对象的父对象推送这个消息（遍历所有父对象推送）。

- `BroadcastMessage`

```csharp
public void BroadcastMessage(string methodName, object parameter = null, SendMessageOptions options = SendMessageOptions.RequireReceiver);
```

类似于 SendMessage ，但是它不仅会向当前对象推送消息，也会向这个对象的子对象推送这个消息（遍历所有子对象推送）。

**（2）SendMessageOptions**

- `RequireReceive`：如果没有找到相应函数，会报错
- `DontRequireReceive`：如果没有找到相应函数，不会报错
