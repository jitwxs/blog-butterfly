---
title: Protobuf 学习手册——语法篇
categories:
  - Java
  - Protobuf
tags: Protobuf
abbrlink: 60aca815
date: 2019-12-24 22:58:39
references:
  - name: 高效的数据压缩编码方式 Protobuf
    url: https://halfrost.com/protobuf_encode
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Language Guide (proto3)
    url: https://developers.google.com/protocol-buffers/docs/proto3
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Protocol Buffer 3学习笔记
    url:  https://www.cntofu.com/book/116/index.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Protocol Buffers 学习（6）：文件 | 字段选项介绍
    url: https://juejin.im/post/59361f79a22b9d0058fdc725
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、Override

`Protobuf`[^1] 是一种语言中立、平台无关、可扩展的**序列化数据的格式**，可用于通信协议，数据存储等。

ProtoBuf 在序列化数据方面，它是灵活的、高效的。相比于 XML 来说，ProtoBuf 更加小巧、更加快速、更加简单。一旦定义了要处理的数据的数据结构之后，就可以利用 ProtoBuf 的代码生成工具生成相关的代码。甚至可以在无需重新部署程序的情况下更新数据结构。只需使用 ProtoBuf 对数据结构进行一次描述，即可利用各种不同语言或从各种不同数据流中对你的结构化数据轻松读写。

**ProtoBuf 很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式**。

![protocol buffers logo](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20191224230003235.png)

### 1.1 为什么要发明 Protobuf

大家可能会觉得 Google 发明 ProtoBuf 是为了解决序列化速度的，其实真实的原因并不是这样的。

ProtoBuf 最先开始是 Google 用来解决索引服务器 request/response 协议的。没有 ProtoBuf 之前，google 已经存在了一种 request/response 格式，用于手动处理 request/response 的编组和反编组。它也能支持多版本协议，不过代码比较丑陋：

```c
 if (version == 3) {
   ...
 } else if (version > 4) {
   if (version == 5) {
     ...
   }
   ...
 }
```

如果非常明确的格式化协议，就会使新协议变得非常复杂。因为开发人员必须确保请求发起者与处理请求的实际服务器之间的所有服务器都能理解新协议，然后才能切换开关以开始使用新协议。这也就是每个服务器开发人员都遇到过的低版本兼容、新旧协议兼容相关的问题。

为了解决这些问题，ProtoBuf 诞生了，它具有以下两个特点：

- 可以很容易地引入新的字段，并且不需要检查数据的中间服务器可以简单地解析并传递数据，而无需了解所有字段。
- 数据格式更加具有自我描述性，可以用各种语言来处理(C++, Java 等各种语言)

最初版本的 ProtoBuf 仍需要自己手写解析的代码，不过随着系统慢慢发展演进，ProtoBuf 目前具有了更多的特性：

- 自动生成序列化和反序列化代码避免了手动解析的需要（官方提供自动生成代码工具，各个语言平台的基本都有）。
- 除了用于 RPC 请求之外，人们开始将 ProtoBuf 用作持久存储数据的便捷自描述格式。
- 服务器的 RPC 接口可以先声明为协议的一部分，然后用 protocol compiler 生成基类，用户可以使用服务器接口的实际实现来覆盖它们。

ProtoBuf 现在是 Google 用于数据的通用语言，它们既用于 RPC 系统，也用于在各种存储系统中持久存储数据。

**总结：** ProtoBuf 诞生之初是为了解决服务器端新旧协议(高低版本)兼容性问题，只不过后期慢慢发展成用于数据传输。

### 1.2 命名由来

>The name originates from the early days of the format, before we had the protocol buffer compiler to generate classes for us. At the time, there was a class called ProtocolBuffer which actually acted as a buffer for an individual method. Users would add tag/value pairs to this buffer individually by calling methods like AddValue(tag, value). The raw bytes were stored in a buffer which could then be written out once the message had been constructed.
>
>Since that time, the "buffers" part of the name has lost its meaning, but it is still the name we use. Today, people usually use the term "protocol message" to refer to a message in an abstract sense, "protocol buffer" to refer to a serialized copy of a message, and "protocol message object" to refer to an in-memory object representing the parsed message.

Protocol buffer 这个名字起源于 format 早期，在我们有 protocol buffer 编译器为我们生成类之前。当时，有一个名为 ProtocolBuffer 的类，它实际上充当了单个方法的缓冲区。用户可以通过调用像 AddValue(tag, value) 这样的方法分别将标签/值对添加到此缓冲区。原始字节存储在一个缓冲区中，一旦构建消息就可以将其写出。

从那时起，名为“缓冲”的部分已经失去了意义，但它仍然是我们使用的名称。今天，人们通常使用术语“protocol message”来指代抽象意义上的消息，“protocol buffer”指的是消息的序列化副本，而“protocol message object”指的是代表内存中对象解析的消息。

