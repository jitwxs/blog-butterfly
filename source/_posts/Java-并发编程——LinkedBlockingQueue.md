---
title: Java 并发编程——LinkedBlockingQueue
tags:
  - 队列
  - LinkedBlockingQueue
categories:
  - Java
  - 并发编程
abbrlink: e00e15f2
date: 2019-11-22 22:48:28
copyright_author: benjaminwhx
---

在集合框架里，ArrayList 和 LinkedList 是使用最多的两种集合。ArrayList 和 ArrayBlockingQueue 一样，内部基于数组来存放元素，而 LinkedBlockingQueue 则和 LinkedList 一样，内部基于链表来存放元素。

`LinkedBlockingQueue` 实现了 BlockingQueue 接口，不同于 ArrayBlockingQueue，它如果不指定容量，容量默认为 **Integer.MAX_VALUE**，也就是**无界队列**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191124224737766.png)

## 一、源码分析

### 1.1 属性

```java
/**
 * 节点类，用于存储数据
 */
static class Node<E> {
    E item;
    Node<E> next;

    Node(E x) { item = x; }
}

/** 阻塞队列的大小，默认为Integer.MAX_VALUE */
private final int capacity;

/** 当前阻塞队列中的元素个数 */
private final AtomicInteger count = new AtomicInteger();

/**
 * 阻塞队列的头结点
 */
transient Node<E> head;

/**
 * 阻塞队列的尾节点
 */
private transient Node<E> last;

/** 获取并移除元素时使用的锁，如 take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** notEmpty条件对象，当队列没有数据时用于挂起执行删除的线程 */
private final Condition notEmpty = takeLock.newCondition();

/** 添加元素时使用的锁如 put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** notFull条件对象，当队列数据已满时用于挂起执行添加的线程 */
private final Condition notFull = putLock.newCondition();
```

从上面的属性我们知道，每个添加到 LinkedBlockingQueue 队列中的数据都将被封装成 Node 节点，添加的链表队列中，其中 `head` 和 `last` 分别指向队列的头结点和尾结点。与 ArrayBlockingQueue 不同的是，LinkedBlockingQueue 内部分别使用了 takeLock 和 putLock 对并发进行控制。也就是说，**添加和删除操作并不是互斥操作**，可以同时进行，这样也就可以大大提高吞吐量。

这里如果不指定队列的容量大小，也就是使用默认的 `Integer.MAX_VALUE`，变成无界队列。如果存在添加速度大于删除速度时候，有可能会内存溢出，这点在使用前要慎重考虑。

另外，LinkedBlockingQueue 对每一个 lock 锁都提供了一个 Condition 用来挂起和唤醒其他线程。

### 1.2 构造方法

```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}

public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

默认的构造函数和最后一个构造函数创建的队列大小都为 Integer.MAX_VALUE，只有第二个构造函数用户可以指定队列的大小。第二个构造函数最后初始化了 last 和 head 节点，让它们都指向了一个元素为 null 的节点。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191124221805626.png)

最后一个构造函数使用了 putLock 来进行加锁，这样并不是为了多线程的竞争而加锁，而是为了放入的元素能立即对其他线程可见。

### 1.3 方法

LinkedBlockingQueue 也有着和 ArrayBlockingQueue 一样的方法，我们先来看看入队列的方法。

#### 1.3.1 入队

LinkedBlockingQueue 提供了多种入队操作的实现来满足不同情况下的需求，入队操作有如下几种：

- void put(E e)
- boolean offer(E e)
- boolean offer(E e, long timeout, TimeUnit unit)

**（1）put(E e)**

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 获取锁中断
    putLock.lockInterruptibly();
    try {
        //判断队列是否已满，如果已满阻塞等待
        while (count.get() == capacity) {
            notFull.await();
        }
        // 把node放入队列中
        enqueue(node);
        c = count.getAndIncrement();
        // 再次判断队列是否有可用空间，如果有唤醒下一个线程进行添加操作
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    // 如果队列中有一条数据，唤醒消费线程进行消费
    if (c == 0)
        signalNotEmpty();
}
```

以上代码总共做了以下情况的考虑：

- 队列已满，阻塞等待。
- 队列未满，创建一个 node 节点放入队列中，如果放完以后队列还有剩余空间，继续唤醒下一个添加线程进行添加。如果放之前队列中没有元素，放完以后要唤醒消费线程进行消费。

很清晰明了是不是？我们来看看该方法中用到的几个其他方法，先来看看 enqueue(Node node) 方法：

