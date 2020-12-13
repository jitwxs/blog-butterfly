---
title: Java 并发编程——CountdownLatch
typora-root-url: ..
abbrlink: c0b61611
date: 2019-08-11 23:10:35
categories: 
  - Java
  - 并发编程
tags: 
  - CountdownLatch
  - AQS
references:
  - name: 《CountDownLatch详解》
    url: https://www.jianshu.com/p/128476015902
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、前言

今天来介绍下 concurrent 包下的一个工具类——`CountDownLatch`，这算是一个比较实用的工具类，在我们日常开发中使用的比较多，而且 API 也很简单，总结记录下。

## 二、基本使用

`CountdownLatch` 的主要功能**是允许一个或多个线程等待直到在其他线程中一组操作执行完成**，用人话说就是多个线程分别执行任务，另外某个线程等待这些线程全部执行完毕后，再做其他操作。

举个例子，麻麻在炒菜的时候，总是喜欢先将炒菜材料都准备好后，再准备开锅炒菜。由于今天家里来了客人，要准备的炒菜材料太多了，妈妈让你和你的弟弟一起来准备炒菜材料，麻麻等你们都准备好后开始炒菜。

在这个例子中，你和弟弟就分别是一个线程，你们做的工作就是准备炒菜材料。麻麻是另外一个线程，她会先休息会，等待你们都准备好后继续执行炒菜任务。

CountdownLatch 的主要 API 只有三个，如下所示：

```java
public CountDownLatch(int count);

public void countDown();

public void await() throws InterruptedException;
```

第一个是构造方法，参数指定了麻麻要等待的次数。假设要等待你和弟弟一人一次，那么值就是2。

第二个 `countDown()` 方法是当你或者弟弟执行完毕后，调用一次，麻麻要等待的次数就减1了。

第三个 `await()` 方法是对于麻麻用的，当麻麻执行该方法时，如果等待的次数不为0，则一直等待。如果为0，表示你和弟弟都准备完毕了，可以开始炒菜了。

```java
public class CountdownLatchDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        CountDownLatch latch = new CountDownLatch(2);

        executor.execute(new TaskThread("你", latch));
        executor.execute(new TaskThread("弟弟", latch));

        try {
            // 麻麻等待你们
            latch.await();
            // 开始炒菜
            System.out.println("麻麻开始炒菜了...");

            executor.shutdown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class TaskThread implements Runnable {
        private String name;
        private CountDownLatch latch;

        TaskThread(String name, CountDownLatch latch) {
            this.name = name;
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                System.out.println(name + "开始干活了...");
                TimeUnit.SECONDS.sleep(2); // 模拟干活
                System.out.println(name + "活干了...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                // 通知麻麻
                latch.countDown();
            }
        }
    }
}
```

程序运行结果如下：

```console
你开始干活了...
弟弟开始干活了...
你活干了...
弟弟活干了...
麻麻开始炒菜了...
```

下面有一些需要注意的：

1. `countdown()` 方法并没有规定一个线程只能调用一次，每调用一次计数器就会减一。
2. `await()` 方法并没有规定只能一个线程调用，如果有多个线程调用，那么这几个线程都会阻塞直到计数器为0。

## 三、源码浅析

`CountDownLatch`  的底层实现其实和 ReentrantLock 是一样的，都是 AQS。对这两个有兴趣的，可以移步[《Java 并发编程——ReentrantLock》](/cbe421eb.html)。

下面是 CountDownLatch 的类图，可以看到它有一个内部类 Sync，Sync 的父类也就是 AQS。

![CountDownLatch](/images/posts/20190811222801389.png)

### 3.1 constructors

构造方法很简单，实参 count 最后设置到了 AQS 的 state 变量上，注意该变量是多线程可共享的。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

