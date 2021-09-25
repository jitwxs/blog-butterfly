---
title: Java-并发编程——CyclicBarrier
tags: CyclicBarrier
categories:
  - Java
  - 并发编程
abbrlink: 6ff6c3e2
date: 2019-11-24 23:34:29
references:
  - name: 【细谈Java并发】谈谈CyclicBarrier
    url: http://benjaminwhx.com/2018/05/03/【细谈Java并发】谈谈CyclicBarrier/
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Java并发编程--CyclicBarrier
    url: https://www.cnblogs.com/zaizhoumo/p/7787064.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Java并发之CyclicBarrier-栅栏详解
    url: https://blog.csdn.net/weixin_38003389/article/details/85117646
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、简介

`CyclicBarrier` 是一个同步工具类，它允许一组线程在到达某个栅栏点(common barrier point)互相等待，发生阻塞，直到最后一个线程到达栅栏点，栅栏才会打开，处于阻塞状态的线程恢复继续执行.它非常适用于一组线程之间必需经常互相等待的情况。CyclicBarrier 字面理解是循环的栅栏，之所以称之为循环的是因为在等待线程释放后，该栅栏还可以复用。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201911/2019112423333697.png)

建议阅读 CyclicBarrier 源码前，先深入研究一下 ReentrantLock 的原理，搞清楚 Condition 里 await 和 signal 的原理。

## 二、应用场景

使用 CyclicBarrier 来模拟一下对战平台中玩家需要完全准备好了才能进入游戏的场景，例如 LOL 的加载过程。

```java
public class CyclicBarrierTest {

    private final static ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(5);
    private final static CyclicBarrier BARRIER = new CyclicBarrier(5);

    public static void main(String[] args) {
        for (int i = 1; i <= 5; i++) {
            final String name = "玩家" + i;
            EXECUTOR_SERVICE.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(2000);
                        System.out.println(name + "已准备,等待其他玩家准备...");
                        BARRIER.await();
                        Thread.sleep(1000);
                        System.out.println(name + "已加入游戏");
                    } catch (InterruptedException | BrokenBarrierException e) {
                        System.out.println(name + "离开游戏");
                    }
                }
            });
        }
        EXECUTOR_SERVICE.shutdown();
    }
}
```

输出结果：

```console
玩家2已准备,等待其他玩家准备...
玩家4已准备,等待其他玩家准备...
玩家3已准备,等待其他玩家准备...
玩家1已准备,等待其他玩家准备...
玩家5已准备,等待其他玩家准备...
玩家4已加入游戏
玩家1已加入游戏
玩家2已加入游戏
玩家3已加入游戏
玩家5已加入游戏
```

## 三、源码分析

CyclicBarrier 基于 ReentrantLock 和 Condition 机制实现。除了 `getParties()` 方法，CyclicBarrier 的其他方法都需要获取锁。

### 3.1 属性

```java
public class CyclicBarrier {
    private static class Generation {
        boolean broken = false;
    }
    // 锁
    private final ReentrantLock lock = new ReentrantLock();
    // 通过lock得到的一个状态变量，用来await和signal
    private final Condition trip = lock.newCondition();
    // 通过构造器传入的参数，表示总的等待线程的数量
    private final int parties;
    // 当屏障正常打开后运行的程序，通过最后一个调用await的线程来执行
    private final Runnable barrierCommand;
    // 当前的Generation。每当屏障失效或者开闸之后都会自动替换掉。从而实现重置的功能
    private Generation generation = new Generation();
    // 和parties一样，每次线程await后减1
    private int count;
    
    ...
}
```

### 3.2 构造方法

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}
```

CyclicBarrier 提供了以上两个构造方法调用：

1. 默认的构造方法是 CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用 await 方法告诉 CyclicBarrier 已经到达屏障位置，线程被阻塞。
2. 另外一个构造方法 CyclicBarrier(int parties, Runnable barrierAction)，其中 barrierAction 任务会在所有线程到达屏障后执行，方便处理更复杂的业务场景。

### 3.3 await()

调用 await() 方法的线程告诉 CyclicBarrier 已经到达同步点，然后当前线程被阻塞。直到 parties（设置的屏障数量）个参与线程调用了await方法。CyclicBarrier 同样提供带超时时间的 await() 方法。

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
        BrokenBarrierException,
        TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
```

await() 和 await(long, TimeUnit) 都是调用 dowait() 方法，这个方法是核心方法，在等待线程数未达到 parties 时静静地等待，满足一下条件就会中断等待：

- 最后一个线程到达，即 index == 0
- 某个参与线程等待超时
- 某个参与线程被中断
- 调用了 CyclicBarrier 的 reset() 方法，该方法会将屏障重置为初始状态

```java
private int dowait(boolean timed, long nanos) throws InterruptedException,  
			BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 将局部变量设置为当前代（generation）
        final Generation g = generation;

        // 如果当前代处于broken状态，则抛出 BrokenBarrierExcption
        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            // 如果当前线程被中断，则将当前代置于broken状态，并重置剩余 count。
            // 唤醒状态变量，这时候其他线程收到broken状态后，也会抛出 BrokenBarrierException。
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;    // 降低当前count
        /**
         * 如果当前状态将为0，说明最后一个线程调用了该方法。
         * 则当前代处于开闸状态，运行可能存在的 command。然后更新代。
         */
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                // 更新代，将count重置，唤醒之前等待的线程。
                nextGeneration();
                return 0;
            } finally {
                 // 如果运行command失败也会导致当前屏障被broken。
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed) // 如果没有时间限制，则直接等待，直到被唤醒
                    trip.await();
                else if (nanos > 0L) // 如果有时间限制，则等待指定时间
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 如果没有换代，且代没有损坏，则将当前代置于broken状态
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // 否则当前线程不处于最新代上，中断线程。
                    Thread.currentThread().interrupt();
                }
            }

            // 当前线程从阻塞中恢复后，如果收到其他线程的中断信号，则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();

            /*
             * 如果已经换代，直接返回 index (最后一个线程已经执行的 nextGeneration，但当前线程还没有执行到该语句)
             * 如果 g == generation，说明还没有换代，那为什么会醒了？
             * 因为一个线程可以使用多个栅栏，当别的栅栏唤醒了这个线程，就会走到这里，所以需要判断是否是当前代。
             * 正是因为这个原因，才需要generation来保证正确。
             */
            if (g != generation)
                return index;

            // 如果有时间限制，且时间小于等于0，销毁栅栏并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

列下上面调用的两个方法：

- `nextGeneration()` 在屏障开闸之后重置状态，以待下一次调用。
- `breakBarrier()` 在屏障打破之后设定打破状态，以唤醒其他线程并通知。

```java
private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}

private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

###  3.4 reset()

reset() 方法将屏障重置为其初始状态，如果所有参与者目前都在屏障处等待，则它们将返回，同时抛出一个BrokenBarrierException。

这里要注意一下要先 broken 当前屏蔽，然后再创建一个新的屏蔽，否则的话可能会导致信号丢失。

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

## 四、CountDownLatch 的区别

1. CountDownLatch 一个线程(或者多个)，等待另外 N 个线程完成某个事情之后才能执行；CyclicBarrier N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。
2. CountDownLatch 是一次性的；CyclicBarrier 可以重复使用。
3. CountDownLatch 基于 AQS 技术；CyclicBarrier 基于锁和 Condition。本质上都是依赖于 volatile 和 CAS 实现的。
