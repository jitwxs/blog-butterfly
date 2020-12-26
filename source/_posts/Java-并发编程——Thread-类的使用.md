---
title: Java 并发编程——Thread 类的使用
tags: Thread
categories: 
  - Java
  - 并发编程
abbrlink: 29b9a7ac
date: 2018-10-02 11:59:46
copyright_author: 海子
---

在学习Thread类之前，先介绍与线程相关知识：线程的几种状态、上下文切换，然后接着介绍Thread类中的方法的具体使用。

## 一、线程的状态

线程从创建到最终的消亡，要经历若干个状态。一般来说，线程包括以下这几个状态：**创建(new)**、**就绪(runnable)**、**运行(running)**、**阻塞(blocked)**、**time waiting**、**waiting**、**消亡（dead）**。

当需要新起一个线程来执行某个子任务时，就创建了一个线程。但是线程创建之后，不会立即进入就绪状态，因为线程的运行需要一些条件（比如内存资源，程序计数器、Java栈、本地方法栈都是线程私有的，所以需要为线程分配一定的内存空间），只有线程运行需要的所有条件满足了，才进入就绪状态。

当线程进入就绪状态后，不代表立刻就能获取CPU执行时间，也许此时CPU正在执行其他的事情，因此它要等待。当得到CPU执行时间之后，线程便真正进入运行状态。

线程在运行状态过程中，可能有多个原因导致当前线程不继续运行下去，比如用户主动让线程睡眠（睡眠一定的时间之后再重新执行）、用户主动让线程等待，或者被同步块给阻塞，此时就对应着多个状态：time waiting（睡眠或等待一定的事件）、waiting（等待被唤醒）、blocked（阻塞）。

当由于突然中断或者子任务执行完毕，线程就会被消亡。

下面这副图描述了线程从创建到消亡之间的状态：

![线程的状态](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002112343855.png)

在有些教程上将blocked、waiting、time waiting统称为阻塞状态，这个也是可以的，只不过这里我想将线程的状态和Java中的方法调用联系起来，所以将waiting和time waiting两个状态分离出来。

## 二、上下文切换

对于单核CPU来说（对于多核CPU，此处就理解为一个核），CPU在一个时刻只能运行一个线程，当在运行一个线程的过程中转去运行另外一个线程，这个叫做`线程上下文切换`（对于进程也是类似）。

由于可能当前线程的任务并没有执行完毕，所以在切换时需要保存线程的运行状态，以便下次重新切换回来时能够继续切换之前的状态运行。举个简单的例子：比如一个线程A正在读取一个文件的内容，正读到文件的一半，此时需要暂停线程A，转去执行线程B，当再次切换回来执行线程A的时候，我们不希望线程A又从文件的开头来读取。

因此需要记录线程A的运行状态，那么会记录哪些数据呢？因为下次恢复时需要知道在这之前当前线程已经执行到哪条指令了，所以需要记录程序计数器的值，另外比如说线程正在进行某个计算的时候被挂起了，那么下次继续执行的时候需要知道之前挂起时变量的值时多少，因此需要记录CPU寄存器的状态。**所以一般来说，线程上下文切换过程中会记录程序计数器、CPU寄存器状态等数据**。

说简单点的：对于线程的上下文切换实际上就是 **存储和恢复CPU状态的过程，它使得线程执行能够从中断点恢复执行**。

虽然多线程可以使得任务执行的效率得到提升，但是由于在线程切换时同样会带来一定的开销代价，并且多个线程会导致系统资源占用的增加，所以在进行多线程编程时要注意这些因素。

## 三、Thread类中的方法

通过查看java.lang.Thread类的源码可知：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002112727866.png)

Thread类实现了Runnable接口，在Thread类中，有一些比较关键的属性，比如`name`是表示Thread的名字，可以通过Thread类的构造器中的参数来指定线程名字，`priority`表示线程的优先级（最大值为10，最小值为1，默认值为5），`daemon`表示线程是否是守护线程，`target`表示要执行的任务。

以下是Thread类中的一些方法：

### 3.1 start

`start()`用来启动一个线程，**当调用start方法后，系统才会开启一个新的线程来执行用户定义的子任务**，在这个过程中，会为相应的线程分配需要的资源。

### 3.2 run

`run()方法`是不需要用户来调用的，**当通过start方法启动一个线程之后，当线程获得了CPU执行时间，便进入run方法体去执行具体的任务**。注意，继承Thread类必须重写run方法，在run方法中定义具体要执行的任务。

