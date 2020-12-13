---
title: Disruptor 官方文档——介绍
typora-root-url: ..
categories:
  - Java
  - Disruptor
tags: Disruptor
abbrlink: a7ed43af
date: 2019-12-16 22:43:27
copyright_author: LMAX-Exchange
---

理解 Disruptor 是什么最好方式就是将其与现有的比较好理解的东西比较 。Disruptor 就相当于 Java 中 `BlockingQueue`。同队列一样，Disruptor 用于在不同的线程之间进行数据交互，然而 Disruptor 也提供了一些关键的不同于队列的特性，如： 

- 广播事件至消费者，并且能遵循消费者依赖关系 
- 预分配用于存储事件内容的内存空间
- 针对极高的性能目标而实现的极度优化和无锁的设计

## 一、核心概念 Core Concepts

 在理解 Disruptor 如何工作之前，定义一些普遍存在于文档和源代码中的术语，对于倾向 DDD 的人而言，它们就是 Disruptor 领域的无处不在的语言。 

### 1.1 核心组件

#### 1.1.1 Disruptor

持有 RingBuffer、消费者线程池 Executor、消费者集合 CounsumerRepository 等引用。

#### 1.1.2 RingBuffer 环形缓冲

RingBuffer 在 3.0 版本之前被认为是 Disruptor 的主要概念。但从 Diruptor 3.0 开始，RingBuffer 只负责存储和更新 Disruptor 的数据，在一些高级的使用场景中用户也可以自定义它。

RingBuffer 是基于数组的缓存实现，存储生产和消费的 Event，它实现了阻塞队列的语义，也是创建 sequencer 与定义 WaitStartegy 的入口。**如果 RingBuffer 满了，则生产者会阻塞等待；如果 RingBuffer 空了，则消费者会阻塞等待**。

#### 1.1.3 Sequence 序列

Disruptor 使用 Sequence 来标记特定组件的处理进度，通过顺序递增的序号来编号，管理进行交换的 Event。一个 Sequence 用于跟踪标识某个特定的事件处理者（RingBuffer、Producer、Consumer）的处理进度。

Sequence 具有许多 AtomicLong 的特征，虽然使用 AtomicLong 也可以用于标识进度，但它可以防止不同的 Sequence 之间的 CPU 缓存伪共享(Flase Sharing)问题。 

#### 1.1.4 Sequencer 序列器

Sequencer 是 Disruptor 中的真正核心，主要实现生产者和消费者之间快速、正确地传递数据的并发算法。

它具有 `SingleProducerSequencer` 和 `MultiProducerSequencer` 这两个实现。 

#### 1.1.5 Sequence Barrier 序列屏障

Sequence Barrier 由 Sequencer 创建，它包含了来自 Sequencer 的已经发布的主要 sequence 的引用，或者包含了依赖的消费者的 sequence。

用于保持对 RingBuffer 的 Main Published Sequence(Producer) 和 Consumer 之间的平衡关系，它决定了 Consumer 是否还有可处理的 Event 的逻辑。

#### 1.1.6 WaitStrategy 等待策略

WaitStrategy 决定了消费者以何种方式等待生产者将 Event 放进 Disruptor 中。

#### 1.1.7 Event 事件

从生产者传到消费者的数据单元叫做 Event。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者自行定义。 

#### 1.1.8 Event Processor 事件处理器

持有特定的消费者的 Sequence，并且拥有一个主事件循环（main event loop）用于处理 Disruptor 的 Event。

其中 BatchEventProcessor 是其具体实现，实现了事件循环（event loop），并且会回调到实现了 EventHandler 的接口对象中。 

#### 1.1.9 EventHandler 事件处理逻辑

由用户实现并且代表了 Disruptor 中的一个消费者的接口，消费者相关的逻辑都需要写在这里。

#### 1.1.10 Producer 生产者

生产者，泛指调用 Disruptor 发布事件的用户代码，它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者自行定义。 

#### 1.1.11 WorkProcessor

确保每个 sequence 只被一个 processor 消费，在同一个 WorkPool 中处理多个 WorkProcessor 不会消费同样的 sequence。

### 1.2 官方示例

为了能够将这些概念关联起来，即放在一个上下文中表示，下图是个例子，展示 LMAX 在高性能的核心服务中使用 Disruptor 的场景。

![Figure 1. Disruptor with a set of dependent consumers.](/images/posts/20191216221318838.png)

在该示例中为我们展示了一个多生产者多消费者的模型，生产者和消费者的交互核心为 RingBuffer 和 Sequencer。

