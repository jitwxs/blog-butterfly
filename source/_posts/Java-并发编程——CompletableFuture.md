---
title: Java 并发编程——CompletableFuture
categories:
  - Java
  - 并发编程
abbrlink: d00bfcd
date: 2020-12-12 18:10:44
---

##  一、前言

本文向大家安利一下 JDK 8 中的并发工具类 `CompletableFuture`，利用它可以方便的进行并发、异步的流式、协调操作。自从用了它，一下次就不想用 CountdownLatch 和 Future 了，赶紧一起来了解下吧。

## 二、实例化

CompletableFuture 提供了两组静态方法来实例化出对象，如下所示。

```java
CompletableFuture.runAsync(Runnable runnable);
CompletableFuture.runAsync(Runnable runnable, Executor executor);

CompletableFuture.supplyAsync(Supplier<U> supplier);
CompletableFuture.supplyAsync(Supplier<U> supplier, Executor executor)
```

- `runAsync()` 方法接收的是 Runnable 实例，用于异步任务不需要返回值的情况
- `supplyAsync()` 方法接受的是 Supplier 实例，用于余部任务需要返回值的情况
- 这两组方法同时提供了 Executor 参数的重载，用于指定线程池来执行。如果不指定，则使用默认的 `ForkJoinPool.commonPool()` 线程池，类型与 parallelStream。

## 三、流式计算

利用 CompletableFuture ，可以实现流式计算，即当任务 A 执行完毕后，在执行任务 B。

这里根据业务不同变种很多，例如 任务 A 需不需要有返回值，任务 B 需不要参数，以及任务 B 自身需不需要返回值等。

```java
CompletableFuture.runAsync(() -> {}).thenRun(() -> {}); 
CompletableFuture.runAsync(() -> {}).thenAccept(resultA -> {}); 
CompletableFuture.runAsync(() -> {}).thenApply(resultA -> "resultB");
```

上面三行代码演示的是任务 A 没有返回值的情况，即 `runAsync()`：

- `thenRun()`：任务 B 在任务 A 执行完毕后执行，任务 B 无需参数也无返回值。
- `thenAccept()`：任务 B 在任务 A 执行完毕后执行，任务 B 需要任务 A 的返回结果作为参数，任务 B 无需返回值。（实际上此处任务B的参数一定为 null，因为任务 A 无返回值）
- `thenApply()`：任务 B 在任务 A 执行完毕后执行，任务 B 需要任务 A 的返回结果作为参数，任务 B 自身也有返回值。实际上此处任务B的参数一定为 null，因为任务 A 无返回值）

> `thenRun()` 、`thenAccept()`、`thenApply()` 也有对应的带 Async 后缀的方法，需要注意的是，并不是说带 Async 就是异步，不带就是同步的意思。而是说带 Async 会被重新放进线程池，由线程池决定何时执行；不带 Async 就会仍然由当前线程执行，不再重新放入线程池中。

对应的，下面三行代码演示的是任务 A 有返回值的情况，即 `supplyAsync()`：

```java
CompletableFuture.supplyAsync(() -> "resultA").thenRun(() -> {});
CompletableFuture.supplyAsync(() -> "resultA").thenAccept(resultA -> {});
CompletableFuture.supplyAsync(() -> "resultA").thenApply(resultA -> resultA + " resultB");
```

另外不一定只有任务 A、任务 B，只要业务需要，后面也许还有任务 C、任务 D...

```java
public class CompletableFutureTest {
    public static Executor executor = Executors.newFixedThreadPool(5);

    public static void main(String[] args) throws InterruptedException {
        IntStream.range(1, 10).boxed()
                .forEach(e -> CompletableFuture.supplyAsync(() -> pow3(e), executor)
                        .thenApplyAsync(CompletableFutureTest::sqrt, executor)
                        .thenAcceptAsync(CompletableFutureTest::print));

        Thread.sleep(10000);
    }

    private static double pow3(double input) {
        return Math.pow(input, 3);
    }

    private static double sqrt(double input) {
        return Math.sqrt(input);
    }

    private static void print(double input) {
        System.out.println(input);
    }
}
```