### 3.3 sleep

`sleep()方法`有两个重载版本：

```java
sleep(long millis)     //参数为毫秒
 
sleep(long millis,int nanoseconds)    //第一参数为毫秒，第二个参数为纳秒
```

sleep相当于让线程睡眠，交出CPU，让CPU去执行其他的任务。

但是有一点要非常注意，**sleep方法不会释放锁**，也就是说如果当前线程持有对某个对象的锁，则即使调用sleep方法，其他线程也无法访问这个对象。看下面这个例子就清楚了：

```java
public class Test {
     
    private int i = 10;
    private Object object = new Object();
     
    public static void main(String[] args) throws IOException  {
        Test test = new Test();
        MyThread thread1 = test.new MyThread();
        MyThread thread2 = test.new MyThread();
        thread1.start();
        thread2.start();
    } 
     
     
    class MyThread extends Thread{
        @Override
        public void run() {
            synchronized (object) {
                i++;
                System.out.println("i:"+i);
                try {
                    System.out.println("线程"+Thread.currentThread().getName()+"进入睡眠状态");
                    Thread.currentThread().sleep(10000);
                } catch (InterruptedException e) {
                    // TODO: handle exception
                }
                System.out.println("线程"+Thread.currentThread().getName()+"睡眠结束");
                i++;
                System.out.println("i:"+i);
            }
        }
    }
}
```

输出结果：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002113112929.png)

从上面输出结果可以看出，当Thread-0进入睡眠状态之后，Thread-1并没有去执行具体的任务。只有当Thread-0执行完之后，此时Thread-0释放了对象锁，Thread-1才开始执行。

注意，如果调用了sleep方法，必须捕获`InterruptedException`异常或者将该异常向上层抛出。当线程睡眠时间满后，不一定会立即得到执行，因为此时可能CPU正在执行其他的任务。所以说**调用sleep方法相当于让线程进入阻塞状态**。

### 3.4 yield

调用`yield()`方法**会让当前线程交出CPU权限，让CPU去执行其他的线程**。它跟sleep方法类似，**同样不会释放锁**。但是yield不能控制具体的交出CPU的时间，另外，yield方法只能让拥有相同优先级的线程有获取CPU执行时间的机会。

注意，**调用yield方法并不会让线程进入阻塞状态，而是让线程重回就绪状态，它只需要等待重新获取CPU执行时间**，这一点是和sleep方法不一样的。

### 3.5 join

join()方法有三个重载版本：

```java
join()
join(long millis)     //参数为毫秒
join(long millis,int nanoseconds)    //第一参数为毫秒，第二个参数为纳秒
```

假如在main线程中，调用thread.join方法，则main方法会等待thread线程执行完毕或者等待一定的时间。如果调用的是无参join方法，则等待thread执行完毕，如果调用的是指定了时间参数的join方法，则等待一定的事件。

看下面一个例子：

```java
public class Test {
     
    public static void main(String[] args) throws IOException  {
        System.out.println("进入线程"+Thread.currentThread().getName());
        Test test = new Test();
        MyThread thread1 = test.new MyThread();
        thread1.start();
        try {
            System.out.println("线程"+Thread.currentThread().getName()+"等待");
            thread1.join();
            System.out.println("线程"+Thread.currentThread().getName()+"继续执行");
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    } 
     
    class MyThread extends Thread{
        @Override
        public void run() {
            System.out.println("进入线程"+Thread.currentThread().getName());
            try {
                Thread.currentThread().sleep(5000);
            } catch (InterruptedException e) {
                // TODO: handle exception
            }
            System.out.println("线程"+Thread.currentThread().getName()+"执行完毕");
        }
    }
}
```

输出结果：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002113515660.png)

可以看出，当调用thread1.join()方法后，main线程会进入等待，然后等待thread1执行完之后再继续执行。

实际上调用join方法是调用了Object的`wait()`方法，这个可以通过查看源码得知：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002113550501.png)

**wait方法会让线程进入阻塞状态，并且会释放线程占有的锁，并交出CPU执行权限。**

**由于wait方法会让线程释放对象锁，所以join方法同样会让线程释放对一个对象持有的锁**。具体的wait方法使用在后面文章中给出。

### 3.6 interrupt

`interrupt`，顾名思义，即中断的意思。单独调用interrupt方法可以使得处于阻塞状态的线程抛出一个异常，也就说，它可以用来中断一个正处于阻塞状态的线程；另外，通过`interrupt()`方法和`isInterrupted()`方法来停止正在运行的线程。

