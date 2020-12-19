---
title: SpringBoot 配置 Logback
categories:
  - Java
  - 日志框架
tags: Logback
abbrlink: a00120ad
date: 2018-12-06 14:16:33
references:
  - name: Spring Boot干货系列：（七）默认日志logback配置解析
    url: https://juejin.im/post/58f86981b123db0062363203
    rel: nofollow noopener noreferrer
    target: _blank
  - name: Spring Boot 日志配置
    url: https://blog.csdn.net/inke88/article/details/75007649
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、前言

`SLF4J`(Simple Logging Facade For Java), 它是针对各类 Java 日志框架的同一抽象，即日志门面。Java 的日志框架众多，SLF4J定义了统一的日志抽象接口。

默认情况下，SpringBoot 采用 `Logback` 来记录日志，并输出 INFO 级别日志到控制台。从下图可以看到，`spring-boot-stater` 的依赖中已经包含了 Logback，因此我们无需手动导入依赖。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201812060950524.png)

从下图可以看到，日志输出内容包含：

|内容|描述|
|:---|:---|
|时间日期|精确到毫秒|
|日志级别|ERROR, WARN, INFO, DEBUG, TRACE|
|进程ID||
|分隔符|标识实际日志的开始|
|线程名|方括号括起来（可能会截断控制台输出）|
|Logger名|通常使用源代码的类名|
|日志内容||

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20181206095123315.png)

## 二、基础配置

### 2.1 控制台输出

日志级别从低到高分为 **TRACE < DEBUG < INFO < WARN < ERROR < FATAL**。如果你将日志级别修改为 ERROR，那么低于 ERROR级别的日志将不会输出。

SpringBoot中默认只输出 `ERROR`、`WARN`、`INFO` 级别的日志到控制台。你可以通过启用调试模式来获取更多的日志输出信息，以下两种方法均可：

1. 在运行命令后加入 `--debug` 标志，如：`$ java -jar hello.jar --debug`
2. 在 application.properties 中配置 `debug=true`，该属性置为 true 的时候，核心Logger（包含嵌入式容器、hibernate、spring）会输出更多内容，但是你自己应用的日志并不会输出为 DEBUG 级别。

### 2.2 文件输出

默认情况下， SpringBoot 不会将日志信息写入到文件中，如果要将其写入到文件中，需要在 application.properties 中配置：

1. `logging.file`，设置文件，可以是绝对路径，也可以是相对路径。如：logging.file=my.log
2. ` logging.path`，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：logging.path=/var/log

>注：二者不能同时使用，如若同时使用，则只有 `logging.file` 生效 
>
>默认情况下，日志文件的大小达到 **10MB** 时会切分一次，产生新的日志文件，默认输出级别为：ERROR、WARN、INFO

### 2.3 级别控制

在 application.properties 中可以自定义日志的输出级别，格式为：`logging.level.* = LEVEL`。其中：

- `logging.level`：日志级别控制前缀，`*` 为包名或 Logger 名
- `LEVEL`：日志级别，选项为 TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF

例如：

- `logging.level.jit.wxs=DEBUG`：jit.wxs 包下所有 class 以 DEBUG 级别输出
- `logging.level.root=WARN`：root 日志以 WARN 级别输出

## 三、自定义配置

### 3.1 自定义配置文件

日志的配置并不一定需要通过 application.properties 进行控制，可以通过系统属性和外部配置文件对日志进行控制和管理。

根据不同的日志系统，按照如下组织配置文件名，就能够正确的加载外部配置文件：

- **Logback**：`logback-spring.xml`, `logback-spring.groovy`, logback.xml, logback.groovy
- **Log4j**：`log4j-spring.properties`, `log4j-spring.xml`, log4j.properties, log4j.xml
- **Log4j2**：`log4j2-spring.xml`, log4j2.xml
- **JDK (Java Util Logging)**：logging.properties

SpringBoot 官方推荐优先使用带有 `-spring` 的文件名作为你的日志配置（如使用 logback-spring.xml，而不是logback.xml），SpringBoot 会对以 -spring 结尾的日志配置添加一些 SpringBoot 所特有的配置项。

