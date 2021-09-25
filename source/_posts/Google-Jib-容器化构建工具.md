---
title: Google Jib 容器化构建工具
categories: 云原生
abbrlink: a526485e
date: 2019-12-17 21:50:11
icons: [fas fa-fire red]
references:
  - name: jib-maven-plugin
    url: https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 谷歌开源 Java 镜像构建工具 Jib
    url: https://www.infoq.cn/article/2018%2F07%2Fgoogle-opensource-Jib
    rel: nofollow noopener noreferrer
    target: _blank
  - name: 还在用 Dockerfile 部署 Spring Boot？out 啦！试试谷歌的大杀器 Jib
    url: https://www.javaboy.org/2019/1107/docker-springboot.html
    rel: nofollow noopener noreferrer
    target: _blank
related_repos:
  - name: jib-sample
    url: https://github.com/jitwxs/blog-sample/blob/master/springboot-sample/jib-sample
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、前言

随着近些年的技术发展，Java 领域微服务已经成为主流的技术方向。随着微服务化，云原生的概念也逐渐火热起来，不了解云原生仿佛就是一个原始人。而在云原生中，`应用容器化` 是其核心属性之一。

应用容器化，用抽象的话来说就是：将软件容器中的应用程序和进程作为独立的应用程序部署单元运行，并作为实现高级别资源隔离的机制。从总体上改进开发者的体验、促进代码和组件重用，而且要为云原生应用简化运维工作。通俗点说，就是借助于 Docker 等容器化技术，将一个个的微服务打包成镜像，在容器中独立部署运行。

## 二、背景

我司目前采用的是基于 GitLab + Jenkins + Rancher 这套 CI/DI 体系。在这套体系中微服务的容器化依赖于 Jenkins 去实现。现在假设我们有一个项目，其组织结构如下：

```bash
parentPro
    |-- moduleA
    |-- moduleB
    |-- rest    [rest 模块为 spring boot 启动入口，并依赖 moduleA、moduleB]
```

对于 SpringBoot 项目，Maven 的默认构建工具是 `Spring-boot-maven-plugin`，构建出产物为 **Fat Jar**。**Fat jar** 中包含有 **rest 模块**中的 **classes**，及 **rest** 所依赖的 **moduleA**、**moduleB** 及其他第三方 jar 库。最终，通过 Jenkins 的 **Dockerfile** 文件将 **Fat jar** 基于 JDK 基础镜像层构建，产生一个新的应用镜像。

每次应用构建新版本镜像时，因为 Maven 构建产出物是 **Fat jar**，当 **rest**、**moduleA**、**moduleB** 模块中任意一处发生变化时，都会产出一个新的 **Fat jar**。构建镜像时都要将整个 **Fat jar** 重新写入到镜像层，并将整个镜像层推送到镜像仓库中，大大降低了镜像构建和推送的性能，并导致同一个应用镜像的多个 Tag 占用大量的存储空间。

## 三、Google Jib

### 3.1 介绍

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191217223236688.png)

[Jib](https://github.com/GoogleContainerTools/jib ) 是谷歌公司推出的开源 Java 镜像构建工具，它可以将一个 Java 应用构建成 OCI 镜像或者是 Docker 镜像，目前最新的 Relaese 版本为 1.8.0。

JIB 具有以下特点：

1. Jib 使用 Java 开发，并作为 Maven 或 Gradle 的一部分运行。你**不需要编写 Dockerfile 或 Docker 环境**，甚至无需创建包含所有依赖的大 JAR 包，就可以构建出镜像，并将镜像推送到镜像仓库。因为 Jib 与 Java 构建过程紧密集成，所以它可以访问到打包应用程序所需的所有信息。在后续的容器构建期间，它将自动选择 Java 构建过的任何变体。
2. JIB 构建出的应用镜像，**具有分层结构， 利用镜像分层和注册表缓存来实现快速、增量的构建**，提高构建镜像、推送镜像的性能，减少镜像存储空间。
3. 幂等性，Jib 支持根据 Maven 和 Gradle 的构建元数据进行声明式的容器镜像构建，只要输入保持不变，就可以通过配置重复创建相同的镜像。 

下图为某微服务开启 Jib 构建后在 Jenkins 中的构建过程，可以看出构建速度的提升主要在 package 和 push 阶段。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191218233315514.png)

### 3.2 原理

Jib 在编译 Java 应用时，会将 Java 项目内的资源及所依赖的资源，基于变化频率不同分成多个部分，并将每个部分都单独作为一个镜像层存在，这样其中一部分资源发生变化时，只需要重新构建该部分所属镜像层即可。以第二节的应用为例，rest 应用镜像将被分为以下镜像层：

- **Classes**: rest 模块中的 class 信息，这部分信息变化频率最高，处于最上层镜像层；

- **Resources**: rest 模块中的配置文件，这部分信息变化频率较低，处于第二层镜像层；

- **Project Dependencies**: rest 模块的项目依赖信息，在当前示例中为 **moduleA**、**moduleB**，这部分内容比依赖第三方 Jar 库更容易变化，所以也单独做为一个镜像层存在；

- **Snapshot Dependencies**：rest 模块所依赖的 **SnapShot Jar 库**；

- **All other Dependencies**: rest 模块所依赖的其他类型 Jar 库；

- **Each extra directory**：其他所依赖额外资源目录；

基于 Jib 插件构建出的镜像，与使用以下 Dockerfile 所构建出的镜像相同：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191217223153497.png)

## 四、简单上手

### 4.1 基础配置

创建一个全新的 SpringBoot 项目，依赖只包含 `spring-boot-starter-web` 这一个即可。编写一个 Controller 类，用于测试：

