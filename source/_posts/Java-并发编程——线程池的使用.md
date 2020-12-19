---
title: Java 并发编程——线程池的使用
tags: 线程池
categories: 
  - Java
  - 并发编程
abbrlink: a916fbe8
date: 2018-09-27 19:35:33
copyright_author: 海子
---

>本文基于 JDK 1.6，在高版本 JDK 中源码有所出入。

## 一、Java中的 ThreadPoolExecutor类

`java.uitl.concurrent.ThreadPoolExecutor` 类是线程池中最核心的一个类，因此如果要透彻地了解 Java 的线程池，必须先了解这个类。下面我们来看一下 ThreadPoolExecutor 类的具体实现源码。

在 ThreadPoolExecutor 类中提供了四个构造方法：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

从上面的代码可以得知，ThreadPoolExecutor 继承了 `AbstractExecutorService` 类，并提供了四个构造器，事实上，通过观察每个构造器的源码具体实现，发现**前面三个构造器都是调用的第四个构造器进行的初始化工作**。

下面解释下一下构造器中各个参数的含义：

- `corePoolSize`：**核心池的大小**，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；

- `maximumPoolSize`：**线程池最大线程数**，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；

- `keepAliveTime`：表示**线程没有任务执行时最多保持多久时间会终止**。默认情况下，只有当线程池中的线程数大于 corePoolSize 时，keepAliveTime 才会起作用，直到线程池中的线程数不大于 corePoolSize，即当线程池中的线程数大于 corePoolSize 时，如果一个线程空闲的时间达到 keepAliveTime，则会终止，直到线程池中的线程数不超过 corePoolSize。但是如果调用了 `allowCoreThreadTimeOut(boolean)`方法，在线程池中的线程数不大于 corePoolSize时，keepAliveTime 参数也会起作用，直到线程池中的线程数为0；

- `unit`：参数 keepAliveTime 的时间单位，有7种取值，在 TimeUnit 类中有7种静态属性：

| TimeUnit的取值 | 描述 |
| :--------------------- | :---- |
| TimeUnit.DAYS         | 天   |
| TimeUnit.HOURS        | 小时 |
| TimeUnit.MINUTES      | 分钟 |
| TimeUnit.SECONDS      | 分钟 |
| TimeUnit.MILLISECONDS | 秒   |
| TimeUnit.MICROSECONDS | 微秒 |
| TimeUnit.NANOSECONDS  | 纳秒 |

- `workQueue`：**一个阻塞队列，用来存储等待执行的任务**，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：`ArrayBlockingQueue`、`LinkedBlockingQueue` 和 `SynchronousQueue`。

  ArrayBlockingQueue 和 PriorityBlockingQueue 使用较少，一般使用 `LinkedBlockingQueue` 和 `Synchronous`。**线程池的排队策略与 BlockingQueue 有关。**

- `threadFactory`：**线程工厂**，主要用来创建线程；

- `handler`：**表示当拒绝处理任务时的策略**，有以下四种取值：

| 拒绝处理任务策略 | 描述 |
| :--------------------- | :---- |
| ThreadPoolExecutor.AbortPolicy| 丢弃任务并抛出 RejectedExecutionException 异常 |
| ThreadPoolExecutor.DiscardPolicy| 也是丢弃任务，但是不抛出异常 |
| ThreadPoolExecutor.DiscardOldestPolicy      | 丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程） |
| ThreadPoolExecutor.CallerRunsPolicy| 由调用线程处理该任务 |

具体参数的配置与线程池的关系将在下一节讲述。

从上面给出的 ThreadPoolExecutor 类的代码可以知道，ThreadPoolExecutor 继承了 AbstractExecutorService，我们来看一下 `AbstractExecutorService` 的实现：

```java
public abstract class AbstractExecutorService implements ExecutorService {
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {};
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {};
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) {};
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {};
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) 
        throws InterruptedException, ExecutionException {};
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {};
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {};
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
        throws InterruptedException {};
}
```

AbstractExecutorService 是一个抽象类，它实现了 `ExecutorService` 接口，我们接着看 ExecutorService 接口的实现：

