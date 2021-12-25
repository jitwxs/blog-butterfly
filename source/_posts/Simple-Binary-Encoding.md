---
title: Simple Binary Encoding
tags:
  - Aeron
  - Agrona
  - SBE
categories:
  - Java High Performance
  - Aeron
related_repos:
  - name: aeron-sample
    url: 'https://github.com/jitwxs/blog-sample/tree/master/javase-sample/aeron-sample'
references:
  - name: github wiki
    url: 'https://github.com/real-logic/simple-binary-encoding/wiki'
  - name: simple-binary-encoding
    url: 'https://aeroncookbook.com/simple-binary-encoding/overview/'
  - name: Inside Simple Binary Encoding (SBE)
    url: 'https://ashkrit.blogspot.com/2018/06/inside-simple-binary-encoding-sbe.html'
abbrlink: 7cdf7e7b
date: 2021-12-25 14:17:00
---

## 一、前言

在本篇文章中，一起来学习下 [SBE(Simple Binary Encoding)](https://github.com/real-logic/simple-binary-encoding) 传输协议，它和 [protobuf](https://github.com/protocolbuffers/protobuf) 一样，都是二进制的传输协议，比 protobuf 传输性能更高，其涉及灵感来源于 [fast binary variant of FIX](https://github.com/FIXTradingCommunity/fix-simple-binary-encoding)，最初的涉及目标就是为了应用于金融级的低延迟交易系统中。在开源软件 [Aeron](https://github.com/real-logic/aeron) 中，也广泛的使用 SBE 作为数据的传输媒介。

## 二、设计原则

### 2.1 Copy-Free

网络和存储系统处理数据时通常需要对数据缓冲区进行编码和解码。Copy-Free 的原则是不使用任何中间缓冲区来编码和解码数据。如果使用了中间缓存区，会因为多次的复制数据产生性能损耗。

SBE 采用直接对底层缓冲区进行编码和解码的方式，这样带来的限制是不支持直接发送长度大于传输缓冲区的数据，对于这种情况，需要进行分段发送和数据重组。

### 2.2 Native Type Mapping

Copy-Free 模式通过将数据直接编码为底层缓冲区中的本地类型而得到性能提升。比如 64 位整数可以作为单个 x86_64 MOV 指令直接编码到底层缓冲区中。如果数据的[字节序](https://www.rfc-editor.org/ien/ien137.txt)（大端/小端）和 CPU 的不一致的化，那么数据在写入底层缓冲区前可以在寄存器中使用 x86 的 BSWAP 指令完成交换。

### 2.3 Allocation-Free

对象的创建会导致 CPU 的缓存减少，从而降低效率。并且后期还需要去管理并释放这些对象。对于 Java 来说，这个过程是由垃圾收集器完成的，它通过触发持续时间不等的 STW（Stop The World）来完成内存回收（新生代的 C4 垃圾收集器除外）。C++ 相对好点，但当内存释放回内存池的时候引入锁机制，也会产生性能开销。
SBE 编解码器使用了享元（flyweight）模式，来避免分配问题。flyweight 窗口在底层缓冲区上直接对数据进行编码和解码，通过消息头中的 templateId 字段来选择适当类型的 flyweigh。如果消息中的字段需要保留岛处理流程之外，则需要单独复制出来。

```java
public class ShapeFactory {
    private static final Map<String, Shape> circleMap = new HashMap<>();
    // 享元模式，存在则直接返回对象复用，不存在则创建
    public static Shape getCircle(String color) {
        Circle circle = (Circle)circleMap.get(color);
        if(circle == null) {
            circle = new Circle(color);
            circleMap.put(color, circle);
            System.out.println("Creating circle of color : " + color);
        }
        return circle;
    }
} 
```

### 2.4 Streaming Access

现代内存子系统已经变得愈发复杂，[访问内存](https://mechanical-sympathy.blogspot.com/2012/08/memory-access-patterns-are-important.html)的算法模式很大程度上决定了性能和一致性。采用基于流的方式以升序模式访问内存地址，可以获得最佳性能和最一致的延迟。
SBE 编解码器根据底层缓冲区中 position 的向前递进对数据进行编码和解码。虽然可以进行一定程度的回溯，但从性能和延迟的角度来看这种操作是非常不鼓励的。

### 2.5 Word Aligned Access

当我们不在 word 的边界位置访问数据时，许多 CPU 架构会表现出显著的性能问题。一个 word 的起始地址应该是其以字节为单位大小的倍数，例如 64 位整数只能从字节地址能被 8 整除的地址开始，32位整数只能从被 4 整除的字节地址开始，以此类推。

SBE 模式支持定义数据中字段起始位置的偏移量（offset）的概念。它假设数据是被封装在 8 字节大小边界的协议帧中。为了实现紧凑与高效，消息字段应该按其类型和大小降序排序。

```c
typedef struct _tcp_hdr
{
    unsigned short src_port; //源端口号
    unsigned short dst_port; //目的端口号
    unsigned int seq_no; //序列号
    unsigned int ack_no; //确认号
#if LITTLE_ENDIAN
    unsigned char reserved_1:4; //保留6位中的4位首部长度
    unsigned char thl:4; //tcp头部长度
    unsigned char flag:6; //6位标志
    unsigned char reseverd_2:2; //保留6位中的2位
#else
    unsigned char thl:4; //tcp头部长度
    unsigned char reserved_1:4; //保留6位中的4位首部长度
    unsigned char reseverd_2:2; //保留6位中的2位
    unsigned char flag:6; //6位标志
#endif
    unsigned short wnd_size; //16位窗口大小
    unsigned short chk_sum; //16位TCP检验和
    unsigned short urgt_p; //16为紧急指针
}tcp_hdr;
```

![TCP 协议数据包](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202112/20211225154006992.jpg)

### 2.6 Backwards Compatibility

消息格式必须能够向后兼容，即旧系统应该能够读取同一消息的较新版本，反正亦然。

SBE 中设计了一种扩展机制，允许在数据中引入新的可选字段，新系统可以使用这些字段，而旧系统在升级之前会忽略这些字段。如果需要更改必填字段或基本结构，则必须使用新的消息类型，因为这已不再是现有数据类型的语义扩展。

## 三、使用教程

### 3.1 XML Grammar

首先我们得学会如何定义一个 SBE 的传输协议，它不同于 protobuf 使用自定义格式，SBE 采用 XML 格式。具体还是得参考[官方文档](https://github.com/real-logic/simple-binary-encoding/wiki/FIX-SBE-XML-Primer)，这里我就不做 CV 侠了。

### 3.2 Maven Plugins

当我们准备好了 SBE 的 XML 文件后，下一步就是需要根据该文件生成对应的编解码器代码，官方推荐的是使用 `sbe-all-${SBE_LIB_VERSION}.jar` 工具包，然后调用 `java -jar` 的方式生成代码，具体的流程以及详细的参数见[官方文档](https://github.com/real-logic/simple-binary-encoding/wiki/Sbe-Tool-Guide)。

这里我介绍下另一种通过 Maven 插件的方式生成代码，用起来更为方便。首先假设我将 XML 文件存放在项目的 `resources` 目录下，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202112/20211225162603625.png)

然后在 Maven 中添加插件：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <version>1.6.0</version>
            <executions>
                <execution>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>java</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <includeProjectDependencies>false</includeProjectDependencies>
                <includePluginDependencies>true</includePluginDependencies>
                <mainClass>uk.co.real_logic.sbe.SbeTool</mainClass>
                <systemProperties>
                    <systemProperty>
                        <key>sbe.output.dir</key>
                        <value>${project.build.directory}/generated-sources/java</value>
                    </systemProperty>
                    <!-- Is XInclude supported for the schema -->
                    <systemProperty>
                        <key>sbe.xinclude.aware</key>
                        <value>true</value>
                    </systemProperty>
                </systemProperties>
                <arguments>
                    <argument>${project.build.resources[0].directory}/sbe-example1.xml</argument>
                    <argument>${project.build.resources[0].directory}/sbe-example2.xml</argument>
                </arguments>
                <workingDirectory>${project.build.directory}/generated-sources/java</workingDirectory>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>uk.co.real-logic</groupId>
                    <artifactId>sbe-tool</artifactId>
                    <version>1.24.0</version>
                </dependency>
            </dependencies>
        </plugin>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
            <version>3.0.0</version>
            <executions>
                <execution>
                    <id>add-source</id>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>add-source</goal>
                    </goals>
                    <configuration>
                        <sources>
                            <source>${project.build.directory}/generated-sources/java/</source>
                        </sources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

然后执行 `maven clean install` 命令后自动生成代码会放置在 `target/generated-sources/java` 目录下。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202112/20211225162716623.png)

注意：如果需要调整 XML 文件位置，或者生成文件的位置，或者生成参数，请自行调整上文插件中的具体参数。

### 3.3 编解码

下一步就是利用生成的编解码类，进行数据传输了，难点就是对它 API 的使用了，这里我给出两个我学习时候用的例子大家在本地对照文档进行 Debug，很快就明白了。



第一个例子来源于 Aerona Cookbook：

- 例子介绍：https://aeroncookbook.com/simple-binary-encoding/basic-sample/
- 对应源码：com.github.jitwxs.sample.aeron.sbe1.SbeExample1Test

第二个例子来源于 SBE GitHub:

- 例子介绍：https://github.com/real-logic/simple-binary-encoding/wiki/Java-Users-Guide
- 对应源码：com.github.jitwxs.sample.aeron.sbe2.ExampleUsingGeneratedStub