上面代码举了个例子，任务 A 负责计算输入值的三次方，任务 B 负责计算输入值的开根号，任务 C 负责打印输出值。

程序运行结果如下：（请忽略没有按照 1 ~ 10 的先后顺序输出的问题）

```java
1.0
5.196152422706632
22.627416997969522
18.520259177452136
8.0
2.8284271247461903
11.180339887498949
14.696938456699069
27.0
```

## 四、异常处理

CompletableFuture 提供了 `exceptionally()` 和 `handle()` 方法来进行异常处理，如下所示。

```java
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn);

public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
```

 CompletableFuture 某个任务执行抛出异常，是不会被被抛出到外部来的，同时会影响后续的任务执行。以上一节最后代码为例，我们修改 `pow3()` 代码，让其抛出异常：

```java
private static double pow3(double input) {
    throw new IllegalArgumentException("xxx");
    // return Math.pow(input, 3);
}
```

重新运行程序后，控制台没有打印任何异常日志，同时后续任务也终止执行了。

### 4.1 exceptionally

使用 `exceptionally()` ，可以当任务内部出现异常时，自行做业务处理，并返回新的结果，并传递给后续任务。

修改我们的例子，main() 方法中代码如下：

```java
IntStream.range(1, 10).boxed()
    .forEach(e -> CompletableFuture.supplyAsync(() -> pow3(e), executor)
            .exceptionally(ex -> {
                System.out.println("pow3() function exception:" + ex.getMessage());
                return 1.0D;
            })
            .thenApplyAsync(CompletableFutureTest::sqrt, executor)
            .thenAcceptAsync(CompletableFutureTest::print));
```

重新运行后输出如下：

```java
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
1.0
1.0
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
pow3() function exception:java.lang.IllegalArgumentException: xxx
1.0
1.0
1.0
1.0
1.0
pow3() function exception:java.lang.IllegalArgumentException: xxx
1.0
1.0
```

### 4.2 handle

除了使用 exceptionally 外，`handle()` 也可以用于处理异常，如下所示：

```java
IntStream.range(1, 10).boxed()
    .forEach(e -> CompletableFuture.supplyAsync(() -> pow3(e), executor)
            .handle((result, ex) -> {
                if(ex != null) {
                    System.out.println("pow3() function exception:" + ex.getMessage());
                    return 1.0D;
                } else {
                    return result;
                }
            })
            .thenApplyAsync(CompletableFutureTest::sqrt, executor)
            .thenAcceptAsync(CompletableFutureTest::print));
```

`handle()` 方法存在有两个参数，参数1为任务本身的返回结果，参数2为任务内抛出的异常。这两个参数至少有一个必然为 null，因此可以通过判断 ex 是否为 null 进行异常处理。

>当然这两个参数也可以都为 null，因为你可以让任务返回 null。

## 五、栅栏

栅栏需求在实际开发中十分普遍，例如：

- 存在任务 A，需要异步处理，所有线程处理完毕后，再做其他业务处理
- 存在任务 A、任务 B、任务 C，任务 A 和 任务 B 可以并行执行，但是必须 任务 A 和 任务 B 全部执行完毕后才能执行任务 C

以往实现这些需求时候，我们会使用 CountdownLatch 和 Future，现在可以多一个选择了。

### 5.1 两个任务的栅栏

```java
CompletableFuture<String> cfA = CompletableFuture.supplyAsync(() -> "resultA");
CompletableFuture<String> cfB = CompletableFuture.supplyAsync(() -> "resultB");

cfA.thenAcceptBoth(cfB, (resultA, resultB) -> {});
cfA.thenCombine(cfB, (resultA, resultB) -> "result A + B");
cfA.runAfterBoth(cfB, () -> {});
```

上面代码介绍了如何实现任务 A 和任务 B 的并行执行：

- `thenAcceptBoth()`：并行执行，需要两个任务的结果作为参数，进行后续处理，无返回值。
- `thenCombine()`：并行执行，需要两个任务的结果作为参数，进行后续处理，有返回值。
- `runAfterBoth()`：并行执行，不需要两个任务的结果作为参数，进行后续处理，无返回值。