### 1.3 Proto2 or Proto3

Protobuf 存在 Proto2 和 Proto3 两个版本，和 Python 一样，Proto3 是今后的主要版本，Proto2 除了用于历史遗留项目外，不被推荐使用。因此对于初学者来说，选择 Proto3 进行学习和使用是毋庸置疑的。截止到本文编写时最新的 Relaese 版本是 **3.11.0**。

你可能会疑惑为什么没有 Proto1，这是因为 Google 起初开源 Protobuf 时，在 Google 内部实际上已经是第二个版本了。为了保持版本的统一，索性开源版本就从 v2.0.0 开始了。

Proto3 不完全兼容 Proto2，下面列出我发现的不同之处，后续我会持续在这里补充。

- 首行必须是 `syntax = "proto3";` 用于标识使用 Protobuf3 解析。
- 移除了 `required`  关键字，因为它破坏了 Protobuf 向前/后兼容的特点。
- 移除了 `optional` 关键字，如果字段不赋值，那么就是使用字段类型的默认值。Protobuf3 在字段被设置为默认值的时候，不会序列化该字段。这样可以节省空间，提高效率。
- 移除了 `default` 关键字，不再允许手动指定某个字段的默认值，只能使用字段类型的默认值。这可能会导致 Protobuf2 和 Protubuf3 的不兼容，例如 Protobuf2 中手动指定了字段的默认值，在 Protobuf3 中就会忽略该字段。
- 枚举类型第一个字段编号必须为 `0`，它是该枚举的默认值。

### 1.4 对比 XML

Protobuf 在序列化方面，与 XML 相比，有诸多优点：

- 更加简单
- 数据体积小 3 - 10 倍
- 更快的反序列化速度，提高 20 - 100 倍
- 可以自动化生成更易于编码方式使用的数据访问类

举个例子，如果要编码一个用户的名字和 email 信息，用 XML 的方式如下：

```xml
<person>
  <name>John Doe</name>
  <email>jdoe@example.com</email>
</person>
```

相同需求，如果换成 Protobuf 来实现，定义文件如下：

```protobuf
# Textual representation of a protocol buffer. This is not the binary format used on the wire.
person {
  name: "John Doe"
  email: "jdoe@example.com"
}
```

Protobuf 通过编码以后，以二进制的方式进行数据传输，最多只需要 28 bytes 空间和 100-200 ns 的反序列化时间。但是 XML 则至少需要 69 bytes 空间（经过压缩以后，去掉所有空格）和 5000-10000 的反序列化时间。

上面说的是性能方面的优势。接下来说说编码方面的优势。

Protobuf 自带代码生成工具，可以生成友好的数据访问存储接口。从而开发人员使用它来编码更加方便。例如上面的例子，如果用 C++ 的方式去读取用户的名字和 email，直接调用对应的 get 方法即可。

```c++
cout << "Name: " << person.name() << endl;
cout << "E-mail: " << person.email() << endl;
```

而 XML 读取数据会麻烦一些：

```c++
cout << "Name: "
     << person.getElementsByTagName("name")->item(0)->innerText()
     << endl;
cout << "E-mail: "
     << person.getElementsByTagName("email")->item(0)->innerText()
     << endl;
```

Protobuf 语义更清晰，无需类似 XML 解析器的东西（因为 Protobuf 编译器会将 .proto 文件编译生成对应的数据访问类以对 Protobuf 数据进行序列化、反序列化操作）。

使用 Protobuf 无需学习复杂的文档对象模型，Protobuf 的编程模式比较友好，简单易学，同时它拥有良好的文档和示例，对于喜欢简单事物的人们而言，Protobuf 比其他的技术更加有吸引力。

Protobuf 最后一个非常棒的特性是，即“向后”兼容性好，人们不必破坏已部署的、依靠“老”数据格式的程序就可以对数据结构进行升级。这样您的程序就可以不必担心因为消息结构的改变而造成的大规模的代码重构或者迁移的问题。因为添加新的消息中的 field 并不会引起已经发布的程序的任何改变(因为存储方式本来就是无序的，k-v 形式)。

当然 Protobuf 也并不是完美的，在使用上存在一些局限性。

由于文本并不适合用来描述数据结构，所以 Protobuf 也不适合用来对基于文本的标记文档（如 HTML）建模。另外，由于 XML 具有某种程度上的自解释性，它可以被人直接读取编辑，在这一点上 Protobuf 不行，它以二进制的方式存储，除非你有 `.proto` 定义，否则你没法直接读出 Protobuf 的任何内容。

### 1.5 学习指南

- Protobuf 官方文档：https://developers.google.com/protocol-buffers/docs/overview

