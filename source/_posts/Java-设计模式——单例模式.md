---
title: Java 设计模式——单例模式
categories: 
  - Java
  - 设计模式
abbrlink: e9b2c5a3
date: 2018-09-26 17:54:29
references:
  - name: Java设计模式之单例模式详解
    url: https://www.cnblogs.com/garryfu/p/7976546.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 常见的几种单例模式
    url: https://www.cnblogs.com/Ycheng/p/7169381.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 单例模式唯一实例为什么必须为静态变量
    url: https://zhidao.baidu.com/question/2206072272164938188.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 为什么用枚举类来实现单例模式越来越流行？
    url: https://juejin.im/post/5d64ca62f265da03b638bb47
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 为什么强烈建议大家使用枚举来实现单例
    url: https://msd.misuland.com/pd/3148108429789238982
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、单例模式的介绍

###  1.1 什么是单例模式

`单例模式`指的是**一个类只会有一个实例**，即一个类只有一个对象实例。它的特点有：

- 单例类**只能有一个**实例

- 单例类**必须自己创建自己**的唯一实例

- 单例类**必须给所有其他对象提供这一实例**

### 1.2 单例模式的应用场景

（1）一个系统中可以存在多个打印任务，但是只能有一个正在工作的任务；售票时，一共有100张票，可有有多个窗口同时售票，但需要保证不要超售（这里的票数余量就是单例，售票涉及到多线程）。

（2）在前端创建工具箱窗口，工具箱要么不出现，出现也只出现一个。

遇到问题：每次点击菜单都会重复创建“工具箱”窗口。

解决方案：使用 if 语句，在每次创建对象的时候首先进行判断是否为 null ，如果为 null 再创建对象。

（3）如果在 5 个地方需要实例出工具箱窗体。

遇到问题：这个小 bug 需要改动 5 个地方，并且代码重复，代码利用率低

解决方案：利用单例模式，保证一个类只有一个实例，并提供一个访问它的全局访问点。

## 二、传统单例模式的实现

传统的单例模式实现可以分为`懒汉式`和`饿汉式`：

- 懒汉式单例模式：**在类加载时不初始化。**

- 饿汉式单例模式：**在类加载时就完成了初始化**，所以类加载比较慢，但获取对象的速度快。

### 2.1 懒汉

懒汉实现又分为线程安全和线程不安全这两种写法。先说下线程不安全的写法，这种写法是**懒加载很明显**，但是在多线程不能正常工作，存在线程安全问题。

```java
/*
 * 懒汉 线程不安全
 */
class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

下面是线程安全的写法，这种写法在`getInstance()`方法中加入了`synchronize` 锁。能够在多线程中很好的工作，而且**也具备很好的懒加载**，但是效率很低（因为锁），并且大多数情况下不需要同步这个功能。

```java
/*
 * 懒汉 线程安全
 */
class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

懒汉模式的**优点**：

- 实现了懒加载，节约了内存空间

懒汉模式的**缺点**：

- 在不加锁的情况下，线程不安全，可能出现多份实例
- 在加锁的情况下，会是程序串行化，使系统有严重的性能问题

### 2.2 饿汉

另一种单例类别是饿汉，这种方式基于 ClassLoder 机制避免了多线程的同步问题，不过 instance 在类装载时就实例化，这时候初始化 instance 显然**没有达到懒加载的效果**。

```java
/*
 * 饿汉
 */
class Singleton {
    private static Singleton instance = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```
对上面代码稍微修改下，就出现了饿汉的一个变种，本质上是一样的。

```java
/*
 * 饿汉 变种
 */
class Singleton {
    private static Singleton instance = null;

    static {
        instance = new Singleton();
    }

    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```

饿汉模式的**优点**：

- 由于使用了 static 关键字，保证了在引用这个变量时，关于这个变量的所以写入操作都完成，所以保证了 JVM 层面的线程安全

饿汉模式的**缺点**：

- 不能实现懒加载，造成空间浪费，如果一个类比较大，我们在初始化的时就加载了这个类，但是我们长时间没有使用这个类，这就导致了内存空间的浪费。

### 2.3 静态内部类

```java
class Singleton {
    private static class SingletonHolder {
        private static Singleton instance = new Singleton();
    }

    private Singleton() {}

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```

这种方式同样利用了ClassLoder的机制来保证初始化instance时只有一个线程，但是恶汉模式只要Singleton类被装载了，那么instance就会被实例化，而这种方式只有显示通过调用`getInstance()`方法时，才会显示装载SingletonHolder类，从而实例化instance，从而**达到懒加载**的效果。

### 2.4 双重校验锁