下面的程序演示了数字 1 ~ 10 并行进行开根号和三次方计算，并在完成这二者计算后输出结果。

```java
public class CompletableFutureTest {
    public static Executor executor = Executors.newFixedThreadPool(5);

    public static void main(String[] args) throws InterruptedException {
        IntStream.range(1, 10).boxed()
                .forEach(e -> CompletableFuture.supplyAsync(() -> pow3(e), executor)
                        .thenAcceptBoth(CompletableFuture.supplyAsync(() -> sqrt(e), executor), (resultA, resultB) -> System.out.println("resultA:" + resultA + ", resultB:" + resultB)));

        Thread.sleep(10000);
    }

    private static double pow3(double input) {
        return Math.pow(input, 3);
    }

    private static double sqrt(double input) {
        return Math.sqrt(input);
    }
}
```

程序运行结果如下：

```java
resultA:1.0, resultB:1.0
resultA:8.0, resultB:1.4142135623730951
resultA:27.0, resultB:1.7320508075688772
resultA:343.0, resultB:2.6457513110645907
resultA:125.0, resultB:2.23606797749979
resultA:216.0, resultB:2.449489742783178
resultA:64.0, resultB:2.0
resultA:729.0, resultB:3.0
resultA:512.0, resultB:2.8284271247461903
```

### 5.2 多个任务的栅栏

上一小节介绍的仅适用于两个任务之间的并行计算，如果我们想要实现更多任务的并行计算，CompletableFuture 也为我们提供了 `allOf()` 和 `anyOf()` 方法。

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {}

public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs) {}
```

#### 5.2.1 allOf

下面代码介绍了 `allOf()` 的用法，它聚合了多个任务，并通过 `join()` 方法进行阻塞等待。需要注意的是，该方法是没有返回值的。

```java
CompletableFuture cfA = CompletableFuture.supplyAsync(() -> "resultA");
CompletableFuture cfB = CompletableFuture.supplyAsync(() -> 123);
CompletableFuture cfC = CompletableFuture.supplyAsync(() -> "resultC");

CompletableFuture<Void> future = CompletableFuture.allOf(cfA, cfB, cfC);
// 所以这里的 join() 将阻塞，直到所有的任务执行结束
future.join();
```

下面的程序演示了数字 1 ~ 10 并行进行三次方计算，并在完所有数字全部计算完成后输出结果。

```java
public class CompletableFutureTest {
    public static void main(String[] args) {
        List<Double> result = new ArrayList<>();

        CompletableFuture.allOf(IntStream.range(1, 10).boxed()
                .map(e -> CompletableFuture.runAsync(() -> result.add(pow3(e))))
                .toArray(CompletableFuture[]::new)
        ).join();

        System.out.println(result);
    }

    private static double pow3(double input) {
        return Math.pow(input, 3);
    }
}
```

程序运行结果如下：

```java
[1.0, 8.0, 27.0, 64.0, 125.0, 729.0, 216.0, 343.0, 512.0]
```

#### 5.2.2 anyOf

anyOf 也很容易理解，就是只要有任意一个 CompletableFuture 任务执行完成就会返回。

下面的程序演示了数字 1 ~ 10 并行进行三次方计算，只要有任意一个数字计算完成后就会输出结果。

```java
public class CompletableFutureTest {
    public static void main(String[] args) {
        Object anyResult = CompletableFuture.anyOf(IntStream.range(1, 10).boxed()
                .map(e -> CompletableFuture.supplyAsync(() -> pow3(e)))
                .toArray(CompletableFuture[]::new)
        ).join();

        System.out.println(anyResult);
    }

    private static double pow3(double input) {
        return Math.pow(input, 3);
    }
}
```

由于线程执行先后的不同，每次运行输出结果都不一样，因此我就不贴出运行接过来。

> 之所以它的返回值使用的是 Object，因为每个任务可能返回的类型不同，它自身无法判断类型。
