---
title: WebSoket 初探并于 SpringBoot 整合
tags: WebSoket
categories:
  - Java Web
  - SpringBoot
abbrlink: 9af7a6d1
date: 2018-10-10 19:43:04
related_repos:
  - name: springboot_ws
    url: https://github.com/jitwxs/blog-sample/blob/master/SpringBoot/springboot_ws
    rel: nofollow noopener noreferrer
    target: _blank
copyright_author: Jitwxs
---

## 一、WebSocket

### 1.1 HTTP与WebSocket

WebSocket 是一种网络通信协议。[RFC6455](https://tools.ietf.org/html/rfc6455) 定义了它的通信标准。WebSocket 是 HTML5 开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。

我们知道，HTTP 协议是一种**无状态的、无连接的、单向**的应用层协议。它采用了请求/响应模型。**通信请求只能由客户端发起，服务端对请求做出应答处理**。

这种通信模型有一个弊端：**HTTP 协议无法实现服务器主动向客户端发起消息**。这就注定了如果服务器有连续的状态变化，客户端要获知就非常麻烦。大多数 Web 应用程序将通过**轮询请求**。轮询的效率低，非常浪费资源（因为必须不停连接，或者 HTTP 连接始终打开）。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010102935122.png)

为了解决HTTP的这一痛点，WebSocket就被发明出来，它的最大特点就是，**服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息**，是真正的双向平等对话。

WebSocket具有以下特点：

（1）建立在 TCP 协议之上，服务器端的实现比较容易。

（2）与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。

（3）数据格式比较轻量，性能开销小，通信高效。

（4）可以发送文本，也可以发送二进制数据。

（5）没有同源限制，客户端可以与任意服务器通信。

（6）协议标识符是`ws`（如果加密，则为wss），服务器网址就是 URL。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010103156458.png)

### 1.2 WebSocket客户端

WebSocket被HTML5所支持，因此创建一个WebSocket客户端十分简单：

```javascript
var ws= null;

if ('WebSocket' in window) {
    ws = new WebSocket("ws://localhost:8080/ws");
} else if ('MozWebSocket' in window) {
    ws = new MozWebSocket("ws://localhost:8080/ws");
} else {
    alert('您的浏览器不支持WebSocket，请更换浏览器');
}
```

以上代码中的第一个参数 url, 指定连接的 URL。第二个参数 protocol 是可选的，指定了可接受的子协议。

通过调用`readyState`属性，可以获取当前状态，具有以下几种取值：

| 常量名               | 数值 | 含义                           |
| :-------------------- | ---- | :------------------------------ |
| WebSocket.CONNECTING | 0    | 正在连接                       |
| WebSocket.OPEN       | 1    | 连接成功，可以通信             |
| WebSocket.CLOSING    | 2    | 连接正在关闭                   |
| WebSocket.CLOSED     | 3    | 连接已经关闭，或者打开连接失败 |

```javascript
switch (ws.readyState) {
  case WebSocket.CONNECTING:
    // do something
    break;
  case WebSocket.OPEN:
    // do something
    break;
  case WebSocket.CLOSING:
    // do something
    break;
  case WebSocket.CLOSED:
    // do something
    break;
  default:
    // this never happens
    break;
}
```

WebSocket具有以下几个回调方法：

```javascript
//连接发生错误的回调方法
ws.onerror = function(){
};

//连接成功建立的回调方法
ws.onopen = function(event){
};

//接收到消息的回调方法
ws.onmessage = function(event){
    console.log(event.data);
    ws.send(event.data);
};

//连接关闭的回调方法
ws.onclose = function(){
    ws.close();
};
```

通过调用`send()`和`close()`方法发送消息和关闭连接。


## 二、与SpringBoot整合

SpringBoot版本：`2.0.5.RELEASE`，相关代码如下：

### 2.1 HelloWorld

#### 2.1.1 导入依赖

如果我们使用SpringBoot内置的Tomcat容器，那么我们直接使用SpringBoot提供的WebSocket包即可，导入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

需要注意 `spring-boot-starter-websocket` 属于高级组件，已经包含了 `spring-boot-starter` 和 `spring-boot-starter-web` ，因此注意不要重复导包。

#### 2.1.2 创建 WebSocket Endpoint

首先要注入`ServerEndpointExporter`，这个bean会自动注册使用了`@ServerEndpoint`注解声明的Websocket endpoint。

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

