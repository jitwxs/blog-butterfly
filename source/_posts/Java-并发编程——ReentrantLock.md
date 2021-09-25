---
title: Java 并发编程——ReentrantLock
abbrlink: cbe421eb
date: 2019-08-07 21:18:07
categories: 
  - Java
  - 并发编程
tags: 
  - ReentrantLock
  - AQS
references:
  - name: Java并发编程之ReentrantLock详解
    url: https://blog.csdn.net/qq_38293564/article/details/80515718
    rel: nofollow noopener noreferrer
    target: _blank
  - name: fullstack-tutorial
    url: https://frank-lam.github.io/fullstack-tutorial
    rel: nofollow noopener noreferrer
    target: _blank
  - name: ReentrantLock(重入锁)功能详解和应用演示
    url: 重入锁
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、简介

`ReentrantLock` 是一个**可重入**且**独占式**的锁，相较于传统的 `Synchronized `，它增加了轮询、超时、中断等高级功能。其类图如下：

![ReentrantLock](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201908/20190806224334764.png)

ReentrantLock 是 java.util.concurrent（J.U.C）包中的锁，相比于 synchronized，它多了以下高级功能：

**1. 等待可中断**

当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。

**2. 可实现公平锁**

公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。

synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。

**3. 锁绑定多个条件**

一个 ReentrantLock 对象可以同时绑定多个 Condition 对象。

ReentrantLock 有一个内部类 `Sync`，它继承了 AbstractQueuedSynchronizer（下文简称“AQS”），抽象了锁的获取和释放操作。Sync 有两个实现类，分别是 `FairSync` 和 `NonfairSync`，分别公平锁实现和非公平锁实现。

## 一、基本使用

ReentrantLock  的使用十分简单，如下所示。通过 `lock()` 方法加锁，通过 `unlock()` 方法释放锁，为了避免死锁，释放锁应当放在 finally 块中，确保锁一定能够释放。 

```java
class X {
    private final ReentrantLock lock = new ReentrantLock();

    public void m() {
        lock.lock(); 
        try {
            doSomething...
          } finally {
            lock.unlock()
          }
    }
}
```

下面来简单实验下：

```java
public class ReentrantLockDemo {
    private Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        ReentrantLockDemo demo = new ReentrantLockDemo();
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(demo::func);
        executorService.execute(demo::func);
    }

    public void func() {
        lock.lock();
        try {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        } finally {
            lock.unlock();
        }
    }
}

// Output: 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 
```

## 三、公平锁与非公平锁

ReentrantLock 的公平锁和非公平锁是通过构造方法实现的，默认无参情况下构造的是非公平锁。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

###  3.1 NonfairSync

#### 3.1.1 Lock

首先说下非公平锁的获取锁操作。当调用 `lock()` 方法时，首先判断 `compareAndSetState(0, 1)`，该方法实际上做的事情就是对 `state` 变量做了一个 CAS 操作（利用反射实现），如果 state 值为 0，就将其修改为 1，且继续执行 setExclusiveOwnerThread() 方法，那么这个 state 是什么呢？

```java
// java.util.concurrent.locks.ReentrantLock.NonfairSync#lock
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#compareAndSetState
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#stateOffset
private static final long stateOffset;
static {
    try {
        stateOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("state"));
    } catch (Exception ex) { throw new Error(ex); }
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#state
private volatile int state;
```

一开始在类图中我们说过 Sync 继承了 `AQS`， 在 AQS 类中，有一个 volatile 变量 `state`，它代表了**ReentrantLock 的重入数**。也就是说如果 ReentrantLock  没有线程占用（即 state = 0），那就就将它占用（state 置为 1）。

接下来看之后的 `setExclusiveOwnerThread()` 方法做了什么。

```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    protected AbstractOwnableSynchronizer() { }

    private transient Thread exclusiveOwnerThread;
 
    // java.util.concurrent.locks.AbstractOwnableSynchronizer#setExclusiveOwnerThread
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
 
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

如上所示，`setExclusiveOwnerThread()` 方法位于 AQS 的父类 AbstractOwnableSynchronizer（下文简称“AOS”）中。就是将 `exclusiveOwnerThread` 变量设置为当前线程。

---

以上都是 CAS 成功的逻辑，如果 CAS 操作失败，也就是说 ReentrantLock 已经被其他线程占用了，看看 `acquire(1)` 方法逻辑。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //中断当前线程
        selfInterrupt();
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#tryAcquire
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

可以看到 `acquire()` 方法是定义在 AQS 类中的，内部调用的 `tryAcquire()` 方法发现也是一个抽象方法，需要子类去具体实现，在非公平锁中，tryAcquire() 方法的实现如下：

```java
// java.util.concurrent.locks.ReentrantLock.NonfairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