```java
public interface ExecutorService extends Executor {
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

而 ExecutorService 又是继承了 `Executor` 接口，我们看一下 Executor 接口的实现：

```java
public interface Executor {
    void execute(Runnable command);
}
```

到这里，大家应该明白了 ThreadPoolExecutor、AbstractExecutorService、ExecutorService 和 Executor 几个之间的关系了。

`Executor` 是一个顶层接口，在它里面只声明了一个方法 execute(Runnable)，返回值为 void，参数为 Runnable 类型，从字面意思可以理解，就是用来执行传进去的任务的；

然后 `ExecutorService` 接口继承了 Executor 接口，并声明了一些方法：submit、invokeAll、invokeAny 以及 shutDown 等；

抽象类 `AbstractExecutorService` 实现了 ExecutorService 接口，基本实现了 ExecutorService 中声明的所有方法；

然后 `ThreadPoolExecutor` 继承了类 AbstractExecutorService。在 ThreadPoolExecutor 类中有几个非常重要的方法：

```java
execute()
submit()
shutdown()
shutdownNow()
```

`execute()` 方法实际上是 Executor 中声明的方法，在 ThreadPoolExecutor 进行了具体的实现，这个方法是 ThreadPoolExecutor 的核心方法，**通过这个方法可以向线程池提交一个任务，交由线程池去执行**。

`submit()` 方法是在 ExecutorService 中声明的方法，在 AbstractExecutorService 就已经有了具体的实现，在 ThreadPoolExecutor 中并没有对其进行重写，**这个方法也是用来向线程池提交任务的，但是它和ex ecute() 方法不同，它能够返回任务执行的结果**，去看 submit() 方法的实现，会发现它实际上还是调用的 execute() 方法，只不过它利用了 Future 来获取任务执行结果（Future 相关内容将在下一篇讲述）。

`shutdown()` 和 `shutdownNow()` 是用来关闭线程池的。

还有很多其他的方法，比如：getQueue() 、getPoolSize() 、getActiveCount()、getCompletedTaskCount()等获取与线程池相关属性的方法，有兴趣的朋友可以自行查阅API。

## 二、深入剖析线程池实现原理

### 2.1 线程池状态

在ThreadPoolExecutor中定义了一个volatile变量，另外定义了几个static final变量表示线程池的各个状态：

```java
volatile int runState;
static final int RUNNING    = 0;
static final int SHUTDOWN   = 1;
static final int STOP       = 2;
static final int TERMINATED = 3;
```

`runState`表示**当前线程池的状态**，它是一个volatile变量用来保证线程之间的可见性；下面的几个static final变量表示runState可能的几个取值。

- 当创建线程池后，初始时，线程池处于`RUNNING`状态；

- 如果调用了shutdown()方法，则线程池处于`SHUTDOWN`状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；

- 如果调用了shutdownNow()方法，则线程池处于`STOP`状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；

- 当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为`TERMINATED`状态。

### 2.2 任务的执行

在了解将任务提交给线程池到任务执行完毕整个过程之前，我们先来看一下ThreadPoolExecutor类中其他的一些比较重要成员变量：

```java
private final BlockingQueue<Runnable> workQueue;  //任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock = new ReentrantLock();  //线程池的主要状态锁，对线程池状态（比如线程池大小、runState等）的改变都要使用这个锁
private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集
 
private volatile long  keepAliveTime;  //线程存货时间   
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数
 
private volatile int   poolSize;       //线程池中当前的线程数
 
private volatile RejectedExecutionHandler handler; //任务拒绝策略
 
private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程
 
private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数
 