然后就可以编写具体的WebSocket操作类了：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Objects;
import java.util.concurrent.CopyOnWriteArraySet;

@ServerEndpoint(value = "/ws")
@Component
public class WebSocketServer {
    private Logger log  = LoggerFactory.getLogger(this.getClass());

    /**
     * 用来存放每个客户端对应的MyWebSocket对象
     */
    private static CopyOnWriteArraySet<WebSocketServer> webSocketSet = new CopyOnWriteArraySet<>();

    /**
     * 与某个客户端的连接会话，需要通过它来给客户端发送数据
     */
    private Session session;

    /**
     * 连接建立成功
     * @author jitwxs
     * @since 2018/10/10 9:44
     */
    @OnOpen
    public void onOpen(Session session) {
        this.session = session;
        webSocketSet.add(this);
        log.info("【WebSocket】客户端：{} 加入连接！当前在线人数为：{}", session.getId(), webSocketSet.size());

        sendMessage("已接受您的连接请求");
    }

    /**
     * 连接关闭
     * @author jitwxs
     * @since 2018/10/10 9:45
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);
        log.info("【WebSocket】客户端：{} 关闭连接！当前在线人数为：{}", this.session.getId(), webSocketSet.size());

    }

    /**
     * 收到客户端消息
     * @author jitwxs
     * @since 2018/10/10 9:45
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("【WebSocket】收到来自客户端：{} 的消息，消息内容：{}", session.getId(), message);
        sendMessage("收到消息：" + message);
    }

    /**
     * 发生错误
     * @author jitwxs
     * @since 2018/10/10 9:46
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.info("【WebSocket】客户端：{} 发生错误，错误信息：", session.getId(), error);
    }


    /**
     * 对当前客户端发送消息
     * @author jitwxs
     * @since 2018/10/10 9:49
     */
    public void sendMessage(String message) {
        this.session.getAsyncRemote().sendText(message);
//        this.session.getBasicRemote().sendText(message);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        WebSocketServer that = (WebSocketServer) o;
        return Objects.equals(session, that.session);
    }

    @Override
    public int hashCode() {
        return Objects.hash(session);
    }
}
```

使用`@ServerEndpoint`注解制定了WebSocket的路径，通过`@Compontent`注解加入Spring容器，通过`@Open`、`@OnClose`、`@OnMessage`、`@OnError`注解处理相应的WebSocket请求。

这里注意下`session.getAsyncRemote()`和`session.getBasicRemote()`的区别：

getAsyncRemote()为**异步**，getBasicRemote()为**同步**。**大部分情况下，推荐使用getAsyncRemote()**。

由于 getBasicRemote() 的同步特性，并且它支持部分消息的发送即 sendText(xxx,boolean isLast)， `isLast` 的值表示是否一次发送消息中的部分消息，对于如下情况：

```java
session.getBasicRemote().sendText(message, false); 
session.getBasicRemote().sendBinary(data);
session.getBasicRemote().sendText(message, true); 
```

由于同步特性，第二行的消息必须等待第一行的发送完成才能进行，而第一行的剩余部分消息要等第二行发送完才能继续发送，所以在第二行会抛出`IllegalStateException`异常。

因此如果要使用 getBasicRemote() 发送消息，则避免尽量一次发送全部消息，使用部分消息来发送。

#### 2.1.3 编写页面

然后写一个简单的页面来测试下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Index</title>
</head>
<body>

<input id="text" type="text"/>
<button onclick="send()">Send</button>
<button onclick="closeWebSocket()">Close</button>
<div id="message"></div>

<script>
    var ws= null;

    // 建立连接
    if ('WebSocket' in window) {
        ws = new WebSocket("ws://localhost:8080/ws");
    } else if ('MozWebSocket' in window) {
        ws = new MozWebSocket("ws://localhost:8080/ws");
    } else {
        alert('您的浏览器不支持WebSocket，请更换浏览器');
    }

    //连接发生错误的回调方法
    ws.onerror = function(){
        setMessageInnerHTML("error");
    };

    //连接成功建立的回调方法
    ws.onopen = function(){
        setMessageInnerHTML("open");
    };

    //接收到消息的回调方法
    ws.onmessage = function(event){
        setMessageInnerHTML(event.data);
    };

    //连接关闭的回调方法
    ws.onclose = function(){
        setMessageInnerHTML("close");
    };

    //监听窗口关闭事件，当窗口关闭时，主动去关闭webSocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        ws.close();
    };

    //将消息显示在网页上
    function setMessageInnerHTML(innerHTML) {
        document.getElementById('message').innerHTML = innerHTML + '<br/>';
    }

    //关闭连接
    function closeWebSocket(){
        ws.close();
    }

    //发送消息
    function send(){
        let message = document.getElementById('text').value;
        ws.send(message);
    }
</script>
</body>
</html>
```

