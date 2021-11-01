---
title: 首次排查 OOM 实录
categories:
  - Java
  - 性能分析
tags: [jvisualvm, OOM]
abbrlink: f4adeb1d
date: 2020-04-19 11:18:15
related_repos:
  - name: oom demo
    url: https://github.com/jitwxs/disruptor-study/blob/master/disruptor-demo/src/test/java/jit/wxs/disruptor/demo/oom
---

## 一、前言

距离上篇文章更新已经一月有余，之所以一直没更新一是工作最近比较忙，二是感觉产出不了什么对自己和他人有价值的文章。因此这段时间，主要的空闲时间在学习技术和写 GitHub，博客这边就暂时落下了。

本篇文章的落成更像是一篇笔记，而不是博客。因为在一年的工作后，首次碰上了 OOM 问题，虽然导致的原因比较简单，但也算是值得纪念的，哈哈。

## 二、问题复现

问题原因和 Disruptor 相关，如果不了解的同学，就把它理解成一个首尾相连的环形 Queue 就 OK 了。

### 2.1 代码实现

首先创建 Disruptor 存放的实体类 Entity，它有个对象叫 dataList，存放的是 EntityData 的引用：

```java
@Data
public class Entity {
    private long id;

    private List<EntityData> dataList;
}

@Data
@NoArgsConstructor(access = AccessLevel.PRIVATE)
@AllArgsConstructor
public class EntityData {
    private long id;

    private String message;
}
```

创建 Disruptor 消费者，从队列中消费数据，打印了当前消费的 sequence，以及 dataList 中引用的对象数量：

```java
class EntityEventHandler implements EventHandler<Entity> {

    @Override
    public void onEvent(Entity event, long sequence, boolean endOfBatch) throws Exception {
        // 从 ringBuffer 中消费数据
        System.out.println("EntityEventHandler Sequence: " + sequence + ", subList size: " + event.getDataList().size());
    }
}
```

创建 Disruptor 生产者，通过调用 `publish()` 方法将数据放入队列中：

```java
class EntityEventTranslator {
    private final RingBuffer<Entity> ringBuffer;

    public EntityEventTranslator(RingBuffer<Entity> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    private static final EventTranslatorTwoArg<Entity, Long, List<EntityData>> TRANSLATOR = (event, sequence, id, dataList) -> {
        event.setId(id);
        event.setDataList(dataList);
    };

    public void publish(Long id, List<EntityData> dataList) {
        ringBuffer.publishEvent(TRANSLATOR, id, dataList);
    }
}
```

创建运行的主类，主要的业务操作就是一个死循环去生产数据。

```java
public class OOMTest {
    private static int BUFFER_SIZE = 65536;

    public static void main(String[] args) throws InterruptedException {
        Disruptor<Entity> disruptor = new Disruptor<>(Entity::new, BUFFER_SIZE, DaemonThreadFactory.INSTANCE, ProducerType.SINGLE, new BlockingWaitStrategy());

        // 2. 添加消费者
        disruptor.handleEventsWith(new EntityEventHandler());

        // 3. 启动 Disruptor
        RingBuffer<Entity> ringBuffer = disruptor.start();

        // 4. 创建生产者
        EntityEventTranslator producer = new EntityEventTranslator(ringBuffer);

        // 5. 死循环发送事件
        while (true) {
            long id = RandomUtils.nextLong(1, 100000);
            List<EntityData> dataList = mockData(RandomUtils.nextInt(10, 1000));

            producer.publish(id, dataList);

            TimeUnit.MILLISECONDS.sleep(10);
        }
    }

    private static List<EntityData> mockData(int size) {
        List<EntityData> result = Lists.newArrayListWithCapacity(size);
        for(int i = 0; i < size; i++) {
            result.add(new EntityData(RandomUtils.nextLong(100000, 1000000), RandomStringUtils.randomAlphabetic(1, 100)));
        }
        return result;
    }
}
```

### 2.2 Java VisualVM

`Java VisualVM` 是自 JDK 1.6 起，安装时自带的 JVM 监控工具，本次我们使用它来监控程序运行后的堆内存情况。运行程序，启动 Java VisualVM 并连接上程序。可以观察到，随着程序的不断运行，堆的大小不断扩大。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200502204332810.png)

### 2.3 复现 OOM