private long completedTaskCount;   //用来记录已经执行完毕的任务个数
```

每个变量的作用都已经标明出来了，这里要重点解释一下 `corePoolSize`、`maximumPoolSize`、`largestPoolSize` 三个变量。

corePoolSize 在很多地方被翻译成`核心池大小`，其实我的理解这个就是线程池的大小。举个简单的例子：

>假如有一个工厂，工厂里面有10个工人，每个工人同时只能做一件任务。因此只要当10个工人中有工人是空闲的，来了任务就分配给空闲的工人做；
<br>当10个工人都有任务在做时，如果还来了任务，就把任务进行排队等待；如果说新任务数目增长的速度远远大于工人做任务的速度，那么此时工厂主管可能会想补救措施，比如重新招4个临时工人进来；然后就将任务也分配给这4个临时工人做；
<br>如果说着14个工人做任务的速度还是不够，此时工厂主管可能就要考虑不再接收新的任务或者抛弃前面的一些任务了。
<br>当这14个工人当中有人空闲时，而新任务增长的速度又比较缓慢，工厂主管可能就考虑辞掉4个临时工了，只保持原来的10个工人，毕竟请额外的工人是要花钱的。

这个例子中的 corePoolSize 就是10，而 maximumPoolSize 就是 14（10+4）。

也就是说**corePoolSize就是线程池大小，maximumPoolSize在我看来是线程池的一种补救措施，即任务量突然过大时的一种补救措施**。不过为了方便理解，在本文后面还是将corePoolSize翻译成核心池大小。

`largestPoolSize` 只是一个用来起记录作用的变量，**用来记录线程池中曾经有过的最大线程数目**，跟线程池的容量没有任何关系。

下面我们进入正题，看一下任务从提交到最终执行完毕经历了哪些过程。

在 ThreadPoolExecutor 类中，最核心的任务提交方法是 `execute()` 方法，虽然通过 submit 也可以提交任务，但是实际上 submit 方法里面最终调用的还是 execute() 方法，所以我们只需要研究 execute() 方法的实现原理即可：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runState != RUNNING || poolSize == 0)
                ensureQueuedTaskHandled(command);
        }
        else if (!addIfUnderMaximumPoolSize(command))
            reject(command); // is shutdown or saturated
    }
}
```

上面的代码可能看起来不是那么容易理解，下面我们一句一句解释：

首先，判断提交的任务 command 是否为 null，若是 null，则抛出空指针异常。接着是这句，这句要好好理解一下：

```java
if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command))
```

由于是或条件运算符，所以先计算前半部分的值，如果线程池中当前线程数不小于核心池大小，那么就会直接进入下面的 if 语句块了。

如果线程池中当前线程数小于核心池大小，则接着执行后半部分，也就是执行：

```java
addIfUnderCorePoolSize(command)
```

如果执行完 addIfUnderCorePoolSize 这个方法返回 true，整个方法就直接执行完毕了。如果返回 false，则继续执行下面的 if 语句块：

```java
if (runState == RUNNING && workQueue.offer(command))
```

如果当前线程池处于RUNNING状态，则将任务放入任务缓存队列；如果当前线程池不处于RUNNING状态或者任务放入缓存队列失败，则执行：

```java
addIfUnderMaximumPoolSize(command)
```

如果执行 addIfUnderMaximumPoolSize 方法失败，则执行 reject() 方法进行任务拒绝处理。回到前面：

```java
if (runState == RUNNING && workQueue.offer(command))
```

这句的执行，如果说当前线程池处于 RUNNING 状态且将任务放入任务缓存队列成功，则继续进行判断：

```java
if (runState != RUNNING || poolSize == 0)
```

这句判断是为了防止在将此任务添加进任务缓存队列的同时其他线程突然调用 shutdown 或者 shutdownNow 方法关闭了线程池的一种应急措施。如果是这样就执行：

```java
ensureQueuedTaskHandled(command)
```

进行应急处理，从名字可以看出是保证添加到任务缓存队列中的任务得到处理。我们接着看2个关键方法的实现：`addIfUnderCorePoolSize()` 和 `addIfUnderMaximumPoolSize()`：