```java
private void enqueue(Node<E> node) {
    last = last.next = node;
}
```

该方法可能有些同学看不太懂，我们用一张图来看看怎样往队列里依次放入元素 A 和元素 B。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191124222149412.png)

接下来我们看看 signalNotEmpty() ，顺带着看 signalNotFul() 方法。

```java
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}

private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

之所以这样写，因为 signal 的时候要先获取到该 signal 对应的 Condition 对象的锁才行。

**（2）offer(E e)**

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        // 队列有可用空间，放入node节点，判断放入元素后是否还有可用空间，
        // 如果有，唤醒下一个添加线程进行添加操作。
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

可以看到 offer 仅仅对 put 方法稍作改动，当队列没有可用元素的时候，不同于 put 方法的阻塞等待，offer 方法直接返回了 false。

**（3）offer(E e, long timeount, TimeUnit unit)**

```java
public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        // 等待超时时间nanos，超时时间到了返回false
        while (count.get() == capacity) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

该方法只是对 offer 方法进行了阻塞超时处理，使用了 Condition 的 awaitNanos 来进行超时等待。由于 awaitNanos 方法是可中断的，为了防止在等待过程中线程被中断，这里使用 while 循环进行等待过程中中断的处理，继续等待剩下需等待的时间。

#### 1.3.2 出队

入队列的方法说完后，我们来说说出队列的方法。LinkedBlockingQueue 提供了多种出队操作的实现来满足不同情况下的需求：

- E take()
- E poll()
- E poll(long timeout, TimeUnit unit)
- drainTo(Collection<? super E> c, int maxElements)

**（1）take()**

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        // 队列为空，阻塞等待
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        // 队列中还有元素，唤醒下一个消费线程进行消费
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 移除元素之前队列是满的，唤醒生产线程进行添加元素
    if (c == capacity)
        signalNotFull();
    return x;
}
```

take 方法看起来就是 put 方法的逆向操作，它总共做了以下情况的考虑：

- 队列为空，阻塞等待。
- 队列不为空，从队首获取并移除一个元素，如果消费后还有元素在队列中，继续唤醒下一个消费线程进行元素移除。如果放之前队列是满元素的情况，移除完后要唤醒生产线程进行添加元素。

我们来看看 dequeue 方法：

```java
private E dequeue() {
    // 获取到head节点
    Node<E> h = head;
    // 获取到head节点指向的下一个节点
    Node<E> first = h.next;
    // head节点原来指向的节点的next指向自己，等待下次gc回收
    h.next = h; // help GC
    // head节点指向新的节点
    head = first;
    // 获取到新的head节点的item值
    E x = first.item;
    // 新head节点的item值设置为null
    first.item = null;
    return x;
}
```

可能有些童鞋链表算法不是很熟悉，我们可以结合注释和图来看就清晰很多了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191124222557509.png)

其实这个写法看起来很绕，我们其实也可以这么写：

```java
private E dequeue() {
    // 获取到head节点
    Node<E> h = head;
    // 获取到head节点指向的下一个节点，也就是节点A
    Node<E> first = h.next;
    // 获取到下下个节点，也就是节点B
    Node<E> next = first.next;
    // head的next指向下下个节点，也就是图中的B节点
    h.next = next;
    // 得到节点A的值
    E x = first.item;
    first.item = null; // help GC
    first.next = first; // help GC
    return x;
}
```

**（2）poll()**

```java
public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

poll 方法去除了 take 方法中元素为空后阻塞等待这一步骤，这里也就不详细说了。同理，poll(long timeout, TimeUnit unit) 也和 offer(E e, long timeout, TimeUnit unit) 一样，利用了 Condition 的 awaitNanos 方法来进行阻塞等待直至超时。这里就不列出来说了。

**（3）drainTo()**

```java
public int drainTo(Collection<? super E> c) {
    return drainTo(c, Integer.MAX_VALUE);
}

public int drainTo(Collection<? super E> c, int maxElements) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    if (maxElements <= 0)
        return 0;
    boolean signalNotFull = false;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        int n = Math.min(maxElements, count.get());
        // count.get provides visibility to first n Nodes
        Node<E> h = head;
        int i = 0;
        try {
            while (i < n) {
                Node<E> p = h.next;
                c.add(p.item);
                p.item = null;
                h.next = h;
                h = p;
                ++i;
            }
            return n;
        } finally {
            // Restore invariants even if c.add() threw
            if (i > 0) {
                // assert h.item == null;
                head = h;
                signalNotFull = (count.getAndAdd(-i) == capacity);
            }
        }
    } finally {
        takeLock.unlock();
        if (signalNotFull)
            signalNotFull();
    }
}
```