下面通过控制 JVM 的堆大小，来达到 OOM 的效果。在 IDEA 中配置运行主类的启动参数如下：

```
-Xms256m -Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=D:\disruptor_oom.hprof
```

第一个指定了堆的初始大小，第二个指定了堆的最大大小，第三个指定了当 JVM 发生 OOM 时，自动生成 dump 文件，第四个指定了 dump 文件的位置。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/2020050220441815.png)

重新运行程序，由于我们配置了 Xmx，因此当内存增长到阈值时，触发 OOM，dump 文件生成在指定位置。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200502204448955.png)

## 三、Heap Dump

### 3.1 分析 Dump

Eclipse 的 `MemoryAnalyzer` 是目前最为常用的 dump 文件分析工具，有 Ecplise 插件版和独立版两种。由于我们使用的 IDE 是 IDEA，因此[安装独立版](https://www.eclipse.org/mat/downloads.php)即可。

安装完毕后，点击 `File -> Open Heap Dump` 打开生成的 dump 文件，等待右下角加载完毕后，选择 `Leak Suspects Report` 打开分析报告。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/2020050220452020.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200502204545100.png)

由于使用的这个 Case 没有干扰项，所以报告刚打开，`Override` 页其实就已经展示出了问题：

```
com.lmax.disruptor.RingBuffer @ 0xf8153f78
Shallow Size: 144 B Retained Size: 232.6 MB
```

我们假装看不见，点击 Leak Suspects，查看内存泄露分析报告。如图中 ① 所示，报告只出现了一个问题。实际项目这里可能会展示多个，你需要去找到真正导致 OOM 的那一个问题。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200502204621957.png)

在 ② 处和 ③ 处，重点关注 `Shallow Heap` 和 `Retained Heap` 两个字段。专业的解释比较晦涩难懂，用大白话来说就是： `Shallow Heap` 表示对象自身的占用大小；`Retained Heap` 表示对象自身及其 GC 后可被释放的所有引用对象的大小。

以下面代码为例，`Shallow Heap` 表示 A 自身的大小，`Retained Heap` 表示 A + B 的大小（严谨的说，只有 B 不被其他对象所引用，即 GC 在释放掉 A 时也能释放掉 B 的话，Retained Heap 才为二者和）。

```java
class A { B b; }

class B { String sss; }
```

回到报告，在 ② 处说明了 RingBuffer 队列底层的 entries 数组，其 `Shallow Heap` 只有 144 byte，而 `Retained Heap` 有 243879304 byte，说明其引用了大量可以被 GC 的对象。在 ③ 处展示 entries 数组中每一项元素，也可以看出来。

### 3.2 问题原因

问题的原因就是这个 entries 数组太大了，大的原因是 Entity 对象有个 EntityData 的 dataList，每个 Entity 都持有了许多个 EntityData，导致  entries 数组较大。

虽然这些 Entity 对象及其持有的 EntityData 都是可以被 GC 的，但是不幸的是还没等到 GC 就已经 OOM 了。因此想要解决这个问题，就得让程序能够撑到 GC。

将程序主类中的 `BUFFER_SIZE` 从 65536 下调到 128，重新运行再试试看。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200502204647774.png)

可以看到程序内存保持稳定。

`BUFFER_SIZE` 设置的就是这个 entries 数组的长度，前面说过可以把它理解成一个首尾相连的环形 Queue，那么当 entries 数组满了，又从头开始覆盖。当被覆盖后，原对象就失去了引用，就可以被 GC。

## 四、结语

生成 dump 文件的方式一般有两种，要么是在程序启动时，通过 JVM 参数，在出现 OOM 时自动 dump，就如同 2.3 节那样；另一种方式是通过命令，把当前的程序进行 dump：

```
jmap -dump:[live,]format=b,file=fileName [pid]
```

`jmap` 也是在安装 JDK 时自动安装的小工具，可以帮助我们对堆进行 dump。`live` 参数是可选的，选中后只输出活跃的对象到 dunmp文件，最后的 pid 指定 java 程序运行的进程号。例如：

```
jmap -dump:format=b,file=/home/admin/disruptor_oom.hprof 1235
```

另外本文使用到的 `Java VisualVM` 和 `MemoryAnalyzer` 这两款工具，并没有详细介绍其功能，后续我将专门辟文，去介绍 Java 性能分析中常用的工具。