- Protobuf For Java 指南：https://developers.google.com/protocol-buffers/docs/javatutorial

- Protobuf GitHub：https://github.com/protocolbuffers/protobuf

## 二、Message 定义

在 Protobuf 中，所有结构化的数据都被称为 `message`。假设你想要查询某个接口，这个接口需要传递这些参数关键字和分页参数（当前页和每页记录数），那么我们就可以把这些参数都定义成一个对象，用 Protobuf 的话说就是定义一个 message。

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

- `syntax = "proto3";` 指定了整个 .proto 文件使用 Protobuf3 解析，否则默认会使用 Protobuf2 解析，必须将其放置在文件的第一行。
- SearchRequest 这个 message 指定了三个数据字段，每个数据字段定义由 **Field Types 数据类型 + Field Names 字段名 = Field Numbers 编号**组成。

### 2.1 字段类型

#### 2.1.1  基础类型

Protobuf 支持 Java、C++、Python、Go、Ruby、C#、PHP 等主流编程语言。限于博主是 Java 开发，因此只列出 Protobuf 数据类型和 Java 语言的类型的对应关系，其他语言见官方[完整表格](https://developers.google.com/protocol-buffers/docs/proto3#scalar)。

| Protobuf 类型 | 备注                                                         | Java 类型  |
| :------------ | :----------------------------------------------------------- | :--------- |
| double        |                                                              | double     |
| float         |                                                              | float      |
| int32         | 使用可变长度编码。对负数编码效率低下，如果您的字段可能有负值，则使用sint 32代替。 | int        |
| int64         | 使用可变长度编码。对负数编码效率低下，如果您的字段可能有负值，则使用sint 64代替。 | long       |
| uint32        | 使用可变长度编码。                                           | int[^2]    |
| uint64        | 使用可变长度编码。                                           | long[^2]   |
| sint32        | 使用可变长度编码。有符号的 int 型，比 int32 更适合处理负数。 | int        |
| sint64        | 使用可变长度编码。有符号的 int 型，比 int36 更适合处理负数。 | long       |
| fixed32       | 总是四个字节，如果值通常大于228，则比 uint 32 更有效。       | int[^2]    |
| fixed64       | 总是八个字节，如果值通常大于256，则比 uint 64 更有效。       | long[^2]   |
| sfixed32      | 总是四个字节                                                 | int        |
| sfixed64      | 总是八个字节                                                 | long       |
| bool          |                                                              | boolean    |
| string        | 字符串必须始终包含 UTF-8 编码或 7 位 ASCII 文本，不长于 232  | String     |
| bytes         | 可以包含不长于232的任意任意序列的字节                        | ByteString |

除了支持以上这些基础数据类型外，Protobuf 还支持枚举和 message 类型。

#### 2.1.2 枚举类型

Protobuf 支持枚举类型，还是以 SearchRequest 例子为例，假设想要加一个参数，用于指定查询的数据来源，例如可以从本地数据查询，也可以从历史数据查询。

```protobuf
syntax = "proto3";

message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
    SearchTypeEnum typeEnum = 4;
}

enum SearchTypeEnum {
    LOCAL = 0;
    HISTORY = 1;
}
```

需要注意的，在 Proto3 中，**枚举类型第一个元素的编号必须是 0，编号为 0 的枚举元素也这个枚举类的默认值**。这样做是因为：

- 强制枚举类包含一个编号为 0 的元素，这样枚举类型的默认值就可以定义为 0 了。
- 在 Proto2 中，第一个元素被认为是枚举类的默认值（除非显式指定），Proto3 这样做是为了兼容 Proto3。

你可以通过将相同的编号分配给不同的枚举常量来定义别名，这样做需要设置  `allow_alias`  为 true，否则编译会因为重复编号而报错。

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}
```

枚举类型序号必须在 32 位整数范围内。 由于枚举值在传输时使用 `varint` 编码，负值效率不高，因此不建议使用负值作为序号。 

在反序列化中，无法识别的枚举值将保留在 message 中， 因为消息反序列化时如何表示是依赖于语言的 。 在支持开放枚举类型且值超过指定符号范围的语言中，例如 C++ 和 Go，未知的枚举值只是存储为其基础整数表示。而在诸如 Java 之类的封闭枚举类型的语言中，枚举中的一个 case 会被用于表示未识别的值，使用特殊的访问器可以访问到底层数值。

在其他情况下，如果消息被序列化，则无法识别的值仍将与消息一起序列化。

#### 2.1.3 Message 类型

你可以使用其他 Message 类型作为字段【注：这有点类似于 Java 一个对象里面又有个对象字段】。如果想要在每个 SearchResponse 中包含 Result，可以像下面的例子一样将 Result 定义在同一个 .proto 文件中。

```protobuf
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