```java
@RestController
public class DemoController {
    @GetMapping("/hello")
    public String hello() {
        return "hello world!";
    }
}
```

然后，在 POM 文件中添加 JIB 插件：

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>1.8.0</version>
  <configuration>
    <from>
      <image>harbor.jitwxs-inc.com/base/java:8-jdk-alpine</image>
    </from>
    <to>
      <image>harbor.jitwxs-inc.com/sample/${artifactId}:v1</image>
    </to>
    <allowInsecureRegistries>true</allowInsecureRegistries>
  </configuration>
</plugin>
```

介绍一下含义：

- `<from>` 基础镜像信息，即构建本镜像所基于的根镜像
- `<to>` 输出镜像信息， 表示本镜像构建完成后，要发布到哪里去 
- `<allowInsecureRegistries>` 允许使用 HTTP 协议连接 Registry 仓库

- `<image>` 镜像名，命名格式为：**Registry 仓库地址/属组/镜像名:Tag名**

>由于 Docker Hub 的速度实在是太感人了，开着梯子都 push 不上去，因此我使用了私服仓库。如果使用 Docker Hub，那么 image 标签内容形如：`docker.io/jitwxs/image_name:tag`，其中 jitwxs 为你的 DockerHub 唯一ID，一般是用户名。

配置完毕后，使用如下命令编译，并自动 push 到仓库中：

```powershell
mvn clean package -DskipTests jib:build
```

> 核心就是 `jib:build`，更多命令见文档： https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#build-your-image 

### 4.2 鉴权

运行后，发现抛了如下的错误。根据错误日志可知连接 Registry 仓库时需要鉴权。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191218225655688.png)

#### 4.2.1 命令行

第一种方式也是最粗暴的，在执行 maven 命令时传递 Registry 仓库的用户名密码。

```powershell
mvn clean package -DskipTests jib:build \
    -Djib.from.auth.username=admin \
    -Djib.from.auth.password=admin \
    -Djib.to.auth.username=admin \
    -Djib.to.auth.password=admin
```

由于 `<from>` 和 `<to>` 中的镜像可能不是来自于同一个 Registry 仓库，因此既要配置 from 的用户名密码，也要配置 to 的用户名密码。

执行完毕后，通过命令行，或者可视化工具，查看是否被 push 上去（此处我使用的工具是 [Harbor]( https://github.com/goharbor/harbor )）。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191218230129659.png)

#### 4.2.2 配置文件

使用命令行方式每次执行都要输入那么长一串命令，这样实在是不方便。另一种方法是在 `pom.xml` 文件直接将用户名密码配置进去，形如：

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>1.8.0</version>
  <configuration>
    <from>
      <image>harbor.jitwxs-inc.com/base/java:8-jdk-alpine</image>
      <auth>
        <username>my_username</username>
        <password>my_password</password>
      </auth>
    </from>
    <to>
      <image>harbor.jitwxs-inc.com/sample/${artifactId}:v1</image>
      <auth>
        <username>my_username</username>
        <password>my_password</password>
      </auth>
    </to>
    <allowInsecureRegistries>true</allowInsecureRegistries>
  </configuration>
</plugin>
```

即给 from 和 to 标签都加上 `<auth>` 标签，但是这种方式实在是不够优雅，因为将用户名密码硬编码在代码中会带来安全性问题。

合适的方法是配置在 Maven 的 `settings.xml` 配置文件中，在 `<servers>` 标签中，新增一个 `<server>` 节点，配置 Registry 仓库的用户名密码。

```XML
<servers>
    ...
    <server>
      <id>harbor.jitwxs-inc.com</id>
      <username>admin</username>
      <password>admin</password>
      <configuration>
        <email>admin@jitwxs-inc.com</email>
      </configuration>
    </server>
</servers>
```

配置完毕后，让我们把项目的 tag 从 v1 修改为 v2，再执行次命令验证下：

```powershell
mvn clean package -DskipTests jib:build
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191218231231412.png)

可以看到正常被 push 上去了。最后官方文档详细介绍了各种鉴权方式的使用，参见：https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#authentication-methods 

### 4.3 本地构建

下面试下在本地进行构建，首先使用 docker 命令将镜像拉取下来：

```powershell
> ~: docker pull harbor.jitwxs-inc.com/sample/springboot_jib:v2
v2: Pulling from sample/springboot_jib
53478ce18e19: Pull complete
d1c225ed7c34: Pull complete
887f300163b6: Pull complete
471ae92a2408: Pull complete
286e54d31846: Pull complete
4f4af7a6fe32: Pull complete
Digest: sha256:dfb6628201b1c5fec5eaca00deec157d437559356043043e636fe11b6f3ce1fe
Status: Downloaded newer image for harbor.jitwxs-inc.com/sample/springboot_jib:v2
```

然后基于该镜像，创建容器，并后台运行在 8080 端口：

```powershell
docker run -d --name jib_test -p 8080:8080 harbor.jitwxs-inc.com/sample/springboot_jib:v2
```

打开浏览器，请求接口 `http://127.0.0.1:8080/hello`，正确输出。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191218231542961.png)

### 4.4 绑定到生命周期

如果你不想单独输入 `jib:build`，你可以把 jib 绑定到 Maven 命令中，在插件中添加如下的 `<executions>` 标签即可。

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  ...
  <executions>
    <execution>
      <phase>package</phase>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

然后通过 Maven 的 `mvn package` 命令就会自动构建镜像了。

## 五、验证

这里推荐一个工具 [dive]( https://github.com/wagoodman/dive)， dive 能够通过文件目录的形式直观地显示一个镜像中的每个镜像层内的内容，便于查看镜像的分层信息。

```powershell
./dive harbor.okcoin-inc.com/sample/springboot_jib:v1
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201912/20191223000748304.png)
