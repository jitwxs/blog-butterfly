---
title: gRPC 在 Java 中的入门实例
categories:
  - Java
  - Protobuf
tags: [Protobuf, gRPC]
abbrlink: d6535904
date: 2019-12-26 21:30:04
related_repos:
  - name: grpc-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/grpc-sample
---

## 一、前言

经过前面三篇 Protobuf 相关文章的学习，相信大家已经对 Protobuf 有所掌握。前文说过， ProtoBuf 很适合做数据存储或 RPC 数据交换格式。可以用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

本节将介绍在 Java 中如何使用 gRPC 和 Protouf。gRpc 也是 Google 推出的 RPC 框架，由于师出同门，Protobuf 和 gRPC 可以非常容易的结合在一起，甚至于使用 Protobuf Maven 插件就可以自动生成 gRPC 代码。

如果你还没有看过前序文章，点击这里查看：

- [Protobuf 学习手册——语法篇](/60aca815.html)
- [Protobuf 学习手册——编码篇](/d4225e9b.html)
- [Protobuf 在 Java 中的入门实例](/a5b690ac.html)

## 二、Protobuf 源文件

本文给大家演示的例子是：根据年龄查询用户列表。首先新建一个 Maven 项目，在 `src/main` 目录下（和 java 目录同级）创建 `proto` 文件夹，用于存放 .proto 文件，该目录也是 Protobuf 的 Maven 插件默认扫描的文件夹。

在该文件夹中创建 `message.proto` 文件，用于表示用户实体。

```protobuf
syntax = "proto3";
option java_package = "jit.wxs.grpc.dto";
option java_outer_classname = "MessageProto";

message User {
    int32 age = 1;
    string name = 2;
}
```

简单介绍下：

- `syntax = "proto3";`：使用 proto3 编译，否则默认使用 proto2 编译。
- `option java_package`：指定当前文件编译成 Java 类后的 Package 包路径。
- `option java_outer_classname`：指定当前文件变异成 Java 类后的文件名。

下面创建 `grpc_user.proto` 文件，用于定义根据年龄查询用户列表的 rpc 接口。

```protobuf
syntax = "proto3";
option java_package = "jit.wxs.grpc.rpc";
option java_outer_classname = "UserRpcProto";

import "message.proto";

message AgeRequest {
    int32 age = 1;
}

message UserResponse {
    int32 code = 1;
    string msg = 2;
    repeated User user = 3;
}

service UserRpcService {
    rpc listByAge(AgeRequest) returns(UserResponse);
}
```

在该文件中，定义一个名为 `UserRpcService` 的 Service 类，在其中定义一个名为 `listByAge` 的 rpc 接口，该接口的入参为 AgeRequest 和 UserResponse。

## 三、生成 Java 类

编辑 POM 文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>jit.wxs</groupId>
    <artifactId>grpc</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <protobuf.version>3.11.0</protobuf.version>
        <grpc.version>1.26.0</grpc.version>
    </properties>

    <dependencies>
        <!-- Protobuf Dependency -->
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf.version}</version>
        </dependency>
        <!-- gRPC Dependency Start -->
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <!-- gRPC Dependency End -->
        
        <dependency>
            <groupId>com.googlecode.protobuf-java-format</groupId>
            <artifactId>protobuf-java-format</artifactId>
            <version>1.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.10</version>
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
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <!-- for protobuf -->
                            <goal>compile</goal>
                            <!-- for grpc -->
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

1. 引入 1 个 Protobuf 依赖、3 个 gRPC 依赖和 2 个工具包依赖；
2. 引入 `os-maven-plugin` 插件，用于获取当前运行系统信息。
3. 引入 `protobuf-maven-plugin` 插件，用于生成 Protobuf Java 类和 gRPC Java 类。
   1. `<protocArtifact>`：Protobuf 执行文件路径。
   2. `<pluginId>` + `<pluginArtifact>`：指定 gRPC 插件执行文件路径。
   3. `<goal>compile</goal>`：生成 Protobuf Java 类命令。
   4. `<goal>compile-custom</goal>`：生成 gRPC Java 类命令。

点开 IDEA Maven 插件面板，同时选中 `protobuf:compile` 和 `protobuf:compile-custom` 命令，右击 Run Maven Build 运行，即可得到生成的 Java 类，如下图所示。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202004/20200411215225665.png)

## 四、gRPC 服务端与客户端

### 4.1 gRPC 实现类

创建类 UserRpcServiceImpl，并去实现刚刚 Maven 插件生成的 `UserRpcServiceGrpc.UserRpcServiceImplBase` 接口：