这些配置文件默认放在 `src/main/resources` 下即可，如果你想自定义存放路径和配置文件名，可以通过设置 `logging.config` 属性即可：

```ini
logging.config=classpath:logging-config.xml
```

### 3.2 节点说明

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration  scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>logback</contextName>
    <property name="log.path" value="E:\\logback.log" />
    <!--输出到控制台-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
       <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>-->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!--输出到文件-->
    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logback.%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="console" />
        <appender-ref ref="file" />
    </root>

    <logger name="jit.wxs.controller"/>
    <logger name="jit.wxs.controller.DemoController" level="WARN" additivity="false">
        <appender-ref ref="console"/>
    </logger>
</configuration>
```

#### 3.2.1 configuration

该节点为配置文件的根节点，它包含以下几个属性：

|属性名|描述|
|:--|:--|
|scan|当值为 true 时，配置文件如果被修改，将会被重新加载，默认 为true|
|scanPeriod|当 scan为true 时，属性生效。设置检测配置文件是否被修改的时间间隔，默认单位是毫秒，默认时间间隔为 1 分钟|
|debug|当值为 true 时，打印 logback 的内部日志信息，默认为 false|

#### 3.2.2 contextName

每个 logger 都关联到 logger 上下文，默认该值为 `defalut`，可以设置成其他名字**用于区分不同应用程序的记录**。设置后无法修改，可以通过 `%contextName` 打印出来。

```xml
<contextName>logback</contextName>
```

#### 3.2.3 property

**用来定义变量值**，具有两个属性，`name` 定义变量的名称，`value` 定义变量的值。定义的值会被保存到 logger 的上下文中，通过 `${}` 取出变量。

```xml
<property name="log.path" value="E:\\logback.log" />
```

#### 3.2.4 appender

**用来格式化日志输出结点**，它的 `class` 属性用来指定输出策略，常用的分为**控制台输出**和**文件输出**。

##### 3.2.4.1 控制台输出

```xml
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
   <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>ERROR</level>
    </filter>-->
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

`ThresholdFilter` 为系统定义的拦截器，上例过滤掉 `ERROR` 级别以下的日志不输出到文件中。

`<encoder>` 标签表示对日志进行格式化：

- `%d{HH: mm:ss.SSS}`——日志输出时间
- `%thread`——输出日志的进程名
- `%-5level`——日志级别，并且使用5个字符靠左对齐
- `%logger{36}`——日志输出者的名字
- `%msg`——日志消息
- `%n`——换行符

##### 3.2.4.2 文件输出

```xml
<appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${log.path}</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>logback.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
        <totalSizeCap>1GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
</appender>
```

 `<file>` 标签指定了输出的文件路径，`<encoder>` 标签表示对日志进行格式化，在 `<rollingPolicy>` 标签中：
- `fileNamePattern` 定义了**日志的切分方式**，上例为按日切分。
- `maxHistory` 表示**日志的保留时长**，上例为保留 30 天的日志。
- `totalSizeCap` 表示**日志文件的上限大小**。例如设置为 1GB 的话，那么当达到该值时，会删除旧的日志。

#### 3.2.5 root

```xml
<root level="info">
    <appender-ref ref="console" />
    <appender-ref ref="file" />
</root>
```

`<root>` 是不可省略的标签，用来指定最基础的日志输出级别，`level` 用来设置打印级别，可设置为：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，默认值为 DEBUG。

可以包含 0 个或多个 `appender-ref` 子标签，用于标识这个 `<appender>` 将会添加到这个 logger 中。

#### 3.2.6 logger

`<logger>`标签用来设置某一个包或者据以的某一个类的日志打印级别，包含以下属性：

- `name`：用来指定手册 logger 约束的某一个包或者某个具体类。
- `level`：可选，设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，还有一个特俗值 INHERITED 或者同义词 NULL，代表强制执行上级的级别。如果未设置此属性，那么当前 logger 将会继承上级的级别。
- `addtivity`：可选，是否向上级传递打印信息，默认为 true。