```java
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < corePoolSize && runState == RUNNING)
            t = addThread(firstTask);        //创建线程去执行firstTask任务   
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

这个是 addIfUnderCorePoolSize 方法的具体实现，从名字可以看出它的意图就**是当低于核心池大小时执行的方法**。

下面看其具体实现，首先获取到锁，因为这地方涉及到线程池状态的变化，先通过if语句判断当前线程池中的线程数目是否小于核心池大小，有朋友也许会有疑问：前面在 execute() 方法中不是已经判断过了吗，只有线程池当前线程数目小于核心池大小才会执行 addIfUnderCorePoolSize 方法的，为何这地方还要继续判断？原因很简单，前面的判断过程中并没有加锁，因此可能在 execute 方法判断的时候 poolSize 小于 corePoolSize，而判断完之后，在其他线程中又向线程池提交了任务，就可能导致 poolSize 不小于 corePoolSize 了，所以需要在这个地方继续判断。然后接着判断线程池的状态是否为 RUNNING，原因也很简单，因为有可能在其他线程中调用了 shutdown 或者 shutdownNow 方法。然后就是执行：

```java
t = addThread(firstTask);
```

这个方法也非常关键，传进去的参数为提交的任务，返回值为 Thread 类型。然后接着在下面判断 t 是否为空，为空则表明创建线程失败（即 poolSize>=corePoolSize 或者 runState 不等于 RUNNING），否则调用 t.start() 方法启动线程。

我们来看一下 `addThread()` 方法的实现：

```java
private Thread addThread(Runnable firstTask) {
    Worker w = new Worker(firstTask);
    Thread t = threadFactory.newThread(w);  //创建一个线程，执行任务   
    if (t != null) {
        w.thread = t;            //将创建的线程的引用赋值为w的成员变量       
        workers.add(w);
        int nt = ++poolSize;     //当前线程数加1       
        if (nt > largestPoolSize)
            largestPoolSize = nt;
    }
    return t;
}
```

在 addThread 方法中，首先用提交的任务创建了一个 `Worker` 对象，然后调用线程工厂 threadFactory 创建了一个新的线程 t，然后将线程 t 的引用赋值给了 Worker 对象的成员变量 thread，接着通过 workers.add(w) 将 Worker 对象添加到工作集当中。

下面我们看一下 Worker 类的实现：

```java
private final class Worker implements Runnable {
    private final ReentrantLock runLock = new ReentrantLock();
    private Runnable firstTask;
    volatile long completedTasks;
    Thread thread;
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
    }
    boolean isActive() {
        return runLock.isLocked();
    }
    void interruptIfIdle() {
        final ReentrantLock runLock = this.runLock;
        if (runLock.tryLock()) {
            try {
        if (thread != Thread.currentThread())
        thread.interrupt();
            } finally {
                runLock.unlock();
            }
        }
    }
    void interruptNow() {
        thread.interrupt();
    }
 
    private void runTask(Runnable task) {
        final ReentrantLock runLock = this.runLock;
        runLock.lock();
        try {
            if (runState < STOP &&
                Thread.interrupted() &&
                runState >= STOP)
            boolean ran = false;
            /*
             * beforeExecute 方法是 ThreadPoolExecutor 类的一个方法，没有具体实现，
             * 用户可以根据自己需要重载这个方法和后面的 afterExecute 方法来进行一些统计信息，比如某个任务的执行时间等。
             */
            beforeExecute(thread, task);
            try {
                task.run();
                ran = true;
                afterExecute(task, null);
                ++completedTasks;
            } catch (RuntimeException ex) {
                if (!ran)
                    afterExecute(task, ex);
                throw ex;
            }
        } finally {
            runLock.unlock();
        }
    }
 
    public void run() {
        try {
            Runnable task = firstTask;
            firstTask = null;
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            workerDone(this);   //当任务队列中没有任务时，进行清理工作       
        }
    }
}
```

它实际上实现了 Runnable 接口，因此上面的 `Thread t = threadFactory.newThread(w);` 效果跟下面这句的效果基本一样：

```java
Thread t = new Thread(w);
```

相当于传进去了一个 Runnable 任务，在线程t中执行这个 Runnable。既然 Worker 实现了 Runnable 接口，那么自然最核心的方法便是 run() 方法了：


```java
public void run() {
    try {
        Runnable task = firstTask;
        firstTask = null;
        while (task != null || (task = getTask()) != null) {
            runTask(task);
            task = null;
        }
    } finally {
        workerDone(this);
    }
}
```

从 run 方法的实现可以看出，它首先执行的是通过构造器传进来的任务 firstTask，在调用 runTask() 执行完 firstTask 之后，在 while 循环里面不断通过 `getTask()`去取新的任务来执行，那么去哪里取呢？自然是从任务缓存队列里面去取，getTask 是 ThreadPoolExecutor 类中的方法，并不是 Worker 类中的方法，下面是 getTask 方法的实现：

```java
Runnable getTask() {
    for (;;) {
        try {
            int state = runState;
            if (state > SHUTDOWN)
                return null;
            Runnable r;
            if (state == SHUTDOWN)  // Help drain queue
                r = workQueue.poll();
            else if (poolSize > corePoolSize || allowCoreThreadTimeOut) //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
                //则通过poll取任务，若等待一定的时间取不到任务，则返回null
                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
            else
                r = workQueue.take();
            if (r != null)
                return r;
            if (workerCanExit()) {    //如果没取到任务，即r为null，则判断当前的worker是否可以退出
                if (runState >= SHUTDOWN) // Wake up others
                    interruptIdleWorkers();   //中断处于空闲状态的worker
                return null;
            }
            // Else retry
        } catch (InterruptedException ie) {
            // On interruption, re-check runState
        }
    }
}
```

在 getTas k中，先判断当前线程池状态：

- 如果 runState 大于 SHUTDOWN（即为 STOP 或者 TERMINATED），则直接返回 null。

- 如果 runState 为 SHUTDOWN 或者 RUNNING，则从任务缓存队列取任务。

- 如果当前线程池的线程数大于核心池大小 corePoolSize 或者允许为核心池中的线程设置空闲存活时间，则调用 poll(time,timeUnit) 来取任务，这个方法会等待一定的时间，如果取不到任务就返回 null。

然后判断取到的任务r是否为 null，为 null 则通过调用 workerCanExit() 方法来判断当前 worker 是否可以退出，我们看一下 `workerCanExit()` 的实现：

```java
private boolean workerCanExit() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    boolean canExit;
    //如果runState大于等于STOP，或者任务缓存队列为空了
    //或者  允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
    try {
        canExit = runState >= STOP || workQueue.isEmpty() ||
            (allowCoreThreadTimeOut &&  poolSize > Math.max(1, corePoolSize));
    } finally {
        mainLock.unlock();
    }
    return canExit;
}
```

也就是说如果线程池处于 STOP 状态、或者任务队列已为空或者允许为核心池线程设置空闲存活时间并且线程数大于1时，允许 worker 退出。

如果允许 worker 退出，则调用 `interruptIdleWorkers()` 中断处于空闲状态的 worker，我们看一下 interruptIdleWorkers() 的实现：

```java
void interruptIdleWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)  //实际上调用的是worker的interruptIfIdle()方法
            w.interruptIfIdle();
    } finally {
        mainLock.unlock();
    }
}
```

从实现可以看出，它实际上调用的是 worker 的 `interruptIfIdle()` 方法，在 worker 的 interruptIfIdle() 方法中：

```java
void interruptIfIdle() {
    final ReentrantLock runLock = this.runLock;
    /*
     * 注意这里，是调用 tryLock() 来获取锁的，因为如果当前worker正在执行任务，锁已经被获取了，是无法获取到锁的。
     * 如果成功获取了锁，说明当前worker处于空闲状态
     */
    if (runLock.tryLock()) {
        try {
            if (thread != Thread.currentThread())
                thread.interrupt();
        } finally {
            runLock.unlock();
        }
    }
}
```

这里有一个非常巧妙的设计方式，假如我们来设计线程池，可能会有一个任务分派线程，当发现有线程空闲时，就从任务缓存队列中取一个任务交给空闲线程执行。但是在这里，并没有采用这样的方式，因为这样会要额外地对任务分派线程进行管理，无形地会增加难度和复杂度，这里直接让执行完任务的线程去任务缓存队列里面取任务来执行。

我们再看 `addIfUnderMaximumPoolSize()` 方法的实现，这个方法的实现思想和 addIfUnderCorePoolSize() 方法的实现思想非常相似，唯一的区别在于 addIfUnderMaximumPoolSize 方法是**在线程池中的线程数达到了核心池大小并且往任务队列中添加任务失败的情况下执行的**：

```java
private boolean addIfUnderMaximumPoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < maximumPoolSize && runState == RUNNING)
            t = addThread(firstTask);
    } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

