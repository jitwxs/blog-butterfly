---
title: 上传 Jar 包至 Maven 中央仓库
categories: 开发工具
tags: Maven
abbrlink: 4714ed08
icons: [fas fa-fire red]
date: 2020-05-17 22:32:48
copyright_author: Jitwxs
---

## 一、前言

随着时间积累，在平常写自己的代码过程中，会有类或者是模块，比较通用，许多项目都能用得到。我们就可以把这些部分抽取成一个公共包，方便其他项目去使用。

本地 install 只能保存在本地中，因此将其上传到中央仓库中，平常就能够轻松的去使用了。上传 Jar 包的过程还算简单，本文将记录这一过程，系统环境为 Windiws。

另外，请使用 windows 自带的 `CMD` 作为整篇文章的命令行工具。不要使用 Git Bash，会有坑。

## 二、创建工单

首先你得有个 sonatype 的账号，[点击这里](https://issues.sonatype.org/secure/Signup!default.jspa)前往注册。最好使用 Chrome 浏览器进行注册，因为当我使用微软 Chrome 内核的 Edge 浏览器注册时，当输入不满足条件时，没有弹出错误提示。

![注册账户](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523211803286.png)

注册完毕后，点击上方导航栏蓝色的创建按钮，创建一个工单。

> 可以参考我的工单:  https://issues.sonatype.org/browse/OSSRH-57685 

**问题类型：** New Project

**概要 / 描述 ：** 简单描述下想要上传的意图就好

**GroupId:** 要上传的 Maven 项目的 GroupId，对于我们个人用户，一般使用 `com.github.用户名` 这种命名方式。当然如果你用的是其他代码管理平台，用其他的也 ok，比如 `com.gitee.用户名`。

**Project Url:** 项目的主页，例如我的是：`https://github.com/jitwxs/commons`

**SCM Url:** 项目 Clone 的地址，注意要是 http 协议的，而不是 git 协议的，例如我的是：`https://github.com/jitwxs/commons.git`

**Already Synced to Central:** 对于上传新项目来说，选择 No 即可。

![创建工单](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523212749327.png)

工单创建完毕后，会有客服发来消息，要求在 GitHub 账户中，创建一个指定名称的仓库，来验证你对账户的所有权。创建完毕后，回复创建完毕即可（创建的这个仓库，等到流程完毕后，可以删了）。

![创建验证仓库](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523213714851.png)

随后客服提示已经审核完毕，可以准备上传了。

![准备上传](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523213915535.png)

## 三、上传 Jar 包

### 3.1 Maven 配置文件

编辑 Maven `setting.xml` 文件。在 `<servers>` 节点中加入刚刚注册的账户信息。

```xml
<servers>  
	<server> 
        <id>sonatype-nexus-snapshots</id> 
        <username>sonatype 账户名</username> 
        <password>sonatype 密码</password> 
    </server>
        <server> 
        <id>sonatype-nexus-staging</id> 
        <username>sonatype 账户名</username> 
        <password>sonatype 密码</password> 
    </server>
</servers>
```

### 3.2 Project Pom

#### 3.2.1 主要信息

在 Pom 文件中添加 `SCM` 和描述信息，这类描述标签，我觉得是越全越好，免得后面提交失败。更多描述标签，可以[参考我的]( https://github.com/jitwxs/commons/blob/master/pom.xml)，是可以提交成功的。

```xml
<description>A simple and useful java commons packaging</description>

<scm>
    <connection>scm:git:https://github.com/jitwxs/commons.git</connection>
    <developerConnection>scm:git:git@github.com:jitwxs/commons.git</developerConnection>
    <url>https://github.com/jitwxs/commons</url>
</scm>
```

#### 3.2.2 build 插件

在 Pom 文件中添加 build 插件，除去 `maven-compiler-plugin` 这个原本就需要的，需要 `maven-source-plugin`、`maven-javadoc-plugin`、`maven-gpg-plugin` 这三个插件。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <execution>
                    <id>attach-sources</id>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <!-- Javadoc -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>3.1.1</version>
            <configuration>
                <charset>UTF-8</charset>
                <encoding>UTF-8</encoding>
                <docencoding>UTF-8</docencoding>
            </configuration>
            <executions>
                <execution>
                    <id>attach-javadocs</id>
                    <goals>
                        <goal>jar</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <!-- GPG -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <version>1.6</version>
            <executions>
                <execution>
                    <phase>verify</phase>
                    <goals>
                        <goal>sign</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### 3.2.3 profile

最后添加一个 profile 用于后续的上传，注意此处 `<distributionManagement>` 中的 id 名，需要与 Maven `setting.xml` 中刚刚配置的 id 一致。

```xml
<profiles>
    <profile>
        <id>sonatype-release</id>
        <distributionManagement>
            <snapshotRepository>
                <id>sonatype-nexus-snapshots</id>
                <name>Sonatype Nexus Snapshots</name>
                <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
            </snapshotRepository>
            <repository>
                <id>sonatype-nexus-staging</id>
                <name>Nexus Release Repository</name>
                <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
            </repository>
        </distributionManagement>
    </profile>
</profiles>
```

### 3.3 Gpg 加密

上传 jar 包过程，需要使用秘钥，对于 Windwos 系统，使用 `Gpg4win` 即可，[点击这里]( https://www.gpg4win.org/download.html )下载。一路傻瓜式安装即可，唯一要注意的是安装组件那边，其实我们只使用了 `GunPG`，因此其他可选项都可以不安装。

![Gpg4win](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201906/20190627150325388.png)

安装完毕后，执行命令 `gpg --version` 验证是否安装成功。

![验证是否安装成功](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523215327357.png)

执行命令 `gpg --gen-key` ，提示填写 `Real name` 和 `Email address`，自行填写即可，输入字母 O 后生成秘钥。

![生成秘钥](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523215608152.png)

执行命令 ` gpg --list-keys` 查看所有已经创建的秘钥信息。

![查看秘钥](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200523215850127.png)

上图红框处为刚刚生成的秘钥的公钥，复制下来，并将其上传至 PGP 秘钥服务器。

```
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --send-keys 红框内容
```

上传完毕后，执行如下命令，查看是否上传成功。

```
gpg --keyserver hkp://keyserver.ubuntu.com:11371 --recv-keys 红框内容
```

![验证是否上传成功](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/202005232201231.png)

## 四、发布

### 4.1 Deploy

进入Maven项目跟目录，执行 mvn 发布命令（-P 后跟的是 profile 的 id 名），会弹窗要求输入密码，输入 sonatype 密码即可。

```cmd
mvn clean deploy -DskipTests -P sonatype-release
```

![命令行上传](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524104350463.png)

此时会抛一些错误，例如我这边是 Javadoc 写的不规范，按提示将错误处理了即可。

![错误提示](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524104431369.png)

但是，我更推荐直接使用 IDEA 操作，勾选 Profike 后，同时选中 clean、deploy，右键 run 即可。使用这种方式，都没有让我输入密码。

![IDEA 上传](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524104449245.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524104509473.png)

### 4.2 Repository Manager

打开 [Nexus Repository Manager](https://oss.sonatype.org)，使用 sonatype 账户登录即可。点击左侧的 `Staging Repositories`，可以找到我们刚刚 deploy 的项目，目前处理 open 状态。

![Staging Repositories](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524104748924.png)

网上几乎所有的相关文章，对于 Repository Manager 相关的描述，都只是说要怎么做，却不告诉大家为什么要这么做，导致让新手操作起来很晕。

1. 其实这边的流程很简单，当我们从本地 deploy 到远程后，它处于暂存区当中，类似于 git 暂存区的概念，即上图中的 Open。在这个状态下，你可以不停地重新 deploy，新的 deploy 内容会覆盖掉老的。

2. 当你觉得修改的差不多了，可以正式提交了。就需要点击上方的 `Close` 按钮，点击 Close 后，系统会对你提交的内容进行一系列的检测。如果没有通过检测，你需要点击 `Drop` 按钮，将本次的 deploy 给删除掉，然后重新 deploy，回到上一步的流程。

3. 如果已经通过了检测，点击上方的 `Release` 按钮，就会将项目正式的提交到中央仓库中。

现在开始正式操作，点击上方的 Close 按钮，准备提交。弹窗中的描述随便填写即可，比如我填写的是本次发布的版本号 1.3。

![Close Project](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524105544719.png)

点击上方的 `Refresh` 按钮刷新一下，在 `Activity` 标签页，可以看到当前的检测进度。

当所有检测项均为 Passed 后，就可以进入下一步了。如果有某一项没有通过，先 `Drop` 掉本次提交，按照错误提示修改完代码后，重新走一遍流程即可。

![Check Pass](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524105820160.png)

点击上方的 `Release` 按钮，正式提交到中央仓库。

![Release Project](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524110025104.png)

点击上方的 `Refresh` 按钮刷新一下，在 `Activity` 标签页，可以看到当前的上传进度。当上传完毕后，内容会自动为空。此时在左侧的搜索框搜索项目，右侧能够显示出来，即表示上传成功。

![Search Project](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524110401760.png)

### 4.3 回复工单

对于项目的首次 Release，需要回复下工单，告诉客服已经发布了。后续的提交，就不再需要回复工单，这个流程了。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/202005/20200524110546973.png)

## 五、常见问题

### 5.1 上传的项目有 Bug 怎么办，我能删掉重传吗？

[官方表示](https://central.sonatype.org/articles/2014/Feb/06/can-i-change-a-component-on-central/)不支持这样的操作，如果传错了，那么就再发布一个新的 Release 版本即可。如果你还是想知道怎么删除掉，可以去创建一个工单，去碰碰运气吧。

### 5.2 我的项目，要过多久才能够上传完毕？

如果你的 Maven 使用官方默认源的话，大概上传后 10 分钟左右就可以使用了，验证方法是匿名方式在 [Nexus Repository Manager](https://oss.sonatype.org) 中进行搜索，能搜索出来就可以了。

如果你使用的是阿里的源，那么大概要 30 分钟，也许更久，这依赖于阿里的同步速度。

如果你想在 [mvnrepository](https://mvnrepository.com) 上搜索出来，得要好几天，佛系点吧。