// java.util.concurrent.CountDownLatch.Sync#Sync
Sync(int count) {
    setState(count);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#setState
protected final void setState(int newState) {
    state = newState;
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#state
private volatile int state;
```

### 3.2 countdown

countdown() 方法调用的底层实际上是 AQS 的 `releaseShared()` 方法，releaseShared() 方法代码如下：

```java
// java.util.concurrent.CountDownLatch#countDown    
public void countDown() {
    sync.releaseShared(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#releaseShared
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

releaseShared() 方法中判断 tryReleaseShared() 是否为 true，为 true 执行 `doReleaseShared()` 方法。先看下 releaseShared() 代码：

```java
// java.util.concurrent.CountDownLatch.Sync#tryReleaseShared        
protected boolean tryReleaseShared(int releases) {
    // 死循环判断
    for (;;) {
        int c = getState(); // 获取 state 值
        if (c == 0)
            return false; // 如果 state 为 0，表示计数结束。
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) // CAS 设置 state 值
            return nextc == 0; // 设置成功后，判断 state 是否为 0
    }
}
```

ReleaseShared 的意思是释放计数器，因此 `tryReleaseShared()` 中只有 state 减1后为0时才需要调用 `doReleaseShared()` 方法，看下该方法逻辑：

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#doReleaseShared
private void doReleaseShared() {
    for (;;) {
        Node h = head; // 记录等待队列中的头结点的线程
        if (h != null && h != tail) { // 头结点不为空，且头结点不等于尾节点
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // SIGNAL状态表示当前节点正在等待被唤醒
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 清除当前节点的等待状态
                    continue;
                unparkSuccessor(h); // 唤醒当前节点的下一个节点
            } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head) // 如果h还是指向头结点，说明前面这段代码执行过程中没有其他线程对头结点进行过处理
            break;
    }
}
```

在 `doReleaseShared()` 方法中，就是对 AQS 的同步队列中其它线程的唤醒操作。首先判断头结点不为空且不为尾节点，说明等待队列中有等待唤醒的线程，这里需要说明的是，在等待队列中，头节点中并没有保存正在等待的线程，其只是一个空的 Node 对象，真正等待的线程是从头节点的下一个节点开始存放的，因而会有对头结点是否等于尾节点的判断。

在判断等待队列中有正在等待的线程之后，其会清除头结点的状态信息，并且调用 `unparkSuccessor(Node)` 方法唤醒头结点的下一个节点。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); // 清除当前节点的等待状态
    
    Node s = node.next;
    if (s == null || s.waitStatus > 0) { // s 的等待状态大于0说明该节点中的线程已经被外部取消等待了
        s = null;
         // 从队列尾部往前遍历，找到最后一个处于等待状态的节点，用s记录下来
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒离传入节点最近的处于等待状态的节点线程
}
```

可以看到，`unparkSuccessor(Node)` 方法的作用是唤醒离传入节点最近的一个处于等待状态的线程，使其继续往下执行。

前面我们讲到过，等待队列中的线程可能有多个，而调用 `countDown()` 方法的线程只唤醒了一个处于等待状态的线程，这里剩下的等待线程是如何被唤醒的呢？其实这些线程是被当前唤醒的线程唤醒的。具体的我们可以看看 `await()` 方法的具体执行过程。

### 3.3 await

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireSharedInterruptibly
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// java.util.concurrent.CountDownLatch.Sync#tryAcquireShared
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

可以看到 await() 实际调用了 `acquireSharedInterruptibly()` 方法，作用是**判断当前线程是否需要以共享状态获取执行权限**。首先判断 state 值是否为0。如果为0，表示当前线程不需要进行权限获取，返回-1则表示当前线程需要进行共享权限，具体看 `doAcquireSharedInterruptibly()` 方法：

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireSharedInterruptibly   
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.SHARED); // 使用当前线程创建一个共享模式的Node节点
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor(); // 获取当前节点的前一个节点
            if (p == head) { // 判断前一个节点是否为头结点
                int r = tryAcquireShared(arg); // 查看当前线程是否获取到了执行权限
                if (r >= 0) { // 由于tryAcquireShared()返回0或-1，因此实际是为0表示获取了执行权限
                    setHeadAndPropagate(node, r);  // 将当前节点设置为头结点，并且唤醒后面处于等待状态的节点
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            
             // 走到这一步说明没有获取到执行权限，就使当前线程进入“搁置”状态
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在 `doAcquireSharedInterruptibly()` 方法中，首先使用当前线程创建一个共享模式的节点。然后死循环判断当前线程是否获取到执行权限，如果有则将当前节点设置为头节点，并且唤醒后续处于共享模式的节点；如果没有，则调用`shouldParkAfterFailedAcquire()` 和 `parkAndCheckInterrupt()` 方法**使当前线程处于阻塞状态**，该阻塞状态是由操作系统进行的，这样可以避免该线程无限循环而获取不到执行权限，造成资源浪费。

当有多个线程调用 await() 方法而进入等待状态时，这几个线程都将等待在此处。这里回过头来看前面将的 countDown() 方法，其会唤醒处于等待队列中离头节点最近的一个处于等待状态的线程，也就是说该线程被唤醒之后会继续从这个位置开始往下执行，此时执行到 tryAcquireShared() 方法时，发现大于0（因为 state 已经被置为 0 了），该线程就会调用 setHeadAndPropagate() 方法，并且退出当前循环，也就开始执行 awat() 方法之后的代码。

下面我们看看 `setHeadAndPropagate()` 方法的具体实现：

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#setHeadAndPropagate
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node); // 将当前节点设置为头节点
    
    // 检查唤醒过程是否需要往下传递，并且检查头结点的等待状态
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared()) // 如果下一个节点是尝试以共享状态获取获取执行权限的节点，则将其唤醒
            doReleaseShared();
    }
}
```

setHeadAndPropagate() 方法主要作用是**设置当前节点为头结点，并且将唤醒工作往下传递**，在传递的过程中，其会判断被传递的节点是否是以共享模式尝试获取执行权限的，如果不是，则传递到该节点处为止（一般情况下，等待队列中都只会都是处于共享模式或者处于独占模式的节点）。

也就是说，**头结点会依次唤醒后续处于共享状态的节点**。这里doReleaseShared()方法也就是我们前面讲到的会将离头结点最近的一个处于等待状态的节点唤醒的方法。

## 四、超时阻塞

在上面介绍 `await()` 时，只有当计数器为 0 时，或者线程被中断后，才会结束阻塞。但是万一执行任务的线程出现了死锁或者死循环，那么岂不是一直阻塞在那，永远都无法继续下去了？

因此同时提供了一个带有超时时间的 await() 方法，当到达超时时间后，立即返回。根据返回值来决定是否执行完毕。

```java
public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