在懒汉模式中对加锁的处理，对于`getInstance()`方法来说，绝大部分的操作都是读操作，读操作是线程安全的，所以我们没必让每个线程必须持有锁才能调用该方法，我们需要调整加锁的问题。由此也产生了一种新的实现模式：**双重检查锁模式**。它是线程安全版懒汉的升级版，在 JDK1.5 之后，使用双重检查锁定才能够正常达到单例效果。

```java
class Singleton {
    private volatile static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

### 2.5 唯一实例为什么是 static

单例模式实现过程如下：

1. 首先，**将该类的构造函数私有化**（目的是禁止其他程序创建该类的对象）；

2. 其次，**在本类中自定义一个对象**（既然禁止其他程序创建该类的对象，就要自己创建一个供程序使用，否则类就没法用，更不是单例）；

3. 最后，**提供一个可访问类自定义对象的类成员方法**（对外提供该对象的访问方式）。

直白的讲就是，你不能用该类在其他地方创建对象，而是通过该类自身提供的方法访问类中的那个自定义对象。那么问题的关键来了，程序调用类中方法只有两种方式：

- 创建类的一个对象，用该对象去调用类中方法；

- 使用类名直接调用类中方法，格式“类名.方法名()”；

上面说了，构造函数私有化后第一种情况就不能用，只能使用第二种方法。

而**使用类名直接调用类中方法，类中方法必须是静态的，而静态方法不能访问非静态成员变量，因此类自定义的实例变量也必须是静态的。**

### 2.6 双重校验锁实现为什么要加 volatile

先说结论，作用主要有两个：

1. **保证该变量在多线程下的可见性**

2. **限制编译器指令重排**

第一点就不解释了，关于 volatile 详细介绍可以看这篇文章：[《Java并发编程——volatile关键字解析》](/dc6706eb.html)

关于第二点，首先解释下编译器的指令重排序，一般来说，处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。

```java
int a = 10;		//语句1
int r = 2;		//语句2
a = a + 3;		//语句3
r = a * a;		//语句4
```

这段代码有4个语句，那么可能的一个执行顺序是： 语句2—>语句1—>语句3—>语句4。但是不可能是： 语句2—>语句1—>语句4—> 语句3。

因为处理器在进行重排序时是会考虑指令之间的数据依赖性，如果一个指令 Instruction 2 必须用到 Instruction 1 的结果，那么处理器会保证 Instruction 1 会在 Instruction 2 之前执行。虽然重排序不会影响单个线程内程序执行的结果，但是多线程下就有可能出现问题，例如：

```java
//线程1:
context = loadContext();   //语句1
inited = true;             //语句2

//线程2:
while(!inited ){
  sleep()
}
doSomethingwithconfig(context);
```

上面代码中，由于语句1和语句2没有数据依赖性，因此可能会被重排序。假如发生了重排序，在线程1执行过程中先执行语句2，而此是线程2会以为初始化工作已经完成，那么就会跳出while循环，去执行 doSomethingwithconfig(context) 方法，而此时 context 并没有被初始化，就会导致程序出错。

介绍完指令重排序，下面言归正传：

对于Java编译器而言，初始化实例和将对象地址写到字段中并非是原子操作，且这两个阶段的执行顺序是未定义的。假设某个线程执行了 new Singleton()，构造方法还未被调用，编译器仅仅为该对象分配了内存空间并设定默认值，此时若其他线程调用 getInstance() 方法，由于 instance != null，但是此时 instance 对象还没有被赋予真正的有效值，从而无法取到正确的单例对象。

**使用 volatile 关键字限制编译器对它的相关读写操作，对它的读写操作进行指令重排，确定对象实例化后才返回引用。**

## 三、反射与序列化

### 3.1 反射问题

《Effective Java》这本书中着重推荐的一种实现。在这本书中，首先它对传统饿汉、懒汉实现评价如下：

> 享有特权的客户端可以借助 AccessibleObject.setAccessible 方法，通过反射机制调用私有构造器。

也就是说利用反射机制是可以破坏它的单例性的，举个例子：

```java
public class SingletonTest {
    public static void main(String[] args) throws Exception {
        Singleton s1 = Singleton.getInstance();
        Singleton s2 = Singleton.getInstance();

        Constructor<Singleton> constructor=Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton s3 = constructor.newInstance();

        System.out.println(s1 + "\n" + s2 + "\n" + s3);
        System.out.println("正常情况下，实例化两个实例是否相同：" + (s1 == s2));
        System.out.println("通过反射攻击单例模式情况下，实例化两个实例是否相同：" + (s1 == s3));
    }
}

class Singleton {
    private volatile static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}

