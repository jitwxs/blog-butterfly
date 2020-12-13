---
title: Disruptor 官方文档——入门
typora-root-url: ..
categories:
  - Java
  - Disruptor
tags: Disruptor
abbrlink: 4ae6dc03
date: 2019-12-16 22:45:52
copyright_author: LMAX-Exchange
---

## 一、Getting the Disruptor

 Disruptor jar 包可以从 maven 仓库  [Maven Central]( https://mvnrepository.com/artifact/com.lmax/disruptor") 获取。

```xml
<dependency>
  <groupId>com.lmax</groupId>
  <artifactId>disruptor</artifactId>
  <version>3.4.2</version>
</dependency>
```

## 二、Basic Event Produce and Consume

为了学习 Disruptor 的使用，这里以非常简单的例子入手：**生产者生产单个 long 型 value 传递给消费者**。为了简化消费者逻辑，这里只打印消费的 value。

首先定义携带数据的 Event： 

```java
public class LongEvent {
    private long value;

    public void set(long value) {
        this.value = value;
    }
}
```

为了使用 Disruptor 的内存预分配 event，我们需要定义一个 EventFactory： 

```java
import com.lmax.disruptor.EventFactory;

public class LongEventFactory implements EventFactory<LongEvent> {
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```

为了让消费者处理这些事件，所以我们这里定义一个事件处理器，负责打印 event： 

```java
import com.lmax.disruptor.EventHandler;

public class LongEventHandler implements EventHandler<LongEvent> {
    public void onEvent(LongEvent event, long sequence, boolean endOfBatch){
        System.out.println("Event: " + event);
    }
}
```

有了事件消费者，我们还需要事件生产者产生事件。为了简单起见，我们假设数据来源于 I/O，如：网络或者文件。由于不同版本的 Disruptor，提供了不同的方式编写生产者。 

### 2.1 Publishing Using Translators

在 Disruptor 的 3.0 版本中，由于加入了丰富的 Lambda 风格的API，可以用来帮助开发人员简化流程。所以在 3.0 版本后首选使用 `Event Publisher/Event Translator` 来发布事件。 

```java
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.EventTranslatorOneArg;

public class LongEventProducerWithTranslator {
    private final RingBuffer<LongEvent> ringBuffer;
    
    public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }
    
    private static final EventTranslatorOneArg<LongEvent, ByteBuffer> TRANSLATOR =
        new EventTranslatorOneArg<LongEvent, ByteBuffer>() {
            public void translateTo(LongEvent event, long sequence, ByteBuffer bb) {
                event.set(bb.getLong(0));
            }
        };

    public void onData(ByteBuffer bb) {
        ringBuffer.publishEvent(TRANSLATOR, bb);
    }
}
```

这种方式的另一个优势在于 Translator 代码可以被分离在单独的类中，同时也比较容易进行无依赖的单元测试。Disruptor 提供了许多不同的接口(EventTranslator, EventTranslatorOneArg, EventTranslatorTwoArg, etc.)，可以通过实现这些接口提供 translators。原因是当转换方法的参数通过对 RingBuffer 的调用传递给转换程序时，允许将转换程序表示为静态类或不捕获的 lambda（ when Java 8 rolls around ）。

### 2.2 Publishing Using the Legacy API

也可以使用 3.0 版本之前的遗留 API 构建生产者发布消息，这种方式比较原始（不推荐）： 

```java
import com.lmax.disruptor.RingBuffer;

public class LongEventProducer {
    private final RingBuffer<LongEvent> ringBuffer;

    public LongEventProducer(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    public void onData(ByteBuffer bb) {
        // Grab the next sequence
        long sequence = ringBuffer.next();
        try {
            // Get the entry in the Disruptor for the sequence
            LongEvent event = ringBuffer.get(sequence); 
            // Fill with data
            event.set(bb.getLong(0));  
        } finally {
            ringBuffer.publish(sequence);
        }
    }
}
```

从以上的代码流程编写可以看出，事件的发布比使用一个简单的队列要复杂。这是由于需要对事件预分配导致。对于消息的发布有两个阶段，首先在 RingBuffer 中声明需要的槽位，然后再发布可用的数据。必须使用 try/finally 语句块包裹消息的发布。必须先在 try 块中声明使用 RingBuffer 的槽位，然后在 finally 块中发布使用的 sequece。如果不这样做，将可能导致 Disruptor 状态的错误。具体来说，在多生产者的情况下，这将导致消费者停滞并且只有重新启动才能恢复。因此推荐使用 EventTranslator 编写 producer。

最后一步需要将以上编写的组件连接起来。虽然可以手动连接各个组件，然而那样可能比较复杂，因此提供了一个 DSL 用于构造以便简化过程。一些更复杂的选项无法通过 DSL 设置，但是对于大多数情况 DSL 还是非常适合的：

```java
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.util.DaemonThreadFactory;
import java.nio.ByteBuffer;

public class LongEventMain {
    public static void main(String[] args) throws Exception {
        // The factory for the event
        LongEventFactory factory = new LongEventFactory();

        // Specify the size of the ring buffer, must be power of 2.
        int bufferSize = 1024;

        // Construct the Disruptor
        Disruptor<LongEvent> disruptor = new Disruptor<>(factory, bufferSize, DaemonThreadFactory.INSTANCE);

        // Connect the handler
        disruptor.handleEventsWith(new LongEventHandler());

        // Start the Disruptor, starts all threads running
        disruptor.start();

        // Get the ring buffer from the Disruptor to be used for publishing.
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducer producer = new LongEventProducer(ringBuffer);

        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            producer.onData(bb);
            Thread.sleep(1000);
        }
    }
}
```

### 2.3 Using Java 8

关于对 Disruptor 的接口设计的影响之一是 Java 8，因为它使用了 [Functional Interfaces](http://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html) 去实现 Java Lambdas。Disruptor API 中的大多数接口定义都符合 Functional Interfaces 的要求，因此可以使用 Lambda 代替自定义类，可以减少所需的代码量。以上的 LongEventMain 使用 Lambdas 简化后： 

```java
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.util.DaemonThreadFactory;
import java.nio.ByteBuffer;

public class LongEventMain {
    public static void main(String[] args) throws Exception {
        // Specify the size of the ring buffer, must be power of 2.
        int bufferSize = 1024;

        // Construct the Disruptor
        Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, DaemonThreadFactory.INSTANCE);

        // Connect the handler
        disruptor.handleEventsWith((event, sequence, endOfBatch) -> System.out.println("Event: " + event));

        // Start the Disruptor, starts all threads running
        disruptor.start();

        // Get the ring buffer from the Disruptor to be used for publishing.
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            ringBuffer.publishEvent((event, sequence, buffer) -> event.set(buffer.getLong(0)), bb);
            Thread.sleep(1000);
        }
    }
}
```

可以看出使用 Lambdas 有大量的类将不再需要，如 handler，translator 等。也可以看出使用 Lambdas 简化 `publishEvent()` 只仅仅涉及到参数传递。上面的代码还可以简化成这样： 

```java
ByteBuffer bb = ByteBuffer.allocate(8);
for (long l = 0; true; l++) {
    bb.putLong(0, l);
    ringBuffer.publishEvent((event, sequence) -> event.set(bb.getLong(0)));
    Thread.sleep(1000);
}
```

不过这样会实例化一个对象去持有 `ByteBuffer bb` 并将其传递给 lambda。这会产生不必要的垃圾，如果对 GC 压力有严格要求的情况下， 则应首选将参数传递给 lambda 的调用。

假定可以使用方法引用代替匿名 lamdbas，则可以以这种方式重写示例。

```java
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.util.DaemonThreadFactory;
import java.nio.ByteBuffer;

public class LongEventMain {
    public static void handleEvent(LongEvent event, long sequence, boolean endOfBatch) {
        System.out.println(event);
    }

    public static void translate(LongEvent event, long sequence, ByteBuffer buffer) {
        event.set(buffer.getLong(0));
    }

    public static void main(String[] args) throws Exception {
        // Specify the size of the ring buffer, must be power of 2.
        int bufferSize = 1024;

        // Construct the Disruptor
        Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, DaemonThreadFactory.INSTANCE);

        // Connect the handler
        disruptor.handleEventsWith(LongEventMain::handleEvent);

        // Start the Disruptor, starts all threads running
        disruptor.start();

        // Get the ring buffer from the Disruptor to be used for publishing.
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            ringBuffer.publishEvent(LongEventMain::translate, bb);
            Thread.sleep(1000);
        }
    }
}
```

这里对 RingBuffer 的 publishEvent() 参数使用了方法引用替换了 lambda，使其更进一步简化。 

## 三、Basic Tuning Options

如果你能确定硬件和软件的环境便可以进一步对 Disruptor 的参数进行调整以提高性能。 主要有两种参数可以被调整：producer 类型和 wait strategy。 

### 3.1 Single vs. Multiple Producers

 提高并发系统的性能的最好方式是遵循 [Single Writer Principle](https://mechanical-sympathy.blogspot.com/2011/09/single-writer-principle.html)，这也适用于 Disruptor。如果在你的场景中只仅仅是单生产者，那么你可以显式设置成单生产者来提升性能： 

```java
public class LongEventMain {
    public static void main(String[] args) throws Exception {
        //.....
        // Construct the Disruptor with a SingleProducerSequencer
        Disruptor<LongEvent> disruptor = new Disruptor(
            factory, bufferSize, DaemonThreadFactory.INSTANCE, ProducerType.SINGLE, new BlockingWaitStrategy());
        //.....
    }
}
```

为了说明通过这种技术方式能提升多少性能优势，这里有一份测试类 [OneToOne performance test](https://github.com/LMAX-Exchange/disruptor/blob/master/src/perftest/java/com/lmax/disruptor/sequenced/OneToOneSequencedThroughputTest.java)，在 i7 Sandy Bridge MacBook Air的运行结果： 

#### 3.1.1 Multiple Producer

```powershell
Run 0, Disruptor=26,553,372 ops/sec
Run 1, Disruptor=28,727,377 ops/sec
Run 2, Disruptor=29,806,259 ops/sec
Run 3, Disruptor=29,717,682 ops/sec
Run 4, Disruptor=28,818,443 ops/sec
Run 5, Disruptor=29,103,608 ops/sec
Run 6, Disruptor=29,239,766 ops/sec
```

#### 3.1.2 Single Producer

```powershell
Run 0, Disruptor=89,365,504 ops/sec
Run 1, Disruptor=77,579,519 ops/sec
Run 2, Disruptor=78,678,206 ops/sec
Run 3, Disruptor=80,840,743 ops/sec
Run 4, Disruptor=81,037,277 ops/sec
Run 5, Disruptor=81,168,831 ops/sec
Run 6, Disruptor=81,699,346 ops/sec
```

### 3.2 Alternative Wait Strategies

Disruptor 默认使用的等待策略是 `BlockingWaitStrategy`。内部的 BlockingWaitStrategy 使用锁和 Condition 处理线程的 wake-up。BlockingWaitStrategy 是等待策略中最慢的，但是其对 CPU 的消耗最小并且在各种不同部署环境中能提供更加一致的性能表现。

然而，如果你对部署系统比较熟悉的话，可以通过调整等待策略参数来获取额外的性能。 

#### 3.2.1 [SleepingWaitStrategy](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/SleepingWaitStrategy.java)

`SleepingWaitStrategy` 的性能表现跟 BlockingWaitStrategy 差不多，对 CPU 的消耗也类似，但其对生产者线程的影响最小，通过使用简单的忙等循环，即 LockSupport.parkNanos(1) 来实现循环等待，在典型的 Linux 系统上会暂停一个线程约 60µs。

这样做的好处是，生产线程不需要采取任何其他增加适当计数器的动作，并且不需要发信号通知条件变量的成本。但是，生产者线程和使用者线程之间数据传递的平均延迟会更高。它适用于不需要低延迟并且对生产线程的影响较小的情况下，一个常见的用例是异步日志记录。 

#### 3.2.2 [YieldingWaitStrategy](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/YieldingWaitStrategy.java)

`YieldingWaitStrategy` 是可以使用在低延迟系统的策略之一。YieldingWaitStrategy 通过自旋（ busy spin ）以等待序列增加到适当的值。通过在循环内部调用 Thread.yield() 以允许其他队列的线程运行。在要求极高性能且事件处理线数小于 CPU **逻辑核心数**的场景中，推荐使用此策略。例如，CPU 开启了超线程。 

#### 3.2.3 [BusySpinWaitStrategy](https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/BusySpinWaitStrategy.java)

` BusySpinWaitStrategy` 是性能最好的等待策略，但是对环境有更高的限制 。在要求极高性能且事件处理线程数小于 CPU **物理核心数**的场景中，才应该使用此策略；例如，CPU 禁用了超线程。 

## 四、Clearing Objects From the Ring Buffer

当通过 Disruptor 传递数据时，对象的存活时间可能超过预期。为了避免这种情况，在事件处理结束后应当清理下事件对象。如果只有单个事件处理程序，则需要在该处理器中清除对应的对象。如果有一连串的事件处理程序，就需要在链的最末尾放置一个特定的处理程序用于清理事件对象。

```java
class ObjectEvent<T> {
    T val;

    void clear() {
        val = null;
    }
}

public class ClearingEventHandler<T> implements EventHandler<ObjectEvent<T>> {
    public void onEvent(ObjectEvent<T> event, long sequence, boolean endOfBatch) {
        // Failing to call clear here will result in the 
        // object associated with the event to live until
        // it is overwritten once the ring buffer has wrapped
        // around to the beginning.
        event.clear(); 
    }
}

public static void main(String[] args) {
    Disruptor<ObjectEvent<String>> disruptor = new Disruptor<>(
        () -> ObjectEvent<String>(), bufferSize, DaemonThreadFactory.INSTANCE);

    disruptor
        .handleEventsWith(new ProcessingEventHandler())
        .then(new ClearingObjectHandler());
}
```