看到没有，其实它和 addIfUnderCorePoolSize 方法的实现基本一模一样，只是if语句判断条件中的 `poolSize < maximumPoolSize` 不同而已。

到这里，大部分朋友应该对任务提交给线程池之后到被执行的整个过程有了一个基本的了解，下面总结一下：

1. 首先，要清楚 corePoolSize 和 maximumPoolSize 的含义；

2. 其次，要知道Worker是用来起到什么作用的；

3. 要知道任务提交给线程池之后的处理策略，这里总结一下主要有4点：

（1）如果当前线程池中的线程数目小于 corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；

（2）如果当前线程池中的线程数目 >= corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务；

（3）如果当前线程池中的线程数目达到 maximumPoolSize，则会采取任务拒绝策略进行处理；

（4）如果线程池中的线程数量大于 corePoolSize 时，如果某线程空闲时间超过 keepAliveTime，线程将被终止，直至线程池中的线程数目不大于 corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过 keepAliveTime，线程也会被终止。

### 2.3 线程池中的线程初始化

默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。

在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：

| 方法 | 描述 |
| :--------------------- | :---- |
| prestartCoreThread()| 初始化一个核心线程 |
|prestartAllCoreThreads()|初始化所有核心线程 |

下面是这2个方法的实现：