/*
Singleton@5e2de80c
Singleton@5e2de80c
Singleton@1d44bcfa
正常情况下，实例化两个实例是否相同：true
通过反射攻击单例模式情况下，实例化两个实例是否相同：false
*/
```

通过反射机制，调用私有的构造方法，就可以破坏它的单例性，解决办法也很简单，在构造方法中抛出异常即可。

```java
public class SingletonTest {
    public static void main(String[] args) throws Exception {
        Singleton s1 = Singleton.getInstance();
        Singleton s2 = Singleton.getInstance();

        Constructor<Singleton> constructor=Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton s3 = constructor.newInstance();

        System.out.println(s1 + "\n" + s2 + "\n" + s3);
        System.out.println("正常情况下，实例化两个实例是否相同：" + (s1 == s2));
        System.out.println("通过反射攻击单例模式情况下，实例化两个实例是否相同：" + (s1 == s3));
    }
}

class Singleton {
    private volatile static Singleton singleton;

    private Singleton() throws IllegalAccessException {
        if(singleton != null) {
            throw new IllegalAccessException("Do not allow access");
        }
    }

    public static Singleton getInstance() throws IllegalAccessException {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}

/*
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at SingletonTest.main(SingletonTest.java:23)
Caused by: java.lang.IllegalAccessException: Do not allow access
	at Singleton.<init>(SingletonTest.java:36)
	... 5 more
*/
```

### 3.2 序列化问题

普通的 Java 类的反序列化过程中，会通过反射调用类的默认构造函数来初始化对象。所以即使单例中构造函数是私有的，也会被反射给破坏掉。由于反序列化后的对象是重新 new 出来的，所以这就破坏了单例。

如下代码所示，如果单例类实现了序列化接口，序列化前和序列化后不是同一个对象。

```java
public class SingletonTest {
    public static void main(String[] args) throws Exception {
        Singleton s1 = Singleton.getInstance();
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("SerSingleton.obj"));
        oos.writeObject(s1);
        oos.flush();
        oos.close();

        FileInputStream fis = new FileInputStream("SerSingleton.obj");
        ObjectInputStream ois = new ObjectInputStream(fis);
        Singleton s2 = (Singleton)ois.readObject();
        ois.close();
        System.out.println(s1+"\n"+s2);
        System.out.println("序列化前后两个是否同一个："+(s1==s2));
    }
}

class Singleton implements Serializable {
    private volatile static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}

/*
Singleton@66d3c617
Singleton@7291c18f
序列化前后两个是否同一个：false
*/
```

## 四、枚举类实现

枚举类实现单例模式被认为是最完美的实现，它规避了传统单例模式的反射问题和序列化问题。

```java
public enum Singleton {
    INSTANCE;
    public static Singleton getInstance(){
        return INSTANCE;
    }
}
```

首先来测试下是否存在反射问题：

```java
public class SingletonTest {
    public enum Singleton {
        INSTANCE;
        public static Singleton getInstance(){
            return INSTANCE;
        }
    }

    public static void main(String[] args) throws Exception {
        Singleton s1 = Singleton.getInstance();
        Singleton s2 = Singleton.getInstance();
        System.out.println("正常情况下，实例化两个实例是否相同：" + (s1 == s2));

        Constructor<Singleton> constructor=Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton s3 = constructor.newInstance();

        System.out.println("通过反射攻击单例模式情况下，实例化两个实例是否相同：" + (s1 == s3));
    }
}

/*
正常情况下，实例化两个实例是否相同：true
Exception in thread "main" java.lang.NoSuchMethodException: SingletonTest$Singleton.<init>()
	at java.lang.Class.getConstructor0(Class.java:3082)
	at java.lang.Class.getDeclaredConstructor(Class.java:2178)
	at SingletonTest.main(SingletonTest.java:22)
*/
```

当反射调用构造方法时，抛出异常，这里的根本原因是反射对于枚举类型，无法通过构造方法创建实例。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201908/2019083000150824.png)

再来看看它在序列化中的表现：

```java
public class SingletonTest {
    public enum Singleton {
        INSTANCE;
        public static Singleton getInstance(){
            return INSTANCE;
        }
    }

    public static void main(String[] args) throws Exception {
        Singleton s1 = Singleton.getInstance();
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("SerSingleton.obj"));
        oos.writeObject(s1);
        oos.flush();
        oos.close();

        FileInputStream fis = new FileInputStream("SerSingleton.obj");
        ObjectInputStream ois = new ObjectInputStream(fis);
        Singleton s2 = (Singleton)ois.readObject();
        ois.close();
        System.out.println(s1+"\n"+s2);
        System.out.println("序列化前后两个是否同一个："+(s1==s2));
    }
}

/*
INSTANCE
INSTANCE
序列化前后两个是否同一个：true
*/
```

在序列化的时候Java仅仅是将枚举对象的 name 属性输出到结果中，反序列化的时候则是通过 `java.lang.Enum` 的 `valueOf` 方法来根据名字查找枚举对象。同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了`writeObject`、`readObject`等方法。