或者定义在其他 .proto 文件并使用 import 导入进来，关于 Import 的更多内容见 [2.8 节](#2-8-Import)。

```protobuf
import "myproject/other_protos.proto";
```

### 2.2 字段编号

每个消息定义中的每个字段都有**唯一的编号**。这些字段编号用于标识消息二进制格式中的字段，并且在使定义后不应更改。

需要注意的是，范围 1 ~ 15 中的字段编号需要**一个字节**进行编码，包括字段编号和字段类型。范围 16 ~ 2047 中的字段编号需要**两个字节**进行编码。所以应该将非常频繁出现的元素放置在序号 1 到 15 上，别忘了为将来可能添加的频繁出现的元素预留出几个编号。

可以指定的最小字段编号为 1，最大字段编号为 $2^{29}-1$ 或 536,870,911。也不能使用数字 19000 到 19999（`FieldDescriptor :: kFirstReservedNumber` 到 `FieldDescriptor :: kLastReservedNumber`，Protobuf 保留区间）。如果在 Protobuf 中使用这些保留数字中的任何一个，编译会报错。同样不能使用任何以前 Protobufs 保留的一些字段号码。保留字段见 [2.6 节](#2-6-保留字段)。

### 2.3 字段规则

- ` singular `:  格式正确的 message 可以包含 0 个或 1 个此字段，这是 Protobuf3 语法的默认字段规则。 
-  `repeated`:  格式正确的 message 中，此字段可以重复任意次（包括 0 次）， 重复值的顺序将保留。 

在 Protobuf3 中， `repeated` 标识的字段如果是基础数据类型，那么默认会使用  `packed`  编码方式，详细见[《Protobuf 学习手册——编码篇》](/d4225e9b.html)。

### 2.4 添加多个 Message

你可以在一个 .proto 文件中定义多个 message【注：就好似一个 Java 文件中可以定义多个 class 一样】。例如想要定义一个和 `SearchRequest` 相对应的响应 message，可以添加它到同一个 .proto 文件中：

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

### 2.5 注释

Protobuf 注释遵循 C / C++ 规范，你可以使用 `//` 或者 `/* ... */` 符号。

```protobuf
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

### 2.6 保留字段

如果您通过完全删除某个字段或将其注释掉来更新枚举，那么未来的用户在对该类型进修改时可以重新使用该字段编号。如果加载到了旧版本的 `.proto` 文件，就会导致系统出现严重问题，例如数据混乱、隐私错误等等。

确保这种情况不会发生的一种方法是指定删除字段的字段编号或名称（这也可能会导致 JSON 序列化问题）为 `reserved`。如果试图使用被 reserved 的字段编号或名称，Protobuf 编译器将会报错。 

```pro
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

**注意，不能在同一个 `reserved` 语句中混合字段名称和字段编号**，如有需要需要请像上面这个例子这样写。 

### 2.7 默认值

解析 message 时，如果编码的 message 不包含特定的  singular 元素，则该字段将设置为该字段的默认值，不同类型的默认值如下：

| 类型         | 默认值                                   |
|:------------ |:---------------------------------------- |
| 字符串       | 空串                                     |
| byte         | 空 bytes                                 |
| bool         | false                                    |
| 数值类型     | 0                                        |
| 枚举类型     | 第一个定义的枚举元素，其序号一定为 0     |
| message 类型 | 该字段未设置，它的确切值取决于依赖的语言 |

repeated  字段的默认值为空，通常是所依赖的语言中的空列表。另外如果将 message 字段显式设置为其默认值，则该值在传输时不会被序列化。

Note that for scalar message fields, once a message is parsed there's no way of telling whether a field was explicitly set to the default value (for example whether a boolean was set to `false`) or just not set at all: you should bear this in mind when defining your message types. For example, don't have a boolean that switches on some behaviour when set to `false` if you don't want that behaviour to also happen by default.  

### 2.8 Import

默认情况下，只能够使用导入的 .proto 文件中的内容。当你想要移动某个 .proto 文件的位置，那么就得手动修改所有关于该文件的 import 【注：如果 IDE 不能帮你智能修改的话，一个个改实在是麻烦，这一点在 Java 中也是一样】。

现在，除了修改所有 import 路径外，多了一个新的选择。就是在原始位置放置一个虚拟的 .proto 文件，在这个 .proto 中使用 `import public` 将所有的 import 转发到新的位置中，调用者就可以传递导入新的位置中的内容【注：这有点类似于 Linux 的软链接，使用该功能后，旧的 .proto 文件就相当于一个快捷方式，会链接到新的 .proto 文件】。例如：

```protobuf
// new.proto
// All definitions are moved here
```

```protobuf
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```

```protobuf
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

Protobuf 使用`-I` 或 `--proto_path` 参数在 Protobuf 命令行中指定的一组目录中搜索导入的文件。如果未指定参数，它将在调用编译器的目录中查找。通常，应将 `--proto_path` 标志设置为项目的根目录，并对所有导入使用完全限定的名称【注：如果使用 `protobuf-maven-plugin` 插件，那么默认扫描路径是：`src/main/proto`】。

## 三、Message 更新

如果发现以前定义 message 需要增加字段了，这个时候就体现出 Protobuf 的优势了——不需要改动之前的代码。不过需要满足以下规则：

- 不要改动原有字段的数据结构。

- 添加新字段，则任何由代码使用“旧”消息格式序列化的消息仍然可以通过新生成的代码进行分析。你应该记住这些元素的默认值，以便新代码可以正确地与旧代码生成的消息进行交互。同样，由新代码创建的消息可以由旧代码解析，旧的二进制文件在解析时会简单地忽略新字段。（具体原因见[第四章](#四、未知字段)）

- 只要字段在更新后的消息类型中不再使用，就可以删除。你可能会重命名，会添加前缀“OBSOLETE_”，或者标记成保留字段号 `reserved`，以便将来的 `.proto` 不会意外重复使用该字段号。

- int32，uint32，int64，uint64 和 bool 全都兼容。这意味着你可以将字段从这些类型之一更改为另一个字段而不破坏向前或向后兼容性。如果一个数字从不适合相应类型的线路中解析出来，则会得到与在 C++ 中将该数字转换为该类型相同的效果（例如，如果将 64 位数字读为 int32，它将被截断为 32 位）。

- sint32 和 sint64 相互兼容，但与其他整数类型不兼容。

- 只要字节是有效的UTF-8，string 和 bytes 是兼容的。

- 如果 bytes 包含 message 的 encoded version，则 Embedded message 与 bytes 兼容。

- fixed32 与 sfixed32 兼容，fixed64 与 sfixed64 兼容。

- enum 就数组而言，是可以与 int32，uint32，int64 和 uint64 兼容（请注意，如果它们不适合，值将被截断）。但是，当消息反序列化时，客户端代码可能会以不同的方式对待它们：例如，未识别的 proto3 枚举类型将保留在消息中，但消息反序列化时如何表示是与语言相关的。Int 域始终只保留它们的值。

- Changing a single value into a member of a **new** `oneof` is safe and binary compatible. Moving multiple fields into a new `oneof` may be safe if you are sure that no code sets more than one at a time. Moving any fields into an existing `oneof` is not safe.

## 四、未知字段

未知字段是格式正确的 Protobuf 序列化数据，表示解析器无法识别的字段。例如，当一个旧的二进制文件解析由新的二进制文件发送的新数据的数据时，这些新的字段将成为旧的二进制文件中的未知字段。

最初，Proto3 message 始终在解析过程中丢弃未知字段，但在 3.5 版本中，我们重新引入了保留未知字段以匹配 Proto2 行为的功能。在 3.5 版本和更高版本中，未知字段将在解析期间保留并包含在序列化输出中。

## 五、Any

`Any`  可以将 message 用作嵌入类型，而无需定义它们的 .proto。`Any` 包含任意序列化的 message（以字节为单位），以及用作该 message 的类型的全局唯一标识符的 URL。要使用 `Any` ，需要 import `google/protobuf/any.proto`。

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

给定 message 类型的默认类型 URL 为 `type.googleapis.com/packagename.messagename`。

不同的语言实现将支持运行时库帮助程序以类型安全的方式打包和解压缩任何值。例如，在 Java 中，Any 类型将具有特殊的`pack()` 和 `unpack()` 访问器，在 C++ 中有 `PackFrom()` 和 `UnpackTo()` 方法：

```c++
// Storing an arbitrary message type in Any.
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// Reading an arbitrary message from Any.
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is<NetworkErrorDetails>()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```

## 六、Oneof

如果某个 message 包含许多字段，但是最多可以同时设置一个字段，则可以使用 oneof 功能强制执行此行为并节省内存。

除了共享内存中的所有字段外，oneof 字段与常规字段类似，最多可以同时设置一个字段。设置 oneof 中的任何成员会自动清除所有其他字段。你可以使用 `case()` 或者 `WhichOneof()`方法来检查 oneof 中的哪个有值，具体取决于您选择的语言。

### 6.1 Using Oneof

要使用 oneof，只要 oneof 关键字加上 oneof 名称即可，在下面的例子中是 `test_oneof`：

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

然后添加字段到 oneof 定义中，你可以添加任何类型的字段，但不能使用重复的字段。在生成的代码中，oneof 字段具有与常规字段相同的 getter 和 setter。你还能通过一种特殊的方法来检查 oneof 中的哪个有值。

### 6.2 Oneof Features

- 设置 oneof 字段将自动清除 oneof 的所有其他字段。

  ```c++
  SampleMessage message;
  message.set_name("name");
  CHECK(message.has_name());
  message.mutable_sub_message();   // Will clear name field.
  CHECK(!message.has_name());
  ```

- 如果 Protobuf 解析器解析 oneof 时发现了多个有值的字段，则仅最后一个字段有效。

- oneof 字段不能是 `repeated`。

- oneof 字段可以使用反射 API。

- 如果将 oneof 字段设置为 message 默认值（例如设置 int32 oneof 字段为 0），the "case" of that oneof field will be set, and the value will be serialized on the wire.

- 如果你使用的是 C++，请确保你的代码不会导致内存崩溃。以下示例代码将崩溃，因为通过调用 set_name() 方法时已经删除了 sub_message。

  ```c++
  SampleMessage message;
  SubMessage* sub_message = message.mutable_sub_message();
  message.set_name("name");      // Will delete sub_message
  sub_message->set_...            // Crashes here
  ```

- 还是在 C++，如果使用 `Swap()` 了两个有 oneof 字段的 message，each message will end up with the other’s oneof case。例如，`msg1` 会有一个 `sub_message` 而 `msg2` 会有一个 `name`。

  ```c++
  SampleMessage msg1;
  msg1.set_name("name");
  SampleMessage msg2;
  msg2.mutable_sub_message();
  msg1.swap(&msg2);
  CHECK(msg1.has_sub_message());
  CHECK(msg2.has_name());
  ```

### 6.3 向后兼容

添加或删除 oneof 字段时时请多加注意。如果检查 oneof 的值返回 `None / NOT_SET`，则可能意味着 oneof 尚未设置或已被设置为 oneof 的不同版本中的字段。由于无法知道传输时的未知字段是否是 oneof 的成员，因此无法分辨出差异。

**标签重用问题：**

- **将字段移入或移出 oneof：** message 被序列化和解析后，可能会丢失一些信息（某些字段将被清除）。但是可以安全地将单个字段移动到新字段中。
- **删除一个 oneof 字段并添加它返回：** 在 message 被序列化和解析之后，这可能会清除你当前设置的 oneof 字段。
- **拆分或合并：** 这与移动常规字段有类似的问题。

## 七、Map

repeated 类型可以用来表示数组，而想要表示字典可以用 Map 类型。

```protobuf
map<key_type, value_type> map_field = N;
```

`key_type` 可以是 int 或者 string 类型，不能是 float、double、bytes 和枚举。`value_type` 可以是除 map 外的任何类型。举个例子：

```protobuf
map<string, Project> projects = 3;
```

需要特别注意的是 ：

- map 是不能用 `repeated` 修饰的。
- 线性数组和 map 迭代顺序的是不确定的，所以你不能依靠你的 map 是在一个特定的顺序。
- 为 `.proto` 生成文本格式时，map 按 key 排序。数字类型 key 按数值排序。
- 从 wire 中解析或合并时，如果有重复的 key，则最后一个 key 有效（覆盖原则）。从文本格式解析 map 时，如果有重复的 key，解析可能会失败。
- 如果往 map 中放了一个没有 value 的 key，字段被序列化时的行为取决于具体的语言。在C ++、Java 和 Python 中，类型的默认值是序列化的，而在其他语言中则没有序列化的值。

Protobuf 虽然不支持 map 类型的数组，但是可以转换一下，用 repeated 实现 maps 数组：

```protobuf
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

## 八、Package

你可以在 .proto 文件中添加可选的 package 说明符，以防 message 之间的名称冲突。

```protobuf
package foo.bar;
message Open { ... }
```

然后，可以在定义 message 的字段时使用包说明符：

```protobuf
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

package 编译成具体的代码后，将根据使用语言的不同有不同的实现。对于 Java，将会被用作 Java package，除非你的 .proto 文件中存在明确的 `option java_package` 定义，[点击这里](https://developers.google.com/protocol-buffers/docs/proto3#packages)查看其他语言。

## 九、Service

如果要使用 RPC（远程过程调用），可以在 `.proto` 文件中定义 RPC 服务接口，Protobuf 编译器将使用所选语言生成服务接口代码和 stubs。例如，如果你定义一个 RPC 服务，入参是 SearchRequest 返回值是 SearchResponse，你可以在你的 `.proto` 文件中定义它，如下所示：

```protobuf
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

与 Protobuf 一起使用的最直接的 RPC 系统是 `gRPC`[^3]。gRPC 可以和 Protobuf 无缝结合，甚至允许你通过 Protobuf 编译插件，直接从 `.proto` 文件中生成 RPC 相关的代码。

还有许多正在进行的第三方项目正在为 Protobuf 开发RPC实现，了解相关内容： [third-party add-ons wiki page](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)。

## 十、JSON Mapping

Proto3 支持 JSON 中的标准编码， 让在系统之间分享数据变得容易 。编码在下表中按类型逐个描述。

如果一个值在 JSON 编码的数据中丢失或者它的值是 null，则在解析为 Protobuf 时，它将被解析为对应默认值。如果一个字段在 Protobuf 中具有默认值，默认情况下它将不会出现在 JSON 编码数据中以节省空间。 具体实现应该提供选项来在 JSON 编码输出中出现带有默认值的字段. 

| proto3                 | JSON          | JSON example                              | Notes                                                        |
| :--------------------- | :------------ | :---------------------------------------- | :----------------------------------------------------------- |
| message                | object        | `{"fooBar": v,"g": null,…}`               | 生成 JSON 对象。message 字段名被转换为小驼峰并成为 JSON 对象的 key。如果指定了`json_name` 字段选项，则将指定的值用作键。解析器接受小驼峰名称（或由 `json_name` 选项指定的名称）和原始 protobuf 字段名称。`null` 是所有字段类型的可接受值，并被视为相应字段类型的默认值。 |
| enum                   | string        | `"FOO_BAR"`                               | 使用在 Protobuf 中指定的枚举值的名称，解析器接受枚举名称和整数值。 |
| map<K,V>               | object        | `{"k": v, …}`                             | 所有键都转换为字符串。                                       |
| repeated V             | array         | `[v, …]`                                  | `null` 被接收且被转换为空 List。                             |
| bool                   | true, false   | `true, false`                             |                                                              |
| string                 | string        | `"Hello World!"`                          |                                                              |
| bytes                  | base64 string | `"YWJjMTIzIT8kKiYoKSctPUB+"`              | JSON 值将使用标准 base64 编码成字符串，接受带"/"不带paddings 的标准或 URL 安全的 base64 编码。 |
| int32, fixed32, uint32 | number        | `1, -10, 0`                               | JSON 值为十进制数字，接受数字或字符串。                      |
| int64, fixed64, uint64 | string        | `"1", "-10"`                              | JSON 值将是一个十进制字符串，接受数字或字符串。              |
| float, double          | number        | `1.1, -10.0, 0, "NaN", "Infinity"`        | JSON 值将是数字或特殊字符串值“NaN”、“ Infinity”或“ -Infinity”。接受数字或字符串。指数表示法也被接受。 |
| Any                    | `object`      | `{"@type": "url", "f": v, … }`            | 如果 Any 包含具有特殊 JSON 映射的值，它将被转换成：`{"@type": xxx, "value": yyy}`。否则，该值将转换为 JSON 对象，“ @type”字段为实际的数据类型。 |
| Timestamp              | string        | `"1972-01-01T10:00:20.021Z"`              | 使用 RFC 3339，其中生成的输出将始终进行 Z 归一化，并使用 0、3、6 或 9 个小数位。也可以接受“ Z”以外的偏移。 |
| Duration               | string        | `"1.000340012s", "1s"`                    | 生成的输出始终包含 0、3、6 或 9 个小数位数，具体取决于所需的精度，后跟后缀“ s”。可接受的任何小数位数，只要它们适合纳秒精度，后缀“ s”是必需的。 |
| Struct                 | `object`      | `{ … }`                                   | 任何 JSON 对象，见 `struct.proto`。                          |
| Wrapper types          | various types | `2, "2", "foo", true, "true", null, 0, …` | Wrapper 在 JSON 中使用与包装后的原始类型相同的表示形式，不同之处在于在数据转换和传输期间允许保留 `null`。 |
| FieldMask              | string        | `"f.fooBar,h"`                            | 见`field_mask.proto`。                                       |
| ListValue              | array         | `[foo, bar, …]`                           |                                                              |
| Value                  | value         |                                           | 任何 JSON 值。                                               |
| NullValue              | null          |                                           | JSON 为 null。                                               |
| Empty                  | object        | {}                                        | 一个空的 JSON 对象。                                         |

Proto3 的 JSON 实现中提供了以下几种选项:

- **使用默认值发送字段**：在默认情况下，默认值的字段在 Proto3 JSON 输出中被忽略。一个实现可以提供一个选项来覆盖这个行为，并使用它们的默认值输出字段。
- **忽略未知字段**：默认情况下，Proto3 JSON 解析器拒绝未知字段，但可能提供一个选项来忽略解析中的未知字段。
- **使用 Protobuf 字段名称而不是转换为小驼峰后的名称**：默认情况下，Proto3 JSON 的 printer 将字段名称转换为小驼峰并将其用作 JSON 名称。实现可能会提供一个选项，将原始字段名称用作 JSON 名称。 Proto3 JSON 解析器需要接受转换后的小驼峰名称和原始字段名称。
-  **将枚举值作为整数而不是字符串发送** ：在 JSON 输出中默认使用枚举值的名称。可以提供一个选项来使用枚举值的数值。

## 十一、Options

.proto 文件中的个别声明可以使用数据的 `option` 注解，option 不改变声明的整体意义, 但是在特定上下文会影响它被处理的方式。可用 option 的完整列表定义在 `google/protobuf/descriptor.proto` 。

有些 option  是 **file 级别**，意味着他们应该写在顶级范围，而不是在任何 message、枚举或者 Service 定义之内。有些 option  是 **message 级别**，意味着他们应该写在 message 定义内。有些选项是 **field 级别**,，意味着他们应该写在字段定义内。下面列出一些常用的 option：

- **java_package (file 级别)**：希望生成的 Java 代码使用的 package。如果 .proto 文件中没有显式给出 `java_package` 选项, 则默认使用 proto package(.proto 文件中通过 "`package`" 关键字指定)。如果不生成 Java 代码，则此 option 不起作用。 

  proto package 通常不适合用作 java package，因为 Java 中的 package 形如反转域名，例如 `com.baidu.xxx`。

  ```protobuf
  option java_package = "com.example.foo";
  ```
  
- **java_multiple_files (file 级别)**：为 true 时，每个 message、枚举和 service 都会被生成为一个类，否则这些都会是 ` java_outer_classname` 的内部类。

  ```protobuf
  option java_multiple_files = true;
  ```

- **java_outer_classname  (file 级别)**：要生成的最外层 Java 类的类名（即文件名）。如果在`.proto`文件中没有指定明确的`java_outer_classname`，则通过将`.proto`文件名转换为大驼峰来构造类名称（例如`foo_bar.proto`变为`FooBar.java`）。 如果不生成Java代码，则此 option 不起作用。 

  ```protobuf
  option java_outer_classname = "Ponycopter";
  ```

- **optimize_for (file 级别)**：可以设置为 SPEED、CODE_SIZE 或 LITE_RUNTIME。 或影响 C ++ 和 Java 代码生成器（以及可能的第三方生成器）的运行效率。

  -  `SPEED` (默认)： Protobuf 编译器将生成用于对 message 类型进行序列化、解析和执行其他常见操作的代码，代码已经高度优化。
  - `CODE_SIZE`：Protobuf 编译器会生成最少的类，会依赖共享的、基于反射的代码来实现序列化、解析和其他操作。生成的代码因此会比用 SPEED 生成的小很多，但是操作更慢一些。会提供和在 SPEED 模式下完全相同的API。这个模式主要用在包含非常多数量的 .proto 文件而又不需要他们都运行的极其快的应用中。【注： 生成的代码类最少，生成的总代码量也小，但是操作速度会变慢 】
  - `LITE_RUNTIME`：Protobuf 编译器将生成仅依赖于“精简版”运行时库的类 (`libprotobuf-lite` 替代 `libprotobuf`)。 精简版运行时比完整库要小得多（大约小一个数量级），但省略了某些功能，例如  descriptors  和反射。 对于在受限平台（如手机）上运行的应用程序特别有用。编译器依然会生成所有方法的快速实现，如同在 SPEED 模式下一样。 生成的类将仅以每种语言实现 `MessageLite` 接口，它是完整 Message 接口方法的一个子集。

  ```protobuf
  option optimize_for = CODE_SIZE;
  ```

- **cc_enable_arenas (file 级别)**：对于 C++ 代码生成开启  [arena allocation](https://developers.google.com/protocol-buffers/docs/reference/arenas) 功能。

- **objc_class_prefix  (file 级别)**： Sets the Objective-C class prefix which is prepended to all Objective-C generated classes and enums from this .proto. There is no default. You should use prefixes that are between 3-5 uppercase characters as [recommended by Apple](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html#//apple_ref/doc/uid/TP40011210-CH10-SW4). Note that all 2 letter prefixes are reserved by Apple. 

- **deprecated  (field 级别)**： 如果设置为 true，表明这个字段被废弃，新代码不应该再使用。在大多数语言中这不会有实质影响。在 Java 中，这将会变成一个 `@Deprecated` 标签。 未来在其他特定语言的代码生成器可能在字段的访问器上生成废弃标签，在编译试图使用这个字段的代码时会生成警告。如果这个字段不再被任何人使用而你想阻止新用户使用它，可以考虑将字段声明替换为保留字段。

[^1]: Protobuf，即 Google Protocol Buffers，协议缓冲区。
[^2]: 在 Java 中，无符号的 32 位和 64 位整数使用带符号的对等体表示，最高位仅存储在符号位中。
[^3]: gRPC，谷歌开发的语言和平台中立的开源 RPC 系统