下面给出两种实例，以下面这个代码为例：

```java
@RestController
public class DemoController {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @PostMapping("/login")
    public Map<String, Object> login(String username, String password, HttpSession session) {
        //日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出。
        logger.trace("日志输出 trace");
        logger.debug("日志输出 debug");
        logger.info("日志输出 info");
        logger.warn("日志输出 warn");
        logger.error("日志输出 error");
        Map<String, Object> map = new HashMap<>(16);
        if (StringUtils.isNotBlank(username) && StringUtils.isNotBlank(password)) {
            User user = new User(username, password);
            session.setAttribute("user", user);
            map.put("result", 1);
        } else {
            map.put("result", 0);
        }
        return map;
    }
}
```

##### 3.2.6.1 不指定级别，不指定 appender

```xml
<logger name="jit.wxs.controller"/>
```

该标签打印 controller 包下的所有类的日志，但是并没有设置打印级别，因此继承父日志级别 info。
因为没有设置 addtivity，使用默认值 true，此时 logger 的打印信息将向父级传递。
因为没有设置 appender，因此本身不会打印任何信息。

如果我们的 `<root>` 标签设置如下：

```xml
<root level="info">
    <appender-ref ref="console" />
</root>
```

那么当执行 login() 方法时，首先执行 `<logger name="jit.wxs.controller"/>`，将级别为 info 级以上日志级别的日志传递给 root，其自身不打印。root 接收到信息后，交给名为 console 的 `appender` 进行处理，appender 输出如下：

```console
11:10:23.407 logback [http-nio-8080-exec-8] INFO  jit.wxs.controller.DemoController - 日志输出 info
11:10:23.408 logback [http-nio-8080-exec-8] WARN  jit.wxs.controller.DemoController - 日志输出 warn
11:10:23.408 logback [http-nio-8080-exec-8] ERROR jit.wxs.controller.DemoController - 日志输出 error
```

##### 3.2.6.2 指定级别，指定 appender

```xml
<logger name="jit.wxs.controller.DemoController" level="WARN" additivity="false">
    <appender-ref ref="console"/>
</logger>
```

设置 `jit.wxs.controller.DemoController` 类的打印级别为 WARN，additivity 属性为 false，表示此 logger 信息不再向父级传递，指定了名为 console 的 appender。

此时执行 login() 方法，先执行 `<logger name="jit.wxs.controller.DemoController" level="WARN" additivity="false">`，将级别为 WARN 及以上级别的日志信息交由名为 console 的 appender 处理，不再向父级 root 传递信息。

打印结果如下：

```console
11:12:54.212 logback [http-nio-8080-exec-8] WARN  jit.wxs.controller.DemoController - 日志输出 warn
11:12:54.212 logback [http-nio-8080-exec-8] ERROR jit.wxs.controller.DemoController - 日志输出 erro
```

#### 3.2.7 多环境日志输出

>如需使用该功能，配置文件名末尾必须带有 `-spring`

```xml
<!-- 测试环境+开发环境. 多个使用逗号隔开. -->
<springProfile name="test,dev">
    <logger name="jit.wxs.controller" level="info" />
</springProfile>
<!-- 生产环境. -->
<springProfile name="prod">
    <logger name="jit.wxs.controller" level="ERROR" />
</springProfile>
```

通过在 application.properties 中指定项目的运行环境，来实现多环境日志输出：

```ini application.properties
# test、dev、prod
spring.profiles.active=test
```

#### 3.2.8 获取 application.propertis 参数

在日志配置文件中，可以通过 `springProperty` 标签来读取 application.propertis 中的参数，例如定义了 contenxtName:

```ini
logback.context-name=logback
```

在日志配置文件中，读取该属性：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <!--application.properties 传递参数，不能使用<property>标签 -->
    <springProperty scope="context" name="contextName" source="logback.context-name"/>

    <contextName>${contextName}</contextName>
	...
</configuration>
```