生产者本身不需要维护 Sequence 序列，这两个生产者只需要直接使用 RingBuffer 的 Sequence 即可，不论谁先生产出数据，都只需要对 RingBuffer 的 Sequence 作 CAS 操作递增即可。

消费者本身需要维护 Sequence 序列，因为每个消费者的消费进度不同，自身的 Sequence 序列标识了当前消费者的消费进度。当各个消费者需要消费更多的数据时，需要通过 Sequence Barrier 进行协调。 

## 二、事件广播 Multicast Events

事件广播是 Disruptor 和队列最大的区别。当你有多个消费者监听了同一个 Disruptor，所有的事件将会被发布到所有的消费者中，相比之下队列的一个事件只能被发到一个消费者中。Disruptor 这一特性被用来需要对同一数据进行多个并行操作的情况。如在 LMAX 系统中有三个操作可以同时进行：日志（ journalling，将数据持久到日志文件中），复制（ replication，将数据发送到其他的机器上，以确保存在数据远程副本），业务逻辑处理（business logic，实际的处理工作）。也可以使用 WokrerPool 来并行处理不同的事件 。

再看上图 Figure 1，有三个事件处理器（JournalConsumer, ReplicationConsumer 和 ApplicationConsumer）监听 Disruptor，每一个都将接受 Disruptor 中所有的可用消息，这允许三个消费者并行的工作。 

### 2.1 消费者依赖图 Consumer Dependency Graph

为了支持实际业务中并行的处理流程，Disruptor 提供了多个消费者之间的协助功能。回到上面的 LMAX 的例子，我们可以让日志处理和远程副本赋值先执行完之后再执行业务处理流程，这个功能被称之为 `gating`。

gating 发生在两种场景中。第一，我们需要确保生产者不要超过消费者。通过调用 RingBuffer.addGatingConsumers() 增加相关的消费者至Disruptor来完成。第二，就是之前所说的场景，通过构造包含需要必须先完成的消费者的 Sequence 的 SequenceBarrier 来实现。 

引用上图 Figure 1， 有三个消费者监听来自 RingBuffer 的事件。这里有一个依赖关系图。ApplicationConsumer 依赖 JournalConsumer 和 ReplicationConsumer。这个意味着 JournalConsumer 和 ReplicationConsumer 可以自由的并发运行。依赖关系可以看成是从 ApplicationConsumer 的 SequenceBarrier 到 JournalConsumer 和 ReplicationConsumer 的 Sequence 的连接。

还有一点值得关注，Sequencer 与下游的消费者之间的关系。它的作用之一是确保发布（publication）不会包裹（wrap） RingBuffer。为此，所有下游消费者的Sequence 不能比 RingBuffer 的 Sequence 小且不能小于 RingBuffer 的 size。因为 ApplicationConsumers 的 Sequence 确保小于等于 JournalConsumer 和 ReplicationConsumer 的 Sequence，因此 Sequencer 只需要检查 ApplicationConsumers 的 Sequence。从更一般的意义上讲，Sequencer 只需了解依赖（dependency）树中叶节点的消费者的 Sequence 即可。 

## 三、事件预分配 Event Preallocation

Disruptor 的目标之一是被用在低延迟的环境中，在低延迟系统中有必要减少和降低内存的占用。在基于 Java 的系统中，需要减少由于 GC 导致的停顿次数（在低延迟的 C/C++ 系统中，由于内存分配器的争用，大量的内存分配也会导致问题）。 

为了满足这点，用户可以在 Disruptor 中为事件预分配内存。 在初始构造期间，EventFactory 由用户提供，并将在 Disruptor 的 RingBuffer 中为每个条目（ entry）调用其 newInstance() 方法。 当将新的数据发布到 Disruptor 中时， API 允许用户获取已经被构造的对象，以便可以调用方法更新该对象的域。Disruptor 将确保这些操作是线程安全。 

## 四、可选择的无锁 Optionally Lock-free

对于低延迟的需求推动的另一个关键实现细节是广泛使用无锁算法来实现 Disruptor。所有的内存可见性和正确性都使用内存屏障和 CAS 操作实现。除了 BlockingWaitStrategy 中使用到了锁以外，而这仅仅是为了使用 Condition，以便消费者线程在等待一个新的事件到来的时候能被 park 住。

许多低延迟系统都使用自旋（busy-wait）来避免使用 Condition 造成的抖动，但是自旋（busy-wait）的数量变多时将会导致性能的下降，特别是 CPU 资源严重受限的情况下。例如，在虚拟环境中的 Web 服务器。 
