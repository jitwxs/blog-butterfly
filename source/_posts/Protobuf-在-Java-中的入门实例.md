---
title: Protobuf 在 Java 中的入门实例
categories:
  - Java High Performance
  - Protobuf && gRPC
tags: Protobuf
abbrlink: a5b690ac
date: 2019-12-23 00:43:58
related_repos:
  - name: protobuf-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/protobuf-sample
---

`Protobuf`[^1] 是一种语言中立、平台无关、可扩展的**序列化数据的格式**，可用于通信协议，数据存储等。

本文将演示在 Java 语言中如何编写一个 Protobuf 的入级程序，也许你可能并不了解 Protobuf，这没有关系，基于 Protobuf 官方文档的衍生博文已经安排上了，只是限于内容较多，我正在一点点写作中，让我们先来简单实战吧！

**注：** 本文及后续所有关于 Protobuf 相关文章均采用 Protobuf3 版本，具体为 `Protobuf 3.11.0`。

## 一、插件

工欲善其事，必先利其器！让我们先在全宇宙第一的 Java IDE 中安装上 Protobuf 的插件。在插件市场中搜索并安装 `Protobuf Support`，或下载离线插件包手动安装： https://plugins.jetbrains.com/plugin/8277-protobuf-support/ 。

![Protobuf Support Plugins](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191223005652462.png)

安装完毕后，创建一个空的 Maven 工程。借助于 Protobuf Maven 插件的功劳，使我们不必在本地搭建 Protobuf 环境。直接编辑 Pom 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    ...
    <properties>
        <protobuf.version>3.11.0</protobuf.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf.version}</version>
        </dependency>
    </dependencies>

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

首先我们引入了 `protobuf-java` 的 Java 包依赖，然后引入了两个 Maven 插件：

- `os-maven-plugin`[^2] 获取当前运行环境的系统信息，作为辅助插件。
- `protobuf-maven-plugin` 负责将 protobuf 文件转换为 Java 类。`<protocArtifact>` 内容是 protobuf 转化为 Java 类的执行文件的配置信息，由于 `${os.detected.classifier}` 可以自动识别当前环境的操作系统信息，自动适配不同操作系统，因此无需修改。

## 二、Protobuf 源文件

`protobuf-maven-plugin` 插件默认会扫描 `src/main/proto` 下的 protobuf 文件，因此我们需要将 protobuf 文件放置在该目录下。如果你想放置在其他地方，请手动修改 maven 插件的 `<protoSourceRoot>`[^3] 属性。

首先创建一个关于性别的 Protobuf 枚举类 `sex_enum.proto`，用于标识男女：

```protobuf
syntax = "proto3";
option java_package = "jit.wxs.demo.enums";
option java_outer_classname = "SexEnumProto";

enum SexEnum {
    INVALID = 0;
    MALE = 1;
    FEMALE = 2;
}
```

- `syntax = "proto3";`：指定使用 Protobuf3 进行编译，不写默认使用 Protobuf2 编译。
- `option java_package`：指定生成 Java 类后的所属包。
- `option java_outer_classname = "SexEnumProto";`：指定生成 Java 类后的类名。

关于枚举类，有以下几点说明：

- 枚举值序号必须从 0 开始。
- 序号为 0 的枚举值必须是第一个元素。
- 为 0 的枚举值是该枚举类的默认值。

然后再创建一个用户 Protobuf 类 `user.proto`：

```protobuf
syntax = "proto3";
option java_package = "jit.wxs.demo.dto";
option java_outer_classname = "UserProto";

import "sex_enum.proto";

message User {
    int32 age = 1;
    string name = 2;
    SexEnum sex = 3;
}
```

- `import "sex_enum.proto";` 表示将 sex_enum.proto 类引入到当前文件中，以便调用。
- `age`：年龄数据类型为 int32，默认值为 0，在 Java 中是 int 类型。
- `name`：姓名数据类型是 string，默认值为空串，在 Java 中数据类型是 String 类型。
- `sex`：性别数据类型为 SexEnum 类型，默认值为 INVALID。

目录结构如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191223011801327.png)

## 三、生成 Java 类

在 IDEA 中，双击 Maven Protobuf 插件的 `protobuf:complie `选项，在项目的 target 文件夹中就会生成 protobuf 文件对应的 Java 类，如下图所示。

![Protobuf Compile](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191223011936993.png)

## 四、运行

将生成的两个 Java 类**移动**（不要复制，否则运行会报文件名重复错误）到项目的 java 目录下（保持 package 的正确性），然后编写一个测试类测试下。

```java
package jit.wxs.demo.test;

import com.google.protobuf.InvalidProtocolBufferException;
import jit.wxs.demo.dto.UserProto;
import jit.wxs.demo.enums.SexEnumProto;

import java.util.Arrays;

/**
 * 测试 ProtoBuf 序列化与反序列化
 * @author jitwxs
 * @date 2019年12月20日 1:22
 */
public class ProtoBufTest {
    public static void main(String[] args) {
        UserProto.User.Builder builder =  UserProto.User.newBuilder();
        builder.setName("zhangsan");
        builder.setAge(18);
        builder.setSex(SexEnumProto.SexEnum.MALE);

        UserProto.User user = builder.build();
        System.out.println("before: " + user.toString());

        byte[] byteArray = user.toByteArray();
        System.out.println("=======");
        System.out.println("User Byte: " + Arrays.toString(byteArray));
        System.out.println("=======");

        try {
            UserProto.User user2 = UserProto.User.parseFrom(byteArray);
            System.out.println("after: " + user2.toString());
        } catch (InvalidProtocolBufferException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果如下：

```
before: age: 18
name: "zhangsan"
sex: MALE

=======
User Byte: [8, 18, 18, 8, 122, 104, 97, 110, 103, 115, 97, 110, 24, 1]
=======
after: age: 18
name: "zhangsan"
sex: MALE
```

最终项目的目录结构如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/2019122301234225.png)

[^1]: Google Protocol Buffers，协议缓冲区。
[^2]:  https://github.com/trustin/os-maven-plugin 
[^3]:  https://www.xolstice.org/protobuf-maven-plugin/index.html 
