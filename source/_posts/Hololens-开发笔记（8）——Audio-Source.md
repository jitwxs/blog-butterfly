---
title: HoloLens 开发笔记（8）——Audio Source
typora-root-url: ..
categories: HoloLens
abbrlink: 602f136c
date: 2018-12-08 16:47:49
related_repos:
  - name: HoloLens Demo
    url: https://github.com/jitwxs/blog-sample/tree/master/HoloLens
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、音频资源

Unity中的音频文件分为两类：原生的音频或者是压缩的音频。

- **压缩音频**：通过从编辑器导入设置选择compressed选项，音频数据将很小，但在播放时会消耗CPU周期来解码。

- **原生音频**：对于短音效使用未压缩音频（WAV，AIFF）。音频数据将较大，但是声音在播放是不需要解码。

Unity 支持导入以下格式：.aif、.wav、.mp3、.ogg

| 格式       | MAC/PC上使用压缩格式 | 移动平台上使用的压缩格式 |
| ---------- | -------------------- | ------------------------ |
| MPEG       | Ogg Vorbis           | MP3                      |
| Ogg Vorbis | Ogg Vorbis           | MP3                      |
| WAV        | Ogg Vorbis           | MP3                      |
| AIFF       | Ogg Vorbis           | MP3                      |

Unity 中对音频文件的属性设置：

- **Force To Mono**（强制单声道）：如果启用，该音频剪辑将向下混合到单通道声音。

- **Load Type**（加载类型）：运行时Unity加载音频的方法。
  - Decompress on load（加载时解压缩）：加载后解压缩声音。使用于较小的压缩声音，以避免运行时解压缩的性能开销。(将使用比在它们在内存中压缩的多10倍或更多内存，因此大文件不要使用这个。)

  - Compressed in memory（内存中压缩）：保持声音在内存中压缩，在播放时解压缩。这有轻微的性能开销（尤其是OGG / Vorbis格式的压缩文件），因此大文件使用这个。

  - Stream from disc（从磁盘流）：直接从磁盘流读取音频数据。这只使用了原始声音占内存大小的很小一部分。使用这个用于很长的音乐。取决于硬件，一般建议1-2线程同时流。

![](/images/posts/20181119144116127.jpg)

## 二、Unity 设置

我们想要在 Unity 中利用声音插件（audio spatalizer plugin）来实现空间声音，只需要在设置菜单中 Edit > Audio > Spatializer 启用 `Microsoft HRTF` 拓展就好。

![](/images/posts/20181119143038972.jpg)

## 三、基本属性

>更多属性见：https://docs.unity3d.com/Manual/class-AudioSource.html

- **AudioClip**：指定需要播放的音频文件。

- **Mute**：是否静音。

- **Play On Awake**：音频是否在场景启动时自动播放。

- **Loop**：是否循环播放。

- **Pitch**：播放速度,取值范围在 -3 到 3 之间，设置1 为正常播放。

- **Spatial Blend**：设置声音是 2D 声音，还是 3D 声音。3D声音距离音源的距离会影响听到声音的大小，2D声音不会影响。

- 3D Sound Settings

  - **Douppler Level**（多普勒级别）：决定了[多普勒效应](https://baike.baidu.com/item/多普勒效应)将被应用到这个音频信号源的数值（如果设置为0，就是无效果）。

  - **Spread**（扩散）：设置 3D 立体声或多声道扬声器的空间的扩散角度。

  - **Volume Rolloff**：声音淡出的速度有多快。该值越高，越接近侦听器最先听到声音。

    - Logarithmic Rolloff（对数衰减）：当你接近音频源，声音响亮；但是当你越远离对象，声音下降越快，对数递减。

    - Linear Rolloff（线性衰减）：越远离音频源，声音越小，线性递减。

    - Custom Rolloff（自定义衰减）：根据自定义的衰减图形决定。

  - **Min Distance**：当距离小于设定值时，停止声音的增大。

  - **Max Distance**：当距离大于设定值时，停止声音的衰减。

  - **Spatialize**：是否开启空间化，3D 立体声需要开启。

![](/images/posts/20181119134743714.jpg)

## 四、Audio Listener
Audio Listener（声音侦听器），其实就是我们在游戏世界中的“耳朵”。我们依靠这个组件来听游戏世界中的声音，如果没有了这个组件，我们是听不到任何声音的。

**每个场景只能有1个音频侦听器正常工作。** 默认状态这个组件，是挂载到摄像机身上的。

![](/images/posts/20181119135601147.jpg)

## 五、常用函数

>更多函数见：https://docs.unity3d.com/ScriptReference/AudioSource.html

| 函数名    | 说明     |
| :-------- | :------- |
| Play()    | 播放音频 |
| Stop()    | 停止播放 |
| Pause()   | 暂停播放 |
| UnPause() | 继续播放 |

## 六、示例代码

>来源于：https://docs.microsoft.com/zh-cn/windows/mixed-reality/holograms-101#chapter-5---spatial-sound

```csharp
using UnityEngine;

public class SphereSounds : MonoBehaviour
{
    AudioSource impactAudioSource = null;
    AudioSource rollingAudioSource = null;

    bool rolling = false;

    void Start()
    {
        // Add an AudioSource component and set up some defaults
        impactAudioSource = gameObject.AddComponent<AudioSource>();
        impactAudioSource.playOnAwake = false;
        impactAudioSource.spatialize = true;
        impactAudioSource.spatialBlend = 1.0f;
        impactAudioSource.dopplerLevel = 0.0f;
        impactAudioSource.rolloffMode = AudioRolloffMode.Logarithmic;
        impactAudioSource.maxDistance = 20f;

        rollingAudioSource = gameObject.AddComponent<AudioSource>();
        rollingAudioSource.playOnAwake = false;
        rollingAudioSource.spatialize = true;
        rollingAudioSource.spatialBlend = 1.0f;
        rollingAudioSource.dopplerLevel = 0.0f;
        rollingAudioSource.rolloffMode = AudioRolloffMode.Logarithmic;
        rollingAudioSource.maxDistance = 20f;
        rollingAudioSource.loop = true;

        // Load the Sphere sounds from the Resources folder
        impactAudioSource.clip = Resources.Load<AudioClip>("Impact");
        rollingAudioSource.clip = Resources.Load<AudioClip>("Rolling");
    }

    // Occurs when this object starts colliding with another object
    void OnCollisionEnter(Collision collision)
    {
        // 两个碰撞物体的相对线性速度
        if (collision.relativeVelocity.magnitude >= 0.1f)
        {
            impactAudioSource.Play();
        }
    }

    // Occurs each frame that this object continues to collide with another object
    void OnCollisionStay(Collision collision)
    {
        Rigidbody rigid = gameObject.GetComponent<Rigidbody>();

        // 如果球滚动的速度大于预设值，且声音位播放时
        if (!rolling && rigid.velocity.magnitude >= 0.01f)
        {
            rolling = true;
            rollingAudioSource.Play();
        }
        // Stop the rolling sound if rolling slows down.
        else if (rolling && rigid.velocity.magnitude < 0.01f)
        {
            rolling = false;
            rollingAudioSource.Stop();
        }
    }

    // Occurs when this object stops colliding with another object
    void OnCollisionExit(Collision collision)
    {
        // Stop the rolling sound if the object falls off and stops colliding.
        if (rolling)
        {
            rolling = false;
            impactAudioSource.Stop();
            rollingAudioSource.Stop();
        }
    }
}
```