```java
package jit.wxs.grpc.common;

import io.grpc.stub.StreamObserver;
import jit.wxs.grpc.dto.MessageProto;
import jit.wxs.grpc.rpc.UserRpcProto;
import jit.wxs.grpc.rpc.UserRpcServiceGrpc;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.commons.lang3.RandomUtils;

import java.util.ArrayList;
import java.util.List;
import java.util.logging.Logger;
import java.util.stream.IntStream;

/**
 * @author jitwxs
 * @date 2019年12月20日 0:53
 */
public class UserRpcServiceImpl extends UserRpcServiceGrpc.UserRpcServiceImplBase {
    private static final Logger logger = Logger.getLogger(UserRpcServiceImpl.class.getName());

    @Override
    public void listByAge(UserRpcProto.AgeRequest request, StreamObserver<UserRpcProto.UserResponse> responseObserver) {
        logger.info("Server Rec listByAge request...");

        // 构造响应，模拟业务逻辑
        UserRpcProto.UserResponse response = UserRpcProto.UserResponse.newBuilder()
                .setCode(0)
                .setMsg("success")
                .addUser(MessageProto.User.newBuilder()
                        .setName(RandomStringUtils.randomAlphabetic(5))
                        .setAge(request.getAge()).build())
                .addUser(MessageProto.User.newBuilder()
                        .setName(RandomStringUtils.randomAlphabetic(5))
                        .setAge(request.getAge()).build())
                .addUser(MessageProto.User.newBuilder()
                        .setName(RandomStringUtils.randomAlphabetic(5))
                        .setAge(request.getAge()).build())
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

在 `listByAge()` 方法中编写当我们接收到 request 请求时的行为，并将结果放入 response 中。这里我模拟了下业务逻辑，返回了三个年龄等于请求参数的用户。

### 4.2 gRPC 服务端

既然是 RPC 框架，那么就会有服务端和客户端。首先创建服务端：

```java
package jit.wxs.grpc.example1;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import jit.wxs.grpc.common.Constant;
import jit.wxs.grpc.common.UserRpcServiceImpl;

import java.io.IOException;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

/**
 * Grpc 服务端
 * @author jitwxs
 * @date 2019年12月20日 1:03
 */
public class Example1Server {
    private static final Logger logger = Logger.getLogger(Example1Server.class.getName());
    private Server server;

    public static void main(String[] args) throws IOException, InterruptedException {
        final Example1Server server = new Example1Server();
        server.start();
        server.blockUntilShutdown();
    }

    private void start() throws IOException {
        server = ServerBuilder.forPort(Constant.RUNNING_PORT)
                .addService(new UserRpcServiceImpl())
                .build()
                .start();
        logger.info("Server started...");

        // 程序停止钩子
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            // Use stderr here since the logger may have been reset by its JVM shutdown hook.
            System.err.println("*** shutting down gRPC server since JVM is shutting down");
            try {
                Example1Server.this.stop();
            } catch (InterruptedException e) {
                e.printStackTrace(System.err);
            }
            System.err.println("*** server shut down");
        }));
    }

    /**
     * 停止服务
     */
    private void stop() throws InterruptedException {
        if (server != null) {
            server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
        }
    }

    /**
     * Await termination on the main thread since the grpc library uses daemon threads.
     */
    private void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }
}
```

- `start()` 方法中，Server 服务绑定了 8848 端口，并将 UserRpcServiceImpl 加入到 Service 列表中。

- 添加一个 ShutdownHook 的回调钩子，当程序终止的前一刻，该钩子会被回调，方法中执行 `stop()` 方法停止服务端服务。

### 4.3 gRPC 客户端

最后编写客户端去请求服务端。

```java
package jit.wxs.grpc.example1;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;
import jit.wxs.grpc.common.Constant;
import jit.wxs.grpc.common.ProtoUtils;
import jit.wxs.grpc.rpc.UserRpcProto;
import jit.wxs.grpc.rpc.UserRpcServiceGrpc;

import java.util.concurrent.TimeUnit;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Grpc 客户端
 * @author jitwxs
 * @date 2019年12月20日 1:06
 */
public class Example1Client {
    private static final Logger logger = Logger.getLogger(Example1Client.class.getName());

    public static void main(String[] args) throws Exception {
        // STEP1 构造 Channel 和 BlockingStub
        ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", Constant.RUNNING_PORT)
                // Channels are secure by default (via SSL/TLS). For the example we disable TLS to avoid needing certificates.
                .usePlaintext()
                .build();

        UserRpcServiceGrpc.UserRpcServiceBlockingStub blockingStub = UserRpcServiceGrpc.newBlockingStub(channel);

        int requestAge = 20;
        logger.info("Will try to query age = " + requestAge + " ...");

        // STEP2 发起 gRPC 请求
        UserRpcProto.AgeRequest request = UserRpcProto.AgeRequest.newBuilder().setAge(20).build();
        try {
            UserRpcProto.UserResponse response = blockingStub.listByAge(request);
            logger.info("Response: " + ProtoUtils.toStr(response));
        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
        } finally {
            // STEP3 关闭 Channel
            channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
        }
    }
}
```

- 构造方法中，通过 `ManagedChannel` 创建对服务端的连接。
- 构造 Request 请求对象，通过 UserRpcServiceBlockingStub 的 listByAge() 接口去请求服务端，并输出返回的 response。

### 4.4 辅助类

本示例程序还使用了以下辅助类：

```java
package jit.wxs.grpc.common;

public class Constant {
    public static int RUNNING_PORT = 8848;
}
```

```java
package jit.wxs.grpc.common;

import com.google.protobuf.Message;
import com.googlecode.protobuf.format.JsonFormat;

public class ProtoUtils {
    private static JsonFormat JSON_FORMAT;

    static {
        JSON_FORMAT = new JsonFormat();
    }

    public static String toStr(Message message) {
        return JSON_FORMAT.printToString(message);
    }
}
```

## 五、运行程序

整个项目代码目录结构如下：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202004/20200411220150795.png)

首先启动服务端，然后启动客户端。服务端首先接收到客户端请求，输出：

```
Server Rec listByAge request...
```

随后客户端接收到服务端响应，输出：

```json
Response: {"msg": "success","user": [{"age": 20,"name": "Rczby"},{"age": 20,"name": "KXVdZ"},{"age": 20,"name": "setgc"}]}
```