```java
public boolean prestartCoreThread() {
    return addIfUnderCorePoolSize(null); //注意传进去的参数是null
}
 
public int prestartAllCoreThreads() {
    int n = 0;
    while (addIfUnderCorePoolSize(null))//注意传进去的参数是null
        ++n;
    return n;
}
```

注意上面传进去的参数是 null，根据第2小节的分析可知如果传进去的参数为 null，则最后执行线程会阻塞在 getTask() 方法中的。

```java
r = workQueue.take();
```

即等待任务队列中有任务。

### 2.4 任务缓存队列及排队策略

在前面我们多次提到了任务缓存队列，即 workQueue，它用来存放等待执行的任务。

workQueue的类型为`BlockingQueue<Runnable>`，通常可以取下面三种类型：

- **ArrayBlockingQueue**：基于数组的先进先出队列，此队列创建时必须指定大小；

- **LinkedBlockingQueue**：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为 Integer.MAX_VALUE；

- **synchronousQueue**：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

### 2.5 任务拒绝策略

当线程池的任务缓存队列已满并且线程池中的线程数目达到 maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

| 拒绝处理任务策略 | 描述 |
| :--------------------- | :---- |
| ThreadPoolExecutor.AbortPolicy| 丢弃任务并抛出RejectedExecutionException异常 |
| ThreadPoolExecutor.DiscardPolicy| 也是丢弃任务，但是不抛出异常 |
| ThreadPoolExecutor.DiscardOldestPolicy      | 丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程） |
| ThreadPoolExecutor.CallerRunsPolicy| 由调用线程处理该任务 |

### 2.6 线程池的关闭

ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是 `shutdown()` 和 `shutdownNow()`，其中：

| 方法 | 描述 |
| :--------------------- | :---- |
|shutdown()| 不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务 |
|shutdownNow()|立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务 |

### 2.7 线程池容量的动态调整

ThreadPoolExecutor提供了动态调整线程池容量大小的方法：`setCorePoolSize()` 和 `setMaximumPoolSize()`。

| 方法 | 描述 |
| :--------------------- | :---- |
|setCorePoolSize()| 设置核心池大小|
|setMaximumPoolSize()|设置线程池最大能创建的线程数目大小 |

当上述参数从小变大时，ThreadPoolExecutor 进行线程赋值，还可能立即创建新的线程来执行任务。

## 三、使用示例

前面我们讨论了关于线程池的实现原理，这一节我们来看一下它的具体使用：


```java
public class Test {
     public static void main(String[] args) {   
         ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                 new ArrayBlockingQueue<Runnable>(5));
          
         for(int i=0;i<15;i++){
             MyTask myTask = new MyTask(i);
             executor.execute(myTask);
             System.out.println("线程池中线程数目："+executor.getPoolSize()+"，队列中等待执行的任务数目："+
             executor.getQueue().size()+"，已执行完别的任务数目："+executor.getCompletedTaskCount());
         }
         executor.shutdown();
     }
}
 
 
class MyTask implements Runnable {
    private int taskNum;
     
    public MyTask(int num) {
        this.taskNum = num;
    }
     
    @Override
    public void run() {
        System.out.println("正在执行task "+taskNum);
        try {
            Thread.currentThread().sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task "+taskNum+"执行完毕");
    }
}
```