#### 2.1.4 测试

当页面加载完毕时，建立连接：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010102439594.png)

客户端发送消息 + 服务端回复：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010102508747.png)

客户端主动关闭连接：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010102530650.png)

### 2.2 心跳包检测

在使用Websocket连接建立数分钟后（一说是10分钟），会自动断开连接，所以就需要一种机制来检测客户端和服务端是否处于正常连接的状态。这就是心跳包，还有心跳说明连接正常，没有心跳说明连接端开。

实现效果是客户端连接后与服务端通过心跳包检测连接状态。当客户端超过一定时间收不到服务端的心跳包，客户端认为与服务端连接断开，关闭连接，并不停的尝试重连。

修改后台的`onMessage()`方法，当收到客户端的心跳包时，响应心跳包：

```java
@OnMessage
public void onMessage(String message, Session session) {
    log.info("【WebSocket】收到来自客户端：{} 的消息，消息内容：{}", session.getId(), message);

    // 如果客户端发送心跳包，返回心跳包
    if("ping".equals(message)) {
        sendMessage("pong");
    } else {
        sendMessage("收到消息：" + message);
    }
}
```

前台代码如下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>WebSocket Heart</title>
</head>
<body>

<button onclick="closeWebSocket()">主动断开连接</button>

<script>
    var ws = null, wsUrl = "ws://localhost:8080/ws1";
    var lockReconnect = false;  //避免ws重复连接

    createWebSocket(wsUrl);

    // 建立连接
    function createWebSocket(url) {
        if ('WebSocket' in window) {
            ws = new WebSocket(url);
        } else if ('MozWebSocket' in window) {
            ws = new MozWebSocket(url);
        } else {
            alert('您的浏览器不支持WebSocket，请更换浏览器');
        }
        initEventHandle();
    }

    // 初始化相关回调函数
    function initEventHandle() {
        //连接成功建立的回调方法
        ws.onopen = function(){
            console.log("客户端连接建立");
            //心跳检测重置
            heartCheck.reset().start();
        };

        //接收到消息的回调方法
        ws.onmessage = function(event){
            console.log("客户端收到消息啦:" +event.data);
            //拿到任何消息都说明当前连接是正常的，重置心跳
            heartCheck.reset().start();
        };

        //连接关闭的回调方法
        ws.onclose = function(){
            console.log("客户端连接关闭");
            // 重连WebSocket
            reconnect(wsUrl);
        };

        //连接发生错误的回调方法
        ws.onerror = function(){
            console.log("客户端连接错误");
            // 重连WebSocket
            reconnect(wsUrl);
        };
    }

    //监听窗口关闭事件，当窗口关闭时，主动去关闭webSocket连接，防止连接还没断开就关闭窗口，server端会抛异常。
    window.onbeforeunload = function(){
        ws.close();
    };

    // 重连WebSocket
    function reconnect(url) {
        if (lockReconnect)
            return;
        lockReconnect = true;

        //没连接上会一直重连，设置延迟避免请求过多
        setTimeout(function () {
            createWebSocket(url);
            lockReconnect = false;
        }, 2000);
    }


    //心跳检测
    var heartCheck = {
        timeout: 10000,        // 10s发一次心跳
        timeoutObj: null,
        serverTimeoutObj: null,
        reset: function () { //心跳包重置
            clearTimeout(this.timeoutObj);
            clearTimeout(this.serverTimeoutObj);
            return this;
        },
        start: function () {
            var self = this;
            this.timeoutObj = setTimeout(function () {
                // 向后台发送心跳
                ws.send("ping");
                self.serverTimeoutObj = setTimeout(function () { //如果超过一定时间还没重置，说明后端主动断开了
                    // 执行ws.close()会回调onclose，然后执行其中的reconnet。如果直接执行reconnect 会触发onclose导致重连两次
                    ws.close();
                }, self.timeout)
            }, this.timeout)
        }
    };

    //关闭连接
    function closeWebSocket(){
        ws.close();
    }
</script>
</body>
</html>
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010191652774.png)

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201810/20181010191709671.png)
