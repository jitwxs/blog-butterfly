---
title: Arthas 初探（4）——Class 命令
tags: Arthas
categories:
  - Java DevOps
  - Arthas
abbrlink: 43dc34ff
date: 2020-12-27 16:55:36
---

## 一、前言

在本章节中，将学习以下 Arthas 的 Class 相关命令，同时我也会附上官方文档的链接，方便大家查阅：

- [sc](https://arthas.aliyun.com/doc/sc.html) Search Class 查看运行中的类信息
- [sm](https://arthas.aliyun.com/doc/sm.html) Search Method 查看类中方法的信息
- [jad](https://arthas.aliyun.com/doc/jad.html) 反编译字节码为源代码
- [mc](https://arthas.aliyun.com/doc/mc.html) Memory Compile 将源代码编译成字节码
- [redefine](https://arthas.aliyun.com/doc/redefine.html) 将编译好的字节码文件加载到 JVM 中运行
- [dump](https://arthas.aliyun.com/doc/dump.html) 将已加载类的 bytecode 下载到特定目录
- [classloader](https://arthas.aliyun.com/doc/classloader.html) 查看 classloader 的继承树，urls，类加载信息

## 二、sc

`sc` 是 Search-Class 的缩写，用于查看 JVM 已加载的类信息，这个命令能搜索出所有已经加载到 JVM 中的 Class 信息。

> class-pattern支持全限定名，如com.taobao.test.AAA，也支持com/taobao/test/AAA 这样的格式，这样，我们从异常堆栈里面把类名拷贝过来的时候，不需要在手动把 `/` 替换为 `.`
>
> sc 默认开启了子类匹配功能，也就是说所有当前类的子类也会被搜索出来，想要精确的匹配，请打开 `options disable-sub-class true` 开关。

| 参数名称              | 参数说明                                                     |
| --------------------- | ------------------------------------------------------------ |
| *class-pattern*       | 类名表达式匹配                                               |
| *method-pattern*      | 方法名表达式匹配                                             |
| [d]                   | 输出当前类的详细信息，包括这个类所加载的原始文件来源、类的声明、加载的ClassLoader等详细信息。 如果一个类被多个ClassLoader所加载，则会出现多次 |
| [E]                   | 开启正则表达式匹配，默认为通配符匹配                         |
| [f]                   | 输出当前类的成员变量信息（需要配合参数-d一起使用）           |
| [x:]                  | 指定输出静态变量时属性的遍历深度，默认为 0，即直接使用 `toString` 输出 |
| `[c:]`                | 指定class的 ClassLoader 的 hashcode                          |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name                   |
| `[n:]`                | 具有详细信息的匹配类的最大数量（默认为100）                  |

（1）模糊搜索，demo 包下所有的类

```shell
[arthas@6096]$ sc demo.*
demo.MathGame
Affect(row-cnt:1) cost in 81 ms.sc demo.*
```

（2）打印 demo.MathGame 类的详细信息

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227170419829.png)

（3）打印 demo.MathGame 类的详细信息 + 变量信息

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227170644232.png)

## 三、sm

`sm` 是 Search-Method 的简写，这个命令能搜索出所有已经加载了 Class 信息的方法信息。

`sm` 命令只能看到由当前类所声明 (declaring) 的方法，父类则无法看到。

| 参数名称              | 参数说明                                    |
| --------------------- | ------------------------------------------- |
| *class-pattern*       | 类名表达式匹配                              |
| *method-pattern*      | 方法名表达式匹配                            |
| [d]                   | 展示每个方法的详细信息                      |
| [E]                   | 开启正则表达式匹配，默认为通配符匹配        |
| `[c:]`                | 指定class的 ClassLoader 的 hashcode         |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name  |
| `[n:]`                | 具有详细信息的匹配类的最大数量（默认为100） |

（1）显示 String 类加载的方法

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227170848562.png)

（2）显示 String 中的 toString 方法详细信息

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227170925022.png)

## 三、jad

`jad` 命令的主要工作是反编译，将 JVM 中实际运行的 class 的 byte code 反编译成 java 代码，便于你理解业务逻辑。

- 在 Arthas Console 上，反编译出来的源码是带语法高亮的，阅读更方便
- 当然，反编译出来的 java 代码可能会存在语法错误，但不影响你进行阅读理解

| 参数名称              | 参数说明                                   |
| --------------------- | ------------------------------------------ |
| *class-pattern*       | 类名表达式匹配                             |
| `[c:]`                | 类所属 ClassLoader 的 hashcode             |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name |
| [E]                   | 开启正则表达式匹配，默认为通配符匹配       |

### 3.1 编译 String 类

```shell
jad <class-pattern>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227171138038.png)

### 3.2 反编译时只显示源代码

默认情况下，反编译结果里会带有 `ClassLoader` 信息，通过 `--source-only` 选项，可以只打印源代码。方便和 [mc](https://arthas.aliyun.com/doc/mc.html)/[redefine](https://arthas.aliyun.com/doc/redefine.html) 命令结合使用。

```shell
jad --source-only <class-pattern>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227171243641.png)

### 3.3 反编译指定的函数

```shell
jad <class-pattern> <method-pattern>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227171402394.png)

## 四、mc

`mc` 是 Memory Compiler 的缩写，编译 `.java` 文件生成 `.class` 。

（1）在内存中编译 Hello.java 为 Hello.class

```shell
mc /root/Hello.java
```

（2）可以通过 -d 命令指定输出目录

```shell
mc -d /root/bbb /root/Hello.java
```

## 五、redefine

加载外部的 `.class` 文件，redefine JVM 已加载的类。

注意， redefine 后的原来的类不能恢复，redefine 有可能失败（比如增加了新的 field），参考 JDK 本身的文档。

| 参数名称              | 参数说明                                   |
| --------------------- | ------------------------------------------ |
| [c:]                  | ClassLoader的hashcode                      |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name |
| [p:]                  | 外部的`.class`文件的完整路径，支持多个     |

### 5.1 自身限制

- 不允许新增加 field / method
- 正在跑的函数，没有退出不能生效，比如下面新增加的 `System.out.println`，只有 `run()` 函数里的会生效

```java
public class MathGame {
    public static void main(String[] args) throws InterruptedException {
        MathGame game = new MathGame();
        while (true) {
            game.run();
            TimeUnit.SECONDS.sleep(1);
            // 这个不生效，因为代码一直跑在 while里
            System.out.println("in loop");
        }
    }
 
    public void run() throws InterruptedException {
        // 这个生效，因为run()函数每次都可以完整结束
        System.out.println("call run()");
        try {
            int number = random.nextInt();
            List<Integer> primeFactors = primeFactors(number);
            print(number, primeFactors);
 
        } catch (Exception e) {
            System.out.println(String.format("illegalArgumentCount:%3d, ", illegalArgumentCount) + e.getMessage());
        }
    }
}
```

### 5.2 命令冲突

- `reset` 命令对 `redefine` 的类无效。如果想重置，需要 `redefine` 原始的字节码。
- `redefine` 命令和 `jad`/`watch`/`trace`/`monitor`/`tt` 等命令会冲突。执行完`redefine` 之后，如果再执行上面提到的命令，则会把 `redefine` 的字节码重置。 原因是 JDK 本身 redefine 和 Retransform 是不同的机制，同时使用两种机制来更新字节码，只有最后修改的会生效。

### 5.3 实战

（1）反编译 MathGame 类

```shell
jad demo.MathGame > MathGame.java
```

（2）编辑该类，增加两行输出。一行在 main 方法死循环中，一行在 run() 方法首行。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227191848404.png)

（2）编译修改后的类

```shell
mc -d C://Users//Jitwxs//Downloads//MathGame.java C://Users//Jitwxs//Downloads
```

> 原谅我这边没有截图，因为我在 Windows 电脑上执行 mc 命令失败了。
>
> 先是提示我 Can not load JavaCompiler from javax.tools.ToolProvider#getSystemJavaCompiler(), please confirm the application running in JDK not JRE。
>
> 解决后又报 FileNotFoundException: C:\Users\Jitwxs\Downloads (拒绝访问) 的错。
>
> 这就告诉我们，虽然是跨平台的，但还是不要用 Windows 去做命令行开发，否则慢慢踩坑吧。。

（3）加载最新的字节码

```shell
redefine C://Users//Jitwxs//Downloads//MathGame.class
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227193548404.png)

### 5.6 上传 .class 文件到服务器的技巧

使用 `mc` 命令来编译 `jad` 的反编译的代码有可能失败。可以在本地修改代码，编译好后再上传到服务器上。有的服务器不允许直接上传文件，可以使用 `base64` 命令来绕过。

1. 在本地先转换 `.class` 文件为 base64，再保存为 result.txt

   ```shell
   base64 < Test.class > result.txt
   ```
2. 到服务器上，新建并编辑 `result.txt`，复制本地的内容，粘贴再保存

3. 把服务器上的 `result.txt` 还原为 `.class`

   ```shell
   base64 -d < result.txt > Test.class
   ```

4. 用 MD5 命令计算哈希值，校验是否一致

## 六、dump

dump 已加载类的 bytecode 到特定目录。

| 参数名称              | 参数说明                                   |
| --------------------- | ------------------------------------------ |
| *class-pattern*       | 类名表达式匹配                             |
| `[c:]`                | 类所属 ClassLoader 的 hashcode             |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name |
| `[d:]`                | 设置类文件的目标目录                       |
| [E]                   | 开启正则表达式匹配，默认为通配符匹配       |

（1）把 String 类的字节码文件保存到当前目录下

```shell
[arthas@18132]$ dump java.lang.String -d .
 HASHCODE  CLASSLOADER  LOCATION
 null                   C:\Users\Jitwxs\Downloads\java\lang\String.class
Affect(row-cnt:1) cost in 11 ms.
```

（2）把 demo 包下所有的类的字节码文件保存到当前目录下

```shell
[arthas@18132]$ dump demo.* -d .
 HASHCODE  CLASSLOADER                                    LOCATION
 5c647e05  +-sun.misc.Launcher$AppClassLoader@5c647e05    C:\Users\Jitwxs\logs\arthas\classdump\sun.misc.Launcher$Ap
             +-sun.misc.Launcher$ExtClassLoader@28d93b30  pClassLoader-5c647e05\demo\MathGame.class
Affect(row-cnt:1) cost in 10 ms.
```

## 七、classloader

`classloader` 命令将 JVM 中所有的classloader的信息统计出来，并可以展示继承树，urls等。

可以让指定的 classloader 去 getResources，打印出所有查找到的 resources 的 url。对于 `ResourceNotFoundException` 比较有用。

| 参数名称              | 参数说明                                   |
| --------------------- | ------------------------------------------ |
| [l]                   | 按类加载实例进行统计                       |
| [t]                   | 打印所有ClassLoader的继承树                |
| [a]                   | 列出所有ClassLoader加载的类，请谨慎使用    |
| `[c:]`                | ClassLoader的hashcode                      |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name |
| `[c: r:]`             | 用ClassLoader去查找resource                |
| `[c: load:]`          | 用ClassLoader去加载指定的类                |

（1）按类加载器的类型查看统计信息

```shell
classloadaer
```

![image-20201227195250913](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195250913.png)

（2）按类加载实例查看统计信息

```shell
classloadaer -l
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195416801.png)

（3）查看 ClassLoader 的继承树

```shell
classloadaer -t
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195450749.png)

（4）通过类加载器的 hash，查看此类加载器实际所在的位置

> 注意hashcode是变化的，需要先查看当前的ClassLoader信息，提取对应ClassLoader的hashcode。对于只有唯一实例的 ClassLoader 可以通过 class name 指定，使用起来更加方便

```shell
 classloader -c 1c1582d6
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195601884.png)

（5）使用 ClassLoader 去查找指定资源 resource 所在的位置

```shell
classloader -c 1c1582d6 -r META-INF/MANIFEST.MF
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195726132.png)

（6）使用 ClassLoader 去加载类

```shell
 classloader -c 5c647e05 --load demo.MathGame
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202012/20201227195937705.png)