// java.util.concurrent.locks.ReentrantLock.Sync#nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取state（重入值）
    int c = getState();
    if (c == 0) { // state = 0，表示没有线程占用
        // 尝试占用，如果占用成功，通过 setExclusiveOwnerThread() 设置为当前线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) { // 如果 state != 0，且占用的线程为当前线程
        // 增加重入数，这里就体现了可重入的概念
        int nextc = c + acquires;
        if (nextc < 0) // 超出最大重入数
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 当前线程不是独占锁的线程，返回 false
    return false;
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#state
private volatile int state;

protected final int getState() { return state; }

protected final void setState(int newState) { state = newState; }
```

`nonfairTryAcquire()` 方法首先根据 state 的值判断 ReentrantLock  是否已经被占用了，如果没有线程占用，则将其占用。如果已经被占用了且当前线程就是 ReentrantLock  的占用者，就增加重入的次数。

在上文的 `acquire()` 方法中，当 `tryAcquire()` 方法执行失败，也就是获取锁失败后，继续执行后面的 `acquireQueued(addWaiter(Node.EXCLUSIVE), arg)` 语句，先来说下 `addWaiter(Node.EXCLUSIVE)`。

在 `addWaiter()` 方法中，根据当前线程加上传入的 Node 节点类型 Node.EXCLUSIVE（独占类型），构造了一个新的 Node 节点，然后取到 AQS 类的 tail 尾节点，然后尝试将新构造出来的节点加入到队列的末尾中。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#addWaiter
private Node addWaiter(Node mode) {
    // 根据当前线程+Node的mode，构造出新的 Node 节点 node
    Node node = new Node(Thread.currentThread(), mode);
    // 获取到队列的尾结点 pred
    Node pred = tail;
    if (pred != null) { // 如果 pred 不为空，则将 node 加入到 pred 后面
        // 将 node 的前驱节点设置为 pred
        node.prev = pred;
        // 尝试将将队列的尾结点设置为 node
        if (compareAndSetTail(pred, node)) {
            // 设置 pred 的后继节点为 node
            pred.next = node;
            return node;
        }
    }
    
    // 如果尾结点 pred 为空，或者其他线程更新了 tail 值，导致 CAS 操作失败，enq() 方法中自旋设置尾结点
    enq(node);
    return node;
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#enq
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#tail
private transient volatile Node tail;
```

通过 `addWaiter(Node.EXCLUSIVE)` 方法已经成功将当前线程加入到 AQS 队列末尾，然后就是调用 `acquireQueued()` 自旋判断是否处于 AQS 队列头部，如果处于头部，说明轮到当前线程执行了，就可以结束自旋返回了。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 是否能够中断执行的标识
        boolean interrupted = false;
        // 自旋判断
        for (;;) {
            // 获取当前node的前驱节点
            final Node p = node.predecessor();
            // 如果node的前驱节点以及位于队列头部，且成功占用了锁
            if (p == head && tryAcquire(arg)) {
                // 将队列新的头部设置为当前node
                setHead(node);
                p.next = null; // 将原来的头部的next节点置为空，帮助 GC
                failed = false;
                return interrupted;
            }
            /**
             * shouldParkAfterFailedAcquire：判断线程可否安全挂起
             * parkAndCheckInterrupt：挂起线程并返回当时中断标识Thread.interrupted()
             */
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer.Node#predecessor
final Node predecessor() throws NullPointerException {
    Node p = prev;
    if (p == null)
        throw new NullPointerException();
    else
        return p;
}
```

#### 3.1.2 UnLock

下面来看下重入锁的释放操作，底层调用 AQS 类的 `release()` 方法。

```java
// java.util.concurrent.locks.ReentrantLock#unlock
public void unlock() {
    sync.release(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#release
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

其中的判断条件 `tryRelease()` 实现如下，代码比较简单，看注释就应该能够明白含义了：

```java
// java.util.concurrent.locks.ReentrantLock.Sync#tryRelease
protected final boolean tryRelease(int releases) {
    // 计算剩余的 state 重入数
    int c = getState() - releases;
    // 当前线程不是锁的占用者
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();

    boolean free = false;
    if (c == 0) { // 如果重入次数已经归0了
        free = true;
        // 清除占用标识
        setExclusiveOwnerThread(null);
    }
    
    // 更新 state
    setState(c);
    return free;
}
```

如果该方法返回 true，即代表锁已经没有线程独占了，下面的处理就是一些对 AQS 同步队列的收尾工作，这里暂且不做展开。

### 3.2 FairSync

说完了非公平锁，下面来看看公平锁的实现，公平锁相较于非公平锁主要的不同就是 `lock()` 方法的逻辑：

```java
//java.util.concurrent.locks.ReentrantLock.FairSync#lock
final void lock() {
    acquire(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#tryAcquire
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

以上的逻辑都是 AQS 类的逻辑，直接看 `tryAcquire()` 方法的公平锁实现：

```java
// java.util.concurrent.locks.ReentrantLock.FairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法和非公平锁的 `nonfairTryAcquire()` 比较，唯一不同的是判断条件多了 `hasQueuedPredecessors()`方法，其定义如下：

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#hasQueuedPredecessors
public final boolean hasQueuedPredecessors() {
    // 判断AQS队列中，当前线程之前是否还有其他的等待线程。如果有：true；没有：false
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    // 头结点 != 尾结点 && 头结点的 next 节点 != 当前线程
    return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

该方法是实现“**公平**”的具体逻辑。它对 AQS 同步队列中当前节点**是否有前驱节点进行判断，如果该方法返回 true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁**，以此来实现公平锁。

### 3.3 测试

下面来分别测试下公平锁和非公平锁。

```java
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    private static CountDownLatch latch;

    public static void main(String[] args) {
        Lock fairLock = new MyReentrantLock(true);
        Lock unFairLock = new MyReentrantLock(false);

        testLock(fairLock);
    }

    private static void testLock(Lock lock) {
        latch = new CountDownLatch(1);
        for (int i = 0; i < 5; i++) {
            Thread thread = new Worker(lock, latch);
            thread.setName("Thread-" + i);
            thread.start();
        }
        latch.countDown();
    }
}

class Worker extends Thread {
    private Lock lock;
    private CountDownLatch latch;

    public Worker(Lock lock, CountDownLatch latch) {
        this.lock = lock;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int i = 0; i < 2; i++) {
            lock.lock();
            try {
                System.out.println("Lock by [" + getName() + "], Waiting by " + ((MyReentrantLock) lock).getQueuedThreads());
            } finally {
                lock.unlock();
            }
        }
    }

    @Override
    public String toString() {
        return getName();
    }
}

class MyReentrantLock extends ReentrantLock {
    MyReentrantLock(boolean fair) {
        super(fair);
    }

    @Override
    public Collection<Thread> getQueuedThreads() {
        List<Thread> arrayList = new ArrayList<>(super.getQueuedThreads());
        Collections.reverse(arrayList);
        return arrayList;
    }
}
```

当使用公平锁运行时，输出大致如下：

```console
Lock by [Thread-3], Waiting by [Thread-4]
Lock by [Thread-4], Waiting by [Thread-0, Thread-1, Thread-2, Thread-3]
Lock by [Thread-0], Waiting by [Thread-1, Thread-2, Thread-3, Thread-4]
Lock by [Thread-1], Waiting by [Thread-2, Thread-3, Thread-4, Thread-0]
Lock by [Thread-2], Waiting by [Thread-3, Thread-4, Thread-0, Thread-1]
Lock by [Thread-3], Waiting by [Thread-4, Thread-0, Thread-1, Thread-2]
Lock by [Thread-4], Waiting by [Thread-0, Thread-1, Thread-2]
Lock by [Thread-0], Waiting by [Thread-1, Thread-2]
Lock by [Thread-1], Waiting by [Thread-2]
Lock by [Thread-2], Waiting by []
```

当使用非公平锁运行时，输出大致如下：

```console
Lock by [Thread-3], Waiting by [Thread-4]
Lock by [Thread-3], Waiting by [Thread-4, Thread-1, Thread-0, Thread-2]
Lock by [Thread-4], Waiting by [Thread-1, Thread-0, Thread-2]
Lock by [Thread-4], Waiting by [Thread-1, Thread-0, Thread-2]
Lock by [Thread-1], Waiting by [Thread-0, Thread-2]
Lock by [Thread-1], Waiting by [Thread-0, Thread-2]
Lock by [Thread-0], Waiting by [Thread-2]
Lock by [Thread-0], Waiting by [Thread-2]
Lock by [Thread-2], Waiting by []
Lock by [Thread-2], Waiting by []
```

从上述结果可以看到，公平锁每次都是队列中的第一个节点获取到锁，而非公平锁出现了一个线程连续获取锁的情况。

为什么会出现连续获取锁的情况呢？因为在 `nonfairTryAcquire(int)` 方法中，每当一个线程请求锁时，只要获取了同步状态就成功获取了锁。在此前提下，刚刚释放锁的线程再次获取到同步状态的几率很大，而其他线程只能在同步队列中等待。

### 3.4 总结

事实上，公平锁往往没有非公平锁的效率高，但是，并不是任何场景都是以 TPS 作为唯一指标，公平锁能够减少“饥饿”发生的概率，等待越久的请求越能够得到优先满足。

非公平锁有可能使线程饥饿，那为什么还要将它设置为默认模式呢？我们再次观察上面的运行结果，如果把每次不同线程获取到锁定义为1次切换，公平锁在测试中进行了10次切换，而非公平锁只有5次切换，这说明非公平锁的开销更小。

## 四、lockInterruptibly & tryLock

### 4.1 lockInterruptibly 

一开始就说过 ReentrantLock  支持等待可中断。在使用 synchronized 时，阻塞在锁上的线程除非获得锁否则将一直等待下去，也就是说这种无限等待获取锁的行为无法被中断。而 ReentrantLock 给我们提供了一个可以响应中断的获取锁的方法 `lockInterruptibly()`。

```java
// java.util.concurrent.locks.ReentrantLock#lockInterruptibly
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireInterruptibly
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        // 没有得到独占锁后
        doAcquireInterruptibly(arg);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireInterruptibly
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    //将该结点尾插到 AQS 同步队列
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            //获取前置节点
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                // 这里没有中断标识,lock和lockInterruptibly区别就是对中断的处理方式 
                return;
            }
            
            //不断自旋直至将前驱结点状态设置为SIGNAL，然后阻塞当前线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                // 无中断标识,直接抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在前面介绍 `lock()` 方法时，其中的 `acquireQueued()` 方法在加锁失败后会设置一个中断标识 interrupted，死循环休眠加锁。而 `doAcquireInterruptibly()` 方法相较于 `acquireQueued()` 方法取消了中断标识，直接返回来实现响应中断。

### 4.2 tryLock

获取锁除了使用 `lock()` 和 `lockInterruptibly ()` 这类阻塞方法以外，ReentrantLock  还提供了非阻塞加锁方法，也就是 `tryLock()`。

1. **tryLock()**

   立即返回，获取成功返回 true，获取失败返回 false。

2. **tryLock(long timeout, TimeUnit unit)**

   在给定时间内，获取成功返回 true，获取失败返回 false。

## 五、对比 Synchronized 

### 5.1 相同点

**1. 都是独占锁**

ReentrantLock 和 synchronized 都是独占锁，只允许线程互斥的访问临界区。但是实现上两者不同，synchronized 加锁解锁的过程是隐式的，用户不用手动操作，优点是操作简单，但显得不够灵活，ReentrantLock 需要手动加锁和解锁。

**2. 都是可重入**

ReentrantLock 和 synchronized 都是可重入的。synchronized 因为可重入因此可以放在被递归执行的方法上，且不用担心线程最后能否正确释放锁。而 ReentrantLock 在重入时要却确保重复获取锁的次数必须和重复释放锁的次数一样，否则可能导致其他线程无法获得该锁。

### 5.2 不同点

**1. 锁的实现**

synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。

**2. 性能**

新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等。目前来看它和 ReentrantLock 的性能基本持平了，因此性能因素不再是选择 ReentrantLock 的理由。synchronized 有更大的性能优化空间，应该优先考虑 synchronized。

**3. 功能**

ReentrantLock 多了一些高级功能。

**4. 使用选择**

除非需要使用 ReentrantLock 的高级功能，否则优先使用 synchronized。这是因为 synchronized 是 JVM 实现的一种锁机制，JVM 原生地支持它，而 ReentrantLock 不是所有的 JDK 版本都支持。并且使用 synchronized 不用担心没有释放锁而导致死锁问题，因为 JVM 会确保锁的释放。