下面看一个例子：

```java
public class Test {
     
    public static void main(String[] args) throws IOException  {
        Test test = new Test();
        MyThread thread = test.new MyThread();
        thread.start();
        try {
            Thread.currentThread().sleep(2000);
        } catch (InterruptedException e) {
             
        }
        thread.interrupt();
    } 
     
    class MyThread extends Thread{
        @Override
        public void run() {
            try {
                System.out.println("进入睡眠状态");
                Thread.currentThread().sleep(10000);
                System.out.println("睡眠完毕");
            } catch (InterruptedException e) {
                System.out.println("得到中断异常");
            }
            System.out.println("run方法执行完毕");
        }
    }
}
```

输出结果：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002113933456.png)

从这里可以看出，通过`interrupt()`方法可以中断处于阻塞状态的线程。那么能不能中断处于非阻塞状态的线程呢？看下面这个例子：

```java
public class Test {
     
    public static void main(String[] args) throws IOException  {
        Test test = new Test();
        MyThread thread = test.new MyThread();
        thread.start();
        try {
            Thread.currentThread().sleep(2000);
        } catch (InterruptedException e) {
             
        }
        thread.interrupt();
    } 
     
    class MyThread extends Thread{
        @Override
        public void run() {
            int i = 0;
            while(i<Integer.MAX_VALUE){
                System.out.println(i+" while循环");
                i++;
            }
        }
    }
}
```

运行该程序会发现，while循环会一直运行直到变量i的值超出Integer.MAX_VALUE。所以说直接调用interrupt方法不能中断正在运行中的线程。

但是如果配合`isInterrupted()`能够中断正在运行的线程，因为调用interrupt方法相当于将中断标志位置为true，那么可以通过调用isInterrupted()判断中断标志是否被置位来中断线程的执行。比如下面这段代码：

```java
public class Test {
     
    public static void main(String[] args) throws IOException  {
        Test test = new Test();
        MyThread thread = test.new MyThread();
        thread.start();
        try {
            Thread.currentThread().sleep(2000);
        } catch (InterruptedException e) {
             
        }
        thread.interrupt();
    } 
     
    class MyThread extends Thread{
        @Override
        public void run() {
            int i = 0;
            while(!isInterrupted() && i<Integer.MAX_VALUE){
                System.out.println(i+" while循环");
                i++;
            }
        }
    }
}
```

运行会发现，打印若干个值之后，while循环就停止打印了。

但是一般情况下不建议通过这种方式来中断线程，一般会在MyThread类中增加一个属性 isStop来标志是否结束while循环，然后再在while循环中判断isStop的值。

```java
class MyThread extends Thread{
        private volatile boolean isStop = false;
        @Override
        public void run() {
            int i = 0;
            while(!isStop){
                i++;
            }
        }
         
        public void setStop(boolean stop){
            this.isStop = stop;
        }
    }
```

那么就可以在外面通过调用setStop()方法来终止while循环。

### 3.7 stop

stop方法已经是一个废弃的方法，它是一个不安全的方法。因为调用stop方法会直接终止run方法的调用，并且会抛出一个`ThreadDeath`错误，如果线程持有某个对象锁的话，会完全释放锁，导致对象状态不一致。所以stop方法基本是不会被用到的。

### 3.8 destroy

destroy方法也是废弃的方法。基本不会被使用到。

### 3.9 线程属性方法

| 方法                     | 描述                                                 |
| :------------------------ | :---------------------------------------------------- |
| getId                    | 用来得到线程ID                                       |
| getName和setName         | 用来得到或者设置线程名称                             |
| getPriority和setPriority | 用来获取和设置线程优先级                             |
| setDaemon和isDaemon      | 用来设置线程是否成为守护线程和判断线程是否是守护线程 |
|currentThread| 获取当前线程|

守护线程和用户线程的区别在于：**守护线程依赖于创建它的线程，而用户线程则不依赖。** 举个简单的例子：如果在main线程中创建了一个守护线程，当main方法运行完毕之后，守护线程也会随着消亡。而用户线程则不会，用户线程会一直运行直到其运行完毕。在JVM中，像垃圾收集器线程就是守护线程。

---

在上面已经说到了Thread类中的大部分方法，那么Thread类中的方法调用到底会引起线程状态发生怎样的变化呢？下面一幅图就是在上面的图上进行改进而来的：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181002114640137.png)
