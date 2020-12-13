---
title: Protobuf 学习手册——编码篇
typora-root-url: ..
categories:
  - Java
  - Protobuf
tags: Protobuf
abbrlink: d4225e9b
katex: true
date: 2019-12-26 00:01:54
copyright_author: Jitwxs
---

## 一、编码规范

Google 官方提供了 Protobuf 的编码规范，通过遵循这些规范，可以使 Protobuf 消息定义及其相应的类保持一致并易于阅读。

Protobuf 编码规范可能随着时间推移而发生变化，对于既有项目，应当保持编码规范的一致性，而不需盲目保持最新的编码规范。但是对于全新项目，应当遵循官方的编码规范，可以[点击这里](https://developers.google.com/protocol-buffers/docs/style)查阅官方最新的编码规范。

### 1.1 Override

- 一行不超过 80 个字符
- 两个空格缩进

### 1.2 文件结构

文件名采用下划线分割命名，形如： `lower_snake_case.proto`。所有 .proto 文件应当遵循以下规范：

1. License 头（如果需要的话）
2. 文件综述
3. Syntax
4. Package
5. Imports (排好序的)
6. File options
7. 一切其他的东西

### 1.3 Package

包名应当小写，且应当对应于目录结构。例如文件位于 `my/package/` 目录, 那么 package 应该是 `my.package`。

> 对于 Java 应用，我们更习惯于使用 `option java_package`，很少使用 package。

### 1.4 Message and field names

- 对于 messages，使用首字母大写的驼峰命名，例如`SongServerRequest`
- 对于参数名，使用下划线分割命名，例如 `song_name`

### 1.5 Repeated fields

对于可重复的字段用复数名：

```protobuf
repeated string keys = 1;
...
repeated MyMessage accounts = 17;
```

### 1.6 枚举

- 枚举名使用首字母大写的驼峰命名
- 枚举值，采用全大写、下划线命名

```protobuf
enum Foo {
  FOO_UNSPECIFIED = 0;
  FOO_FIRST_VALUE = 1;
  FOO_SECOND_VALUE = 2;
}
```

每个枚举值使用分号分割，推荐给枚举值添加前缀标识。编号为 0 的枚举值应当包含 ` UNSPECIFIED ` 后缀。

### 1.7 Service

如果在 Protobuf 中定义了 RPC 服务，使用首字母大写的驼峰命名 Service 及其每一个方法。

```protobuf
service FooService {
  rpc GetSomething(FooRequest) returns (FooResponse);
}
```

## 二、编码原理

本章节将介绍 Protobuf Message 的编码原理，在实际应用中不需要关注这些，但是对于了解 Protobuf 如何编码的及生成 Message 的大小非常有用。

首先让我们先看一个非常简单的 Message 定义：

```protobuf
message Test1 {
  optional int32 a = 1;
}
```

在使用过程中，创建了 Test1 这个 message，并设置 a 的值为 150。将其转化为二进制流并打印出来，你会得到：

```
08 96 01
```

如此的小！这是怎么做到的呢，让我们往下看。

### 2.1 Base 128 Varints 编码

为了了解 Protobuf 编码，首先先了解下 `Varints` 编码。Varint 是一种使用一个或多个字节序列化整数的方法，也可以说是一种**压缩算法**，值越小的数字使用越少的字节数。压缩的依据是：**越小的数字，越经常使用。**

以上面 150 这个数值为例，使用正常 32 位整型（二进制）传输，那么编码内容为：

```
0000 0000 0000 0000 0000 0000 1001 0110
```

可以看到使用传统编码方式，对于大部分数，用不完 32 位，导致有大量的 0，存在空间浪费问题。

#### 2.1.1 编码

Varints 的编码规则如下【注：**大端字节序下**】：

1. 将数值转换为二进制，从最低位开始，自右至左每 7 位作为一组进行分割。
2. 翻转组。
3. 在每一组最前面插入一位`最高有效位`（msb）[^1]，凑成一个字节（8 位）。最后一组插入 0，表示后面没有字节出现；其他组插入 1 ，表示后面还有字节出现。
4. 此时每一组都有 8 位，即一组就是一个字节，将结果转换为十六进制输出。

以 150 为例，首先转换为二进制：

```
1001 0110
```

7 位一组进行分割：

```
000 0001, 001 0110
```

翻转组：

```
001 0110, 000 0001
```

每一组最前面插入 `msb`，除最后一组插入 0 外，其余组插入 1：

```
1001 0110, 0000 0001
```

转换为十六进制表示：

```
96, 01
```

#### 2.1.2 解码

Varints 的解码就是对编码的逆操作，以 150 的编码结果进行解码为例：

1. 将编码后数据（十六进制）转换为二进制
2. 去除每个字节最高位的 `msb`
3. 翻转，然后转换为 10 进制输出

以 96, 01 为例，首先转换为二进制：

```
1001 0110, 0000 0001
```

去除每个字节最高位的 `msb`：

```
001 0110, 000 0001
```

翻转：

```
000 0001, 001 0110
```

转换回十进制：

```
128 + 16 +4 + 2 = 150
```

#### 2.1.3 存储

一个字节的 Varints 编码有 7 位可以存储数据（最高位为 msb），则可以传输 $[0,2^7-1]$。以此类推，两个字节就是 $[2^7, 2^{14}-1]$。

#### 2.1.4 小结

读到这里可能有读者会问了，Varints 不是为了紧凑 int 的么？那 150 本来可以用 1 个字节表示，现在却是 2 个字节了，哪里紧凑了，还多花费了空间！

Varints 确实是一种紧凑的表示数字的方法。它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数。这能减少用来表示数字的字节数。比如对于 int32 类型的数字，一般需要 4 个字节来表示。但是采用 Varints，对于很小的 int32 类型的数字，则可以用 1 个字节来表示。当然凡事都有好的也有不好的一面，采用 Varints 表示法，大的数字则需要 5 个字节来表示。从统计的角度来说，一般不会所有的消息中的数字都是大数，因此大多数情况下，采用 Varints 后，可以用更少的字节数来表示数字信息。

150 如果用 int32 表示，需要 4 个字节，现在用 Varints 表示，只需要 2 个字节了。缩小了一半！

### 2.2 Message 结构

我们知道，Protobuf message 是一系列的键值对。在进行序列化时，对于 key 只序列化它的类型和编号；反序列化时，只有根据 .proto 文件才能确定某个 key 具体的含义是什么。这一点也是 Protobuf 比 JSON、XML 更安全的原因，如果没有数据结构描述 `.proto` 文件，拿到数据以后也是无法解析成正常的数据的。

message 进行编码时， key 和 value 被串联成一个字节流。对 message 进行解码时，解析器需要能够跳过无法识别的字段。这样，可以将新字段添加到 message 中，而不会破坏不知道它们的旧程序。因此，key 被序列化时包含**类型（wire_type）和编号（field's num）**。

key 的计算方法是 `(field_number << 3) | wire_type `，换句话说，key 的最后 3 位表示的就是 `wire_type`。Protobuf 支持的字段类型见下表：

| Type  | Meaning          | Used For                                                 |
| ----- | ---------------- | -------------------------------------------------------- |
| 0     | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1     | 64-bit           | fixed64, sfixed64, double                                |
| 2     | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| ~~3~~ | ~~Start group~~  | ~~groups (deprecated)~~                                  |
| ~~4~~ | ~~End group~~    | ~~groups (deprecated)~~                                  |
| 5     | 32-bit           | fixed32, sfixed32, float                                 |

举个例子，还是以 Test1 这个 message 为例。其二进制输出流为：

```
08 96 01
```

后两个字节我们已经解释过了，就是 150 的 Varints 编码，第一个字节 08 就是 150 所对应的 key。

将其转换为二进制：

```
0000 1000
```

去除 msb 位：

```
000 1000
```

根据 `(field_number << 3) | wire_type `，最后三位表示字段类型：

```
000 --> 0 --> 查阅类型表格：Varint
```

右移三位，得到字段编号：

```
0001 ---> 1 ---> 1在Test1中是: a
```

![](/images/posts/20191226000506791.jpg)

### 2.3 更多值类型

#### 2.3.1 有符号整型

根据上面的字段类型表格所示，所有 type = 0 关联的 Protobuf 类型被称为 `Varint`。

当数值为负数时，有符号整型（sint32, sint64）和标准整型（int32, int64）有一个重要的差别。如果使用 int32 或 int64 存储负数，那么 Varints 编码后的结果一定是 10 个字节（实际上，它被视为一个非常大的无符号整数）。而如果使用 sint32 或 sint64 存储负数，则会使用效率更高的 `ZigZag` 编码。

>**10 个字节是怎么计算出来的呢？**
>
>首先对于负数，即使使用 int32 进行存储进行 Varints 编码时也会转成 int64。至于为什么要这么规定呢，猜想可能是怕 32 位的负数转换会有溢出的可能(只是猜想)。
>
>又因为计算机定义负数的符号位为数字的最高位，以 -1 为例，其二进制在 int64 下为：
>
>10000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
>
>进行 7 位分组：
>
>00000001 0000000 0000000 0000000 0000000 0000000 0000000 0000000 0000000 0000001
>
>可以看到，即使是 -1，分组后被都被分成了 10 组，再添加上 msr，就是 10 个字节了。
>
>**这也是为什么不推荐使用 int32 或 int64 存储负数的原因。**

为此 Protobuf 定义了 sint32 和 sint64 这种类型，采用 ZigZag 编码。**将所有整数映射成无符号整数，然后再采用 Varints 编码方式编码**，这样绝对值小的整数，编码后也会有一个较小的 varint 编码值。ZigZag 映射函数为：

$$
ZigZag(n) = (n << 1) \bigoplus (n >> 31), n \in sint32
$$

$$
ZigZag(n) = (n << 1) \bigoplus (n >> 63), n \in sint64
$$

按照这种方法，会在正数和负数整型之间来回摇摆。-1 将会被编码成 1，1 将会被编码成 2，-2 会被编码成 3，如下表所示：

| 原始有符号整型 | 编码结果   |
| -------------- | ---------- |
| 0              | 0          |
| -1             | 1          |
| 1              | 2          |
| 2              | 3          |
| 2147483647     | 4294967294 |
| -2147483648    | 4294967295 |

需要注意的是，第二个转换 `（n >> 31）` 部分，是一个算术转换。换句话说，移位的结果要么是一个全为 0（如果 n 是正数），要么是全部 1（如果 n 是负数）。

当 sint32 或 sint64 被解析时，它的值被解码回原始的带符号的版本。

#### 2.3.2 Non-varint 数字

Non-varint 数字比较简单，double 、fixed64 的 wire_type 为 1，在解析时告诉解析器，该类型的数据需要一个 64 位大小的数据块即可。同理，float 和 fixed32 的 wire_type 为 5，给其 32 位数据块即可。在这两种情况下，值均以**小端字节顺序**存储。

【注：说 Protobuf 压缩数据没有到极限，就因为并没有压缩 float、double 这些浮点类型。】

#### 2.3.3 字符串

wire_type 类型为 2 的数据，是一种指定长度的编码方式：`key + length + content`，key 的编码方式是统一的，length 采用 Varints 编码方式，content 就是由 length 指定长度的字节。

举例，假设定义如下的 message 格式：

```protobuf
message Test2 {
  optional string b = 2;
}
```

设置 b 的值为"testing"，转换为二进制流打印出来：

```
12 07 74 65 73 74 69 6e 67
```

- "74 65 73 74 69 6e 67" 是“testing”的 UTF8 代码。

- "12" 是 key 的 Varints 编码，对其进行解码，得到 wire_type 为 2，字段编号为 2。

- "07" 表示 length 为 7即后面跟着 7 个字节。

### 2.4 Embedded Messages

假设，定义如下嵌套消息：

```protobuf
message Test3 {
  optional Test1 c = 3;
}
```

设置字段值为整数 150，编码后的字节为：

```
1a 03 08 96 01
```

"08 96 01" 这三个代表的是 150，上面讲解过，这里就不再赘述了。

"1a" 解码后得到  wire type 为 2，字段编号为 3。

"1a" 表示 length 为 3，代表后面有 3 个字节 。

因此， Embedded Messages 处理方式和字符串完全相同。

### 2.5 可选和重复元素

在 proto2 中定义成 repeated 的字段，（没有加上 [packed=true] 选项 ），编码后的 message 有一个或者多个包含相同编号的 key-value 对。这些重复的 value 不需要连续的出现。他们可能与其他的字段交错出现。尽管它们是无序的，但是在解析时它们是需要有序的。在 proto3 中 repeated 字段默认采用 packed 编码。

对于 proto3 中的任何非 repeated 字段或 proto2 中的 optional 字段，编码的 message 可能有也可能没有包含该字段号的键值对。

通常, 编码好的 message 绝不会有一个可选或必需字段的多个实例。但是，解析器被期望能处理这种情况。对数值和字符串，如果相同的值出现多次，解析器接受它看到的最后一个值。对于 Embedded 字段，解析器合并它接收到的同一字段的多个实例。就如 `Message::MergeFrom` 方法一样。这样的规则的结果是，**解析连接在一起的两个编码后的 message 和分别解析两个 message 然后合并得到的对象，是一致的**。例如：

```c++
MyMessage message;
message.ParseFromString(str1 + str2);
```

等价于：

```c++
MyMessage message, message2;
message.ParseFromString(str1);
message2.ParseFromString(str2);
message.MergeFrom(message2);
```

这种方法有时是非常有用的。比如，即使不知道 message 的类型，也能够将其合并。

### 2.6 Packed Repeated 字段

在 2.1.0 版本以后，Protobuf 引入了该种类型，其与 repeated 字段一样，只是在末尾声明了 `[packed=true]`。类似 repeated 字段却又不同。在 proto3 中 Repeated 字段默认就是以这种方式处理。对于 packed repeated 字段，如果 message 中没有赋值，则不会出现在编码后的数据中。否则的话，该字段所有的元素会被打包到单一一个 key-value 对中，且它的 wire_type=2，长度确定。每个元素正常编码，只不过前面没有字段序号。例如有如下 message 类型：

```protobuf
message Test4 {
  repeated int32 d = 4 [packed=true];
}
```

构造一个 Test4 字段，并且设置 repeated 字段 d 三个值：3、270 和 86942，编码后：

```
22        // key (field number 4, wire type 2)
06        // payload size (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```

只有原始数字类型（使用varint，32位或64位）的重复字段才可以声明为“packed”。

有一点需要注意，对于 packed 的 repeated 字段，尽管通常没有理由将其编码为多个 key-value 对，编码器必须有接收多个 key-pair 对的准备。这种情况下，payload 必须是串联的，每个 pair 必须包含完整的元素。

Protocol Buffer 解析器必须能够解析被重新编译为 packed 的字段，就像它们未被 packed 一样，反之亦然。这允许以正向和反向兼容的方式将 [packed = true] 添加到现有字段。

### 2.7 字段顺序

字段编号可以在 .proto 文件中以任何顺序使用，编码 / 解码与字段顺序无关，这一点由 key-value 机制保证。

序列化 message 时，对于如何写入其已知字段或未知字段没有保证的顺序。序列化顺序是一个实现细节，将来任何特定实现的细节都可能更改。因此，Protobuf 解析器必须能够以任何顺序解析字段。

### 2.8 例子

假设存在 message 定义如下：

```protobuf
syntax = "proto3";

option java_outer_classname = "PersonProto";

message Person {
    string user_name = 1;
    int64 favourite_number = 2;
    repeated string interests = 3;
}
```

实例化该 message 并输出字节数组：

```java
public class ProtoBufTest1 {
    public static void main(String[] args) {
        PersonProto.Person person = PersonProto.Person.newBuilder()
                .setUserName("Martin")
                .setFavouriteNumber(1337)
                .addInterests("daydreaming")
                .addInterests("hacking").build();

        System.out.println(bytesToHex(person.toByteArray()));
    }

    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte aByte : bytes) {
            String hex = Integer.toHexString(aByte & 0xFF);
            if (hex.length() < 2) {
                sb.append(StringUtils.leftPad(hex, 2, "0"));
            } else {
                sb.append(hex);
            }
            sb.append(" ");
        }
        return sb.toString();
    }
}
```

运行结果：

```
0a 06 4d 61 72 74 69 6e 10 b9 0a 1a 0b 64 61 79 64 72 65 61 6d 69 6e 67 1a 07 68 61 63 6b 69 6e 67
```

每个字节解码详细解释见下图。

![](/images/posts/20191226000356778.png)

## 三、技巧

这里介绍下在 Protobuf 中被广泛使用的技巧，或是设计模式。你也可以在 [Protocol Buffers discussion group](http://groups.google.com/group/protobuf) 发表设计与使用相关问题。

### 3.1 流化多个 Message

如果想要将多个 message 写入到单个文件或者流中，你需要自己跟踪 message 开始和结束的位置。Protobuf wire 格式不是 self-delimiting 的，因此 Protobuf 解析器不能自己检测 message 的结束。

解决这个问题最简单的方法是在写入 message 自身之外写入每个 message 的大小。当反序列化回消息时，先读取大小，然后根据大小分割字节 buffer，再从 buffer 进行解析。（如果想要避免复制字节的消耗，可以使用 C++ 和 Java 中的 `CodedInputStream`）。

### 3.2 大数据集

Protobuf 不适合处理大 message，根据经验如果要处理的 message 均大于 1MB，是时候考虑一个替代策略了。

也就是说，Protobuf 非常适合处理大型数据集中的单个 message。 通常大数据集实际上只是小片段的集合， 每个小片段可能是结构化的数据片段。即使 Protobuf 无法一次处理整个集合，但是使用 Protobuf 对每个片段进行编码可极大简化问题：现在需要处理的是字节字符串的集合而不是结构体的集合。 

Protobuf 不包含对大型数据集的任何内置支持，因为不同的情况要求使用不同的解决方案。有时只需要一个简单的记录列表，而其他时候可能希望更像一个数据库。每个解决方案都应作为独立的库进行开发，这样只有那些需要它的人才需要付出代价。  

### 3.3 自描述 Messages

Protobuf 不包含其自身类型的描述。因此仅给出原始消息，而没有相应的 .proto 文件定义其类型，很难提取任何有用的数据。

但是，.proto 文件的内容本身可以使用 Protobuf  表示。`src/google/protobuf/descriptor.proto ` 定义了所涉及的 message 类型。. protoc 可以通过使用 `--descriptor_set_out` 选项输出一个 `FileDescriptorSet`。 FileDescriptorSet 展现了多个 .proto 文件的集合， 你可以像这样定义一个自描述 Protobuf message。

```protobuf
syntax = "proto3";

import "google/protobuf/any.proto";
import "google/protobuf/descriptor.proto";

message SelfDescribingMessage {
  // Set of FileDescriptorProtos which describe the type and its dependencies.
  google.protobuf.FileDescriptorSet descriptor_set = 1;

  // The message and its type, encoded as an Any message.
  google.protobuf.Any message = 2;
}
```

在 C++ 或 Java 中，通过使用类似 `DynamicMessage` 的类，可以编写工具来操作 SelfDescribingMessages。话虽如此，Protobuf  库中未包含此功能的原因是因为从未在 Google 内部中使用过它。

此技术需要使用 descriptors 支持动态 message，在使用自描述 message 之前，请检查您的平台是否支持此功能。

## 四、总结

相信大家经过[《Protobuf 学习手册——语法篇》](/60aca815.html)和[《Protobuf 学习手册——编码篇》](/d4225e9b.html)这两篇文章的学习，对 protobuf 已经有了一定的了解了，下面让我们总结下。

1. Protobuf 利用 varint 原理压缩数据以后，二进制数据非常紧凑，option 也算是压缩体积的一个举措。所以 pb 体积更小，如果选用它作为网络数据传输，势必相同数据，消耗的网络流量更少。但是并没有压缩到极限，float、double 浮点型都没有压缩。
2. Protobuf 比 JSON 和 XML 少了 {、}、: 这些符号，体积也减少一些。再加上 varint 压缩，gzip 压缩以后体积更小！
3. Protobuf 是 Tag - Value (Tag - Length - Value)的编码方式的实现，减少了分隔符的使用，数据存储更加紧凑。
4. Protobuf 另外一个核心价值在于提供了一套工具，一个编译工具，自动化生成 get/set 代码。简化了多语言交互的复杂度，使得编码解码工作有了生产力。
5. Protobuf 不是自我描述的，离开了数据描述 `.proto` 文件，就无法理解二进制数据流。这点即是优点，使数据具有一定的“加密性”，也是缺点，数据可读性极差。所以 Protobuf 非常适合内部服务之间 RPC 调用和传递数据。
6. Protobuf 具有向后兼容的特性，更新数据结构以后，老版本依旧可以兼容，这也是 Protobuf 诞生之初被寄予解决的问题。因为编译器对不识别的新增字段会跳过不处理。

[^1]: 最高有效位，most significant bit