```console
正在执行task 0
线程池中线程数目：1，队列中等待执行的任务数目：0，已执行完别的任务数目：0
线程池中线程数目：2，队列中等待执行的任务数目：0，已执行完别的任务数目：0
正在执行task 1
线程池中线程数目：3，队列中等待执行的任务数目：0，已执行完别的任务数目：0
正在执行task 2
线程池中线程数目：4，队列中等待执行的任务数目：0，已执行完别的任务数目：0
正在执行task 3
线程池中线程数目：5，队列中等待执行的任务数目：0，已执行完别的任务数目：0
正在执行task 4
线程池中线程数目：5，队列中等待执行的任务数目：1，已执行完别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：2，已执行完别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：3，已执行完别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：4，已执行完别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：5，已执行完别的任务数目：0
线程池中线程数目：6，队列中等待执行的任务数目：5，已执行完别的任务数目：0
正在执行task 10
线程池中线程数目：7，队列中等待执行的任务数目：5，已执行完别的任务数目：0
正在执行task 11
线程池中线程数目：8，队列中等待执行的任务数目：5，已执行完别的任务数目：0
正在执行task 12
线程池中线程数目：9，队列中等待执行的任务数目：5，已执行完别的任务数目：0
正在执行task 13
线程池中线程数目：10，队列中等待执行的任务数目：5，已执行完别的任务数目：0
正在执行task 14
task 3执行完毕
task 0执行完毕
task 2执行完毕
task 1执行完毕
正在执行task 8
正在执行task 7
正在执行task 6
正在执行task 5
task 4执行完毕
task 10执行完毕
task 11执行完毕
task 13执行完毕
task 12执行完毕
正在执行task 9
task 14执行完毕
task 8执行完毕
task 5执行完毕
task 7执行完毕
task 6执行完毕
task 9执行完毕
```

从执行结果可以看出，当线程池中线程的数目大于 5 时，便将任务放入任务缓存队列里面，当任务缓存队列满了之后，便创建新的线程。如果上面程序中，将 for 循环中改成执行 20 个任务，就会抛出任务拒绝异常了。

不过在 javadoc 中，并不提倡我们直接使用 ThreadPoolExecutor，而是使用 Executors 类中提供的几个静态方法来创建线程池：

```java
Executors.newCachedThreadPool();        //创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE
Executors.newSingleThreadExecutor();   //创建容量为1的缓冲池
Executors.newFixedThreadPool(int);    //创建固定容量大小的缓冲池
```

下面是这三个静态方法的具体实现;

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
```

从它们的具体实现来看，它们实际上也是调用了 ThreadPoolExecutor，只不过参数都已配置好了。

- `newFixedThreadPool` 创建的线程池 corePoolSize 和 maximumPoolSize 值是相等的，它使用的 LinkedBlockingQueue；

- ` newSingleThreadExecutor` 将 corePoolSize 和 maximumPoolSize 都设置为 1，也使用的 LinkedBlockingQueue；

- `newCachedThreadPool` 将 corePoolSize 设置为 0，将 maximumPoolSize 设置 为Integer.MAX_VALUE，使用的 SynchronousQueue，也就是说来了任务就创建线程运行，当线程空闲超过60秒，就销毁线程。

实际中，如果 Executors 提供的三个静态方法能满足要求，就尽量使用它提供的三个方法，因为自己去手动配置 ThreadPoolExecutor 的参数有点麻烦，要根据实际任务的类型和数量来进行配置。

另外，如果 ThreadPoolExecutor 达不到要求，可以自己继承 ThreadPoolExecutor 类进行重写。

## 四、如何合理配置线程池的大小

本节来讨论一个比较重要的话题：如何合理配置线程池大小，仅供参考。

一般需要根据任务的类型来配置线程池大小：

- 如果是 CPU 密集型任务，就需要尽量压榨CPU，参考值可以设为 $N_{CPU}+1$

- 如果是 IO 密集型任务，参考值可以设置为 $2 * N_{CPU}$

当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。