在实际使用过程中，CountdownLatch 有可能被应用于多个微服务之间的数据获取，一旦接口没有做好超时处理，那么 CountDownLatch 就有可能长时间甚至一直阻塞，因此使用带超时的 await() 方法是十分必要的。

## 五、使用实例

CountDownLatch 非常适合对于任务进行拆分，使子任务能够并行执行，这也是我目前实际开发使用中的一个主要应用点。后续我接触到了其它应用场景，会继续分享给大家~

下面是一个任务拆分模拟实现，将 List 中 0 ~ 100 这些数据进行分批处理，每个线程处理 10 个，等到全部处理完毕后，打印一下 CountDownLatch complete!

```java
public class CountdownLatchDemo {
    private ExecutorService executor = Executors.newFixedThreadPool(5);

    public static void main(String[] args) {
        List<Integer> dataList = IntStream.range(1, 100).boxed().collect(Collectors.toList());
        int size = 10;

        new CountdownLatchDemo1().func(dataList, size);
    }

    /**
     * @param dataList 待处理数据列表
     * @param segment  单个线程处理数据量
     */
    private void func(List<Integer> dataList, int segment) {
        int threadCount = dataList.size() % segment == 0 ? dataList.size() / segment : dataList.size() / segment + 1;
        CountDownLatch latch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            List<Integer> subList;
            if (i == threadCount - 1) {
                subList = dataList.subList(i * segment, dataList.size());
            } else {
                subList = dataList.subList(i * segment, (i + 1) * segment);
            }
            executor.execute(new TaskThread(subList, latch));
        }

        try {
            latch.await();
            System.out.println("CountDownLatch complete!");
            executor.shutdown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public class TaskThread implements Runnable {
        private List<Integer> list;
        private CountDownLatch latch;

        TaskThread(List<Integer> list, CountDownLatch latch) {
            this.list = list;
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                System.out.println(String.format("Thread-%s: %s", Thread.currentThread().getName(), list));
            } finally {
                latch.countDown();
            }
        }
    }
}
```

运行结果如下：

```console
Thread-pool-1-thread-1: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
Thread-pool-1-thread-4: [31, 32, 33, 34, 35, 36, 37, 38, 39, 40]
Thread-pool-1-thread-2: [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
Thread-pool-1-thread-3: [21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
Thread-pool-1-thread-1: [51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
Thread-pool-1-thread-5: [41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
Thread-pool-1-thread-4: [61, 62, 63, 64, 65, 66, 67, 68, 69, 70]
Thread-pool-1-thread-3: [81, 82, 83, 84, 85, 86, 87, 88, 89, 90]
Thread-pool-1-thread-1: [91, 92, 93, 94, 95, 96, 97, 98, 99]
Thread-pool-1-thread-2: [71, 72, 73, 74, 75, 76, 77, 78, 79, 80]
CountDownLatch complete!
```