`drainTo` 相比于其他获取方法，能够一次性从队列中获取所有可用的数据对象（还可以指定获取数据的个数）。通过该方法，可以提升获取数据效率，不需要多次分批加锁或释放锁。

#### 1.3.3 获取元素

```java
public E peek() {
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```

加锁后，获取到 head 节点的 next 节点，如果为空返回 null；如果不为空，返回 next 节点的 item 值。

#### 1.3.4 删除元素

```java
public boolean remove(Object o) {
    if (o == null) return false;
    // 两个lock全部上锁
    fullyLock();
    try {
        // 从head开始遍历元素，直到最后一个元素
        for (Node<E> trail = head, p = trail.next;
             p != null;
             trail = p, p = p.next) {
            // 如果找到相等的元素，调用unlink方法删除元素
            if (o.equals(p.item)) {
                unlink(p, trail);
                return true;
            }
        }
        return false;
    } finally {
        // 两个lock全部解锁
        fullyUnlock();
    }
}

void fullyLock() {
    putLock.lock();
    takeLock.lock();
}

void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
```

因为 remove 方法使用两个锁全部上锁，所以其他操作都需要等待它完成，而该方法需要从 head 节点遍历到尾节点，所以时间复杂度为 $O(n)$。我们来看看 unlink 方法。

```java
void unlink(Node<E> p, Node<E> trail) {
    // p的元素置为null
    p.item = null;
    // p的前一个节点的next指向p的next，也就是把p从链表中去除了
    trail.next = p.next;
    // 如果last指向p，删除p后让last指向trail
    if (last == p)
        last = trail;
    // 如果删除之前元素是满的，删除之后就有空间了，唤醒生产线程放入元素
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}
```

## 二、问题

看源码的时候，产生了以下几个疑问：

1. 为什么 dequeue 里的 h.next 不指向 null，而指向 h？
2. 为什么 unlink 里没有 p.next = null 或者 p.next = p 这样的操作？

这个疑问一直困扰着我，直到我看了迭代器的部分源码后才豁然开朗，下面放出部分迭代器的源码：

```java
private Node<E> current;
private Node<E> lastRet;
private E currentElement;

Itr() {
    fullyLock();
    try {
        current = head.next;
        if (current != null)
            currentElement = current.item;
    } finally {
        fullyUnlock();
    }
}

private Node<E> nextNode(Node<E> p) {
    for (;;) {
        // 解决了问题1
        Node<E> s = p.next;
        if (s == p)
            return head.next;
        if (s == null || s.item != null)
            return s;
        p = s;
    }
}
```

迭代器的遍历分为两步，第一步加双锁把元素放入临时变量中，第二部遍历临时变量的元素。也就是说 remove 可能和迭代元素同时进行，很有可能 remove 的时候，有线程在进行迭代操作，而如果 unlink 中改变了 p 的 next，很有可能在迭代的时候会造成错误，造成不一致问题。这个解决了问题2。

而问题1其实在 nextNode 方法中也能找到，为了正确遍历，nextNode 使用了  s == p 的判断，当下一个元素是自己本身时，返回 head 的下一个节点。

## 三、总结

LinkedBlockingQueue 是一个阻塞队列，内部由两个 ReentrantLock 来实现出入队列的线程安全，由各自的 Condition 对象的 await 和 signal 来实现等待和唤醒功能。它和 ArrayBlockingQueue 的不同点在于：

1. 队列大小有所不同，ArrayBlockingQueue 是有界的初始化必须指定大小，而 LinkedBlockingQueue 可以是有界的也可以是无界的(Integer.MAX_VALUE)，对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。
2. 数据存储容器不同，ArrayBlockingQueue 采用的是数组作为数据存储容器，而 LinkedBlockingQueue 采用的则是以 Node 节点作为连接对象的链表。
3. 由于 ArrayBlockingQueue 采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而 LinkedBlockingQueue 则会生成一个额外的 Node 对象。这可能在长时间内需要高效并发地处理大批量数据的时，对于 GC 可能存在较大影响。
4. 两者的实现队列添加或移除的锁不一样，ArrayBlockingQueue 实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个 ReenterLock 锁，而 LinkedBlockingQueue 实现的队列中的锁是分离的，其添加采用的是 putLock，移除采用的则是 takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。
